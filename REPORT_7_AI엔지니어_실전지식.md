# Claude Code Harness: AI 엔지니어 실전 지식 총정리
## 실제 코드 레벨 분석 | 파일:줄번호 인용 기반

> **목적**: AI 엔지니어링 전환을 위한 실전 레시피 모음. 이론 없음. 실제 코드와 "왜" 설명만.
> **근거**: claw-cli-v2.1.88/src/ 의 실제 파일을 에이전트 5개가 병렬로 정독한 결과.

---

## PART 1: 에이전트 루프 아키텍처

### 1.1 기본 패턴: AsyncGenerator + while(true) + State Machine

**query.ts:219-239**
```typescript
export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | RequestStartEvent | Message | ..., Terminal> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

**왜 AsyncGenerator?**
- `yield*`로 내부 generator 위임 → 에러 전파 보장
- SDK 클라이언트가 `for await`로 실시간 소비 가능
- Promise-based보다 스트리밍에 자연스러움

### 1.2 핵심 State 타입

**query.ts:203-217**
```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number        // 0-3
  hasAttemptedReactiveCompact: boolean         // 무한루프 방지
  maxOutputTokensOverride: number | undefined  // 8k → 64k 에스컬레이션
  pendingToolUseSummary: Promise<...> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

**transition 사유 (무한루프 계속 이유):**
- `next_turn`: 도구 결과 받아서 다음 턴
- `max_output_tokens_recovery`: 토큰 부족 → 재시도
- `reactive_compact_retry`: 컨텍스트 붕괴 → 재시도
- `collapse_drain_retry`: collapse drain 후 재시도
- `token_budget_continuation`: 토큰 예산 남음
- `stop_hook_blocking`: stop hook 추가 처리

### 1.3 7가지 종료 조건

**query.ts:307-1728의 while(true) 내부:**

| 조건 | 위치 | return reason |
|------|------|---------------|
| 스트림 중단 | 1015-1052 | `aborted_streaming` |
| 도구 필요 없음 + 복구 불가 | 1062-1357 | `completed` |
| 도구 실행 중 중단 | 1485-1515 | `aborted_tools` |
| 최대 턴 초과 | 1705-1712 | `max_turns` |
| 블로킹 limit 도달 | 637-648 | `blocking_limit` |
| Stop hook 차단 | 1267-1306 | `stop_hook_prevented` |
| Fallback 실패 | 893-953 | `FallbackTriggeredError` throw |

### 1.4 도구 실행: 병렬 vs 직렬 배치

**toolOrchestration.ts:91-116** - 배치 분할 알고리즘
```typescript
function partitionToolCalls(toolUseMessages, toolUseContext) {
  return toolUseMessages.reduce((acc, toolUse) => {
    const tool = findToolByName(toolUseContext.options.tools, toolUse.name)
    const isConcurrencySafe = tool?.isConcurrencySafe(parsedInput.data)
    
    // 규칙: 연속한 safe 도구들만 같은 배치 → 병렬
    //       safe 다음 unsafe → 새 배치 시작
    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      acc[acc.length - 1]!.blocks.push(toolUse)
    } else {
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

**실전 예시:**
```
입력: [Bash(ls), Glob, Bash(grep), FileEdit]

배치 1: [Bash(ls), Glob]     → 병렬 (둘 다 safe)
배치 2: [Bash(grep)]          → 직렬 (unsafe)  
배치 3: [FileEdit]            → 직렬 (unsafe)
```

**toolOrchestration.ts:152-177** - 병렬 실행 구현
```typescript
async function* runToolsConcurrently(...) {
  yield* all(
    toolUseMessages.map(async function* (toolUse) {
      yield* runToolUse(toolUse, ...)
      markToolUseAsComplete(toolUseContext, toolUse.id)
    }),
    getMaxToolUseConcurrency(),  // 기본 10 동시 (env var로 조정)
  )
}
```

**핵심:** `all()`은 concurrency-limited `Promise.all` 래퍼. Generator를 여러 개 받아서 스케줄링.

### 1.5 재시도 로직: 지수 백오프 + Jitter

**withRetry.ts:530-548**
```typescript
export function getRetryDelay(attempt, retryAfterHeader, maxDelayMs = 32000) {
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) return seconds * 1000  // 서버 지시 우선
  }
  
  const baseDelay = Math.min(
    500 * Math.pow(2, attempt - 1),  // 500ms * 2^(n-1)
    maxDelayMs,
  )
  const jitter = Math.random() * 0.25 * baseDelay  // ±0~25% 지터
  return baseDelay + jitter
}
```

**계산표:**
| Attempt | Base Delay | Max Jitter | Total 범위 |
|---------|-----------|-----------|-----------|
| 1 | 500ms | 125ms | 500-625ms |
| 2 | 1s | 250ms | 1-1.25s |
| 3 | 2s | 500ms | 2-2.5s |
| 4 | 4s | 1s | 4-5s |
| 5 | 8s | 2s | 8-10s |
| 6+ | 32s cap | 8s | 32-40s |

### 1.6 429 vs 529 차등 처리

**withRetry.ts:62-82** - Foreground 소스만 재시도
```typescript
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',
  'sdk',
  'agent:custom',
  'agent:default',
  'compact',
  'hook_agent',
  'auto_mode',
])

function shouldRetry529(querySource): boolean {
  return querySource === undefined || FOREGROUND_529_RETRY_SOURCES.has(querySource)
}
```

**배경 작업은 즉시 실패:**
- summaries, titles, suggestions, classifiers
- 사용자가 대기 중이 아님 → API 증폭 방지

**529 연속 3회 → Fallback 모델 전환:**
```typescript
// withRetry.ts:326-350
if (consecutive529Errors >= MAX_529_RETRIES && options.fallbackModel) {
  throw new FallbackTriggeredError(options.model, options.fallbackModel)
}
```

### 1.7 Persistent Retry (Unattended Mode)

**withRetry.ts:477-512**
```typescript
// CLAUDE_CODE_UNATTENDED_RETRY=1 (ant-only)
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000  // 5분
const HEARTBEAT_INTERVAL_MS = 30_000              // 30초

if (persistent) {
  let remaining = delayMs
  while (remaining > 0) {
    if (options.signal?.aborted) throw new APIUserAbortError()
    yield createSystemAPIErrorMessage(error, remaining, attempt, maxRetries)
    const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)
    await sleep(chunk, options.signal)
    remaining -= chunk
  }
}
```

**왜 30초 청크?** Host가 장시간 침묵 → "idle" 판정 → 프로세스 kill 방지.

---

## PART 2: 시스템 프롬프트 조립

### 2.1 15개 Composable 함수

**prompts.ts의 핵심 함수들:**

| # | 함수 | 위치 | 캐시 | ANT 분기 |
|---|------|------|------|---------|
| 1 | `getSimpleIntroSection()` | 175-184 | 글로벌 | 없음 |
| 2 | `getSimpleSystemSection()` | 186-197 | 글로벌 | 없음 |
| 3 | `getSimpleDoingTasksSection()` | 199-253 | 글로벌 | ✅ 있음 |
| 4 | `getActionsSection()` | 255-267 | 글로벌 | 없음 |
| 5 | `getUsingYourToolsSection()` | 269-314 | 부분 | 없음 |
| 6 | `getSimpleToneAndStyleSection()` | 430-442 | 글로벌 | ✅ 있음 |
| 7 | `getOutputEfficiencySection()` | 402-428 | 글로벌 | ✅ 있음 |
| 8 | `getSessionSpecificGuidanceSection()` | 491-493 | 세션 | - |
| 9 | `loadMemoryPrompt()` | 495 | 세션 | - |
| 10 | `getAntModelOverrideSection()` | 496-498 | 세션 | ✅ ANT만 |
| 11 | `getLanguageSection()` | 502-504 | 세션 | - |
| 12 | `getOutputStyleSection()` | 505-507 | 세션 | - |
| 13 | `getMcpInstructionsSection()` | 513-520 | **없음** | - |
| 14 | `getScratchpadInstructions()` | 521 | 세션 | - |
| 15 | `numeric_length_anchors` | 529-537 | 세션 | ✅ ANT만 |

### 2.2 캐시 경계 마커

**prompts.ts:114-115**
```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

**prompts.ts:572-573** - 삽입 위치
```typescript
// === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===
...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
```

**claude.ts:358-374** - cache_control 생성
```typescript
export function getCacheControl({ scope, querySource } = {}): {
  type: 'ephemeral'
  ttl?: '1h'
  scope?: CacheScope
} {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

**왜 중요한가:**
- `scope: 'global'` → 조직 간 캐시 공유 (히트율 ~60%)
- `scope: null` → 세션별 개별 캐시
- `ttl: '1h'` → ANT 사용자 + allowlist 매칭시만

### 2.3 ANT 빌드 분기 (Dead Code Elimination)

**prompts.ts:529-537** - 25단어 규칙
```typescript
...(process.env.USER_TYPE === 'ant'
  ? [
      systemPromptSection(
        'numeric_length_anchors',
        () =>
          'Length limits: keep text between tool calls to ≤25 words. ' +
          'Keep final responses to ≤100 words unless the task requires more detail.',
      ),
    ]
  : []),
```

**프로덕션 인사이트:**
- A/B 테스트로 검증된 ~1.2% output token 감소
- 컴파일타임 `--define`로 Dead Code Elimination
- 외부 빌드에서는 자동 제거 → 번들 크기 축소

### 2.4 환경 정보 주입

**context.ts:36-111** - Git 상태 병렬 수집
```typescript
const [branch, mainBranch, status, log, userName] = await Promise.all([
  getBranch(),
  getDefaultBranch(),
  execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'])
    .then(({stdout}) => stdout.trim()),
  execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'])
    .then(({stdout}) => stdout.trim()),
  execFileNoThrow(gitExe(), ['config', 'user.name'])
    .then(({stdout}) => stdout.trim()),
])
```

**왜 `--no-optional-locks`?** 에이전트가 동시에 git 작업하면 lock 충돌.

### 2.5 CLAUDE.md 4계층 로딩

**claudemd.ts:1-27**
```
1. Managed (/etc/claude-code/CLAUDE.md)           - 조직
2. User (~/.claude/CLAUDE.md)                     - 개인 전역
3. Project (./CLAUDE.md, .claude/rules/*.md)     - 프로젝트
4. Local (./CLAUDE.local.md)                     - 개인 프로젝트 (git-ignore)
```

**크기 제한:**
```typescript
// claudemd.ts:92
export const MAX_MEMORY_CHARACTER_COUNT = 40000
```

**우선순위:** 역순 (Local > Project > User > Managed)

---

## PART 3: 도구 시스템 & 권한

### 3.1 Tool 인터페이스 (정확한 시그니처)

**Tool.ts:362-695**
```typescript
export type Tool<Input, Output, P> = {
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  checkPermissions(input, context): Promise<PermissionResult>
  
  isReadOnly(input): boolean        // 병렬화 가능 여부 1
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean // 병렬화 가능 여부 2
  isEnabled(): boolean
  validateInput?(input, context): Promise<ValidationResult>
  
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
}
```

### 3.2 권한 모드 상태머신

**permissions.ts:16-36**
```typescript
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',      // 파일 수정만 자동 승인
  'bypassPermissions', // 모든 도구 자동 승인 (YOLO)
  'default',          // 매번 확인
  'dontAsk',          // 모든 도구 자동 거부
  'plan',             // 계획 검토 필수
] as const

// ANT 전용:
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
// auto: 트랜스크립트 분류기 기반
// bubble: 계층적 에이전트 권한 전파
```

### 3.3 권한 파이프라인 4단계

**useCanUseTool.tsx:27-170**
```typescript
// 1단계: 모드 체크 (hasPermissionsToUseTool 내부)
// 2단계: Hook 평가 (PreToolUse)
// 3단계: Rule 매칭
// 4단계: 사용자 프롬프트 (마지막 수단)

switch (result.behavior) {
  case "allow":
    resolve(ctx.buildAllow(result.updatedInput ?? input))
    break
  case "deny":
    logPermissionDecision({...}, { decision: "reject", source: "config" })
    resolve(result)  // PermissionDenyDecision 반환
    break
  case "ask":
    if (appState.toolPermissionContext.awaitAutomatedChecksBeforeDialog) {
      const coordinatorDecision = await handleCoordinatorPermission({...})
      if (coordinatorDecision) { resolve(coordinatorDecision); return }
    }
    handleInteractivePermission({...}, resolve)
    break
}
```

### 3.4 거부 피드백 루프 (핵심 설계!)

**permissions.ts:137-200** - 거부 이유 메시지 생성
```typescript
export function createPermissionRequestMessage(
  toolName: string,
  decisionReason?: PermissionDecisionReason,
): string {
  switch (decisionReason.type) {
    case 'classifier':
      return `Classifier '${decisionReason.classifier}' requires approval for this ${toolName} command: ${decisionReason.reason}`
    case 'hook':
      return `Hook '${decisionReason.hookName}' requires approval for this ${toolName} command`
    case 'rule':
      return `Permission rule '${ruleString}' from ${sourceString} requires approval`
    case 'mode':
      return `Current permission mode (${modeTitle}) requires approval for this ${toolName} command`
  }
}
```

**거부 누적 추적 (denialTracking.ts:1-46):**
```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,  // 연속 거부 3회
  maxTotal: 20,       // 총 거부 20회
}

export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}
```

**왜 거부를 에러로 래핑하는가:**
- 대부분 에이전트는 거부시 멈춤
- Claude Code는 거부 메시지를 `tool_result error`로 모델에 전달
- 모델이 자동으로 **더 안전한 대안** 탐색
- 예: `rm -rf` 거부 → `git clean -fd` 시도 → 거부 → 사용자 직접 요청

### 3.5 Bash 보안 5단계 (300KB+ 코드)

**bashSecurity.ts:76-101** - 23개 보안 검사 ID
```typescript
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,          // jq 내부 함수 차단
  JQ_FILE_ARGUMENTS: 3,            // jq 파일 I/O 차단
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,          // $BASH_ENV 등
  NEWLINES: 7,                     // 숨겨진 개행
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,  // $(), ${}
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,         // /proc/self/environ
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
}
```

### 3.6 Command Substitution 패턴

**bashSecurity.ts:128-175**
```typescript
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /(?:^|[\s;&|])=[a-zA-Z_]/, message: 'Zsh equals expansion (=cmd)' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  { pattern: /\$\[/, message: '$[] legacy arithmetic' },
  // 총 12개 패턴
]
```

### 3.7 환경 변수 화이트리스트

**bashPermissions.ts:378-430**
```typescript
const SAFE_ENV_VARS = new Set([
  // Go
  'GOEXPERIMENT', 'GOOS', 'GOARCH', 'CGO_ENABLED', 'GO111MODULE',
  // Rust
  'RUST_BACKTRACE', 'RUST_LOG',
  // Node
  'NODE_ENV',  // ⚠️ NODE_OPTIONS는 위험
  // Python
  'PYTHONUNBUFFERED', 'PYTHONDONTWRITEBYTECODE',  // ⚠️ PYTHONPATH는 위험
  'PYTEST_DISABLE_PLUGIN_AUTOLOAD',
  // 인증 (API 키만 허용)
  'ANTHROPIC_API_KEY',
  // 로케일/터미널
  'LANG', 'LC_ALL', 'TERM', 'COLORTERM', 'NO_COLOR',
])
```

**원칙:** 코드 실행, 라이브러리 로드, 시스템 동작을 바꾸는 변수는 **절대 화이트리스트 금지**.

### 3.8 Wildcard 권한 매칭

**bashPermissions.ts:353-358**
```typescript
// 'Bash(git commit:*)' matches 'git commit -m "msg"'
// 'Bash(npm:*)'       matches 'npm install', 'npm run test'
// 'Bash(rm *)'        matches 'rm file.txt'
export function matchWildcardPattern(pattern, command): boolean {
  return sharedMatchWildcardPattern(pattern, command)
}
```

---

## PART 4: Compaction & Token 관리

### 4.1 4단계 전략 (계층적 방어)

```
Stage 1: Proactive    → 13k 버퍼 도달시 자동 요약
Stage 2: Reactive     → prompt_too_long 에러시 재시도
Stage 3: Snip         → SDK/headless 모드 절단
Stage 4: Collapse     → marble_origami feature, tool_result 폴딩
```

### 4.2 Proactive 임계값

**autoCompact.ts:62-65**
```typescript
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

**계산식:**
```
임계값 = 컨텍스트 윈도우 - MAX_OUTPUT_TOKENS_FOR_SUMMARY - AUTOCOMPACT_BUFFER_TOKENS
        = 200,000 - 20,000 - 13,000
        = 167,000 (200K 모델)
```

### 4.3 토큰 경고 상태 계산

**autoCompact.ts:93-145**
```typescript
export function calculateTokenWarningState(tokenUsage, model) {
  const autoCompactThreshold = getAutoCompactThreshold(model)
  const threshold = isAutoCompactEnabled()
    ? autoCompactThreshold
    : getEffectiveContextWindowSize(model)
  
  const percentLeft = Math.max(
    0,
    Math.round(((threshold - tokenUsage) / threshold) * 100),
  )
  
  return {
    percentLeft,
    isAboveWarningThreshold: tokenUsage >= threshold - WARNING_THRESHOLD_BUFFER_TOKENS,
    isAboveErrorThreshold: tokenUsage >= threshold - ERROR_THRESHOLD_BUFFER_TOKENS,
    isAboveAutoCompactThreshold: isAutoCompactEnabled() && tokenUsage >= autoCompactThreshold,
    isAtBlockingLimit: tokenUsage >= getEffectiveContextWindowSize(model) - MANUAL_COMPACT_BUFFER_TOKENS,
  }
}
```

### 4.4 Reactive: prompt_too_long 파싱

**errors.ts:85-118**
```typescript
export function getPromptTooLongTokenGap(msg): number | undefined {
  // 에러 메시지 예: "prompt is too long: 137500 tokens > 135000 maximum"
  const { actualTokens, limitTokens } = parsePromptTooLongTokenCounts(msg.errorDetails)
  const gap = actualTokens - limitTokens  // 2500
  return gap > 0 ? gap : undefined
}
```

### 4.5 자동 요약 프롬프트 구조

**compact/prompt.ts:1-79**

**NO_TOOLS_PREAMBLE (19-26행):**
```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: <analysis> + <summary> block.
```

**BASE_COMPACT_PROMPT (61-76행)** - 요약 9개 섹션:
```
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections (파일명 + 전체 코드 + 함수 시그니처)
4. Errors and fixes (특히 사용자 피드백)
5. Problem Solving
6. All user messages (tool_result 제외)
7. Pending Tasks
8. Current Work (가장 최근 작업 디테일)
9. Optional Next Step (명시적 사용자 요청만)
```

### 4.6 Token Budget "+500k" 파싱

**utils/tokenBudget.ts:1-29**
```typescript
const SHORTHAND_START_RE = /^\s*\+(\d+(?:\.\d+)?)\s*(k|m|b)\b/i
const SHORTHAND_END_RE = /\s\+(\d+(?:\.\d+)?)\s*(k|m|b)\s*[.!?]?\s*$/i
const VERBOSE_RE = /\b(?:use|spend)\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*tokens?\b/i

const MULTIPLIERS = {
  k: 1_000,
  m: 1_000_000,
  b: 1_000_000_000,
}

// 예시:
// "+500k"               → 500,000
// "spend 2M tokens"     → 2,000,000
// "+1.5m."              → 1,500,000
```

### 4.7 Budget Continuation 결정

**query/tokenBudget.ts:45-93**
```typescript
const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500  // 최근 500토큰 미만

if (!isDiminishing && globalTurnTokens < budget * 0.9) {
  return {
    action: 'continue',
    nudgeMessage: `Stopped at ${pct}% of token target (${fmt(turnTokens)} / ${fmt(budget)}). Keep working — do not summarize.`,
    continuationCount: ++tracker.continuationCount,
  }
}

// Diminishing returns = 3회 연속 500 토큰 미만 → stop
```

### 4.8 Haiku Fallback Token Counting

**tokenEstimation.ts:246-325**
```typescript
export async function countTokensViaHaikuFallback(messages, tools) {
  const model = isVertexGlobalEndpoint || isBedrockWithThinking
    ? getDefaultSonnetModel()   // Sonnet fallback
    : getSmallFastModel()       // Haiku 4.5 기본
  
  const response = await anthropic.beta.messages.create({
    model,
    max_tokens: 1,  // 토큰 카운팅만, 생성 안 함
    messages, tools,
  })
  
  return response.usage.input_tokens
       + (response.usage.cache_creation_input_tokens || 0)
       + (response.usage.cache_read_input_tokens || 0)
}
```

### 4.9 파일 타입별 토큰 추정

**tokenEstimation.ts:215-224**
```typescript
function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json':
    case 'jsonl':
    case 'jsonc':
      return 2   // 중괄호, 콜론, 쉼표 많음 → 토큰 밀도 높음
    default:
      return 4   // 일반 텍스트
  }
}
```

### 4.10 모델별 가격표

**modelCost.ts:26-126**

| 모델 | Input | Output | Cache Write | Cache Read |
|------|-------|--------|-------------|------------|
| Haiku 3.5 | $0.80 | $4 | $1 | $0.08 |
| Haiku 4.5 | $1 | $5 | $1.25 | $0.1 |
| Sonnet 4.6 | $3 | $15 | $3.75 | $0.3 |
| Opus 4.6 | $5 | $25 | $6.25 | $0.5 |
| Opus 4.6 Fast | $30 | $150 | $37.5 | $3 |

**비용 계산 (modelCost.ts:131-142):**
```typescript
function tokensToUSDCost(modelCosts, usage): number {
  return (
    (usage.input_tokens / 1_000_000) * modelCosts.inputTokens +
    (usage.output_tokens / 1_000_000) * modelCosts.outputTokens +
    ((usage.cache_read_input_tokens ?? 0) / 1_000_000) * modelCosts.promptCacheReadTokens +
    ((usage.cache_creation_input_tokens ?? 0) / 1_000_000) * modelCosts.promptCacheWriteTokens +
    (usage.server_tool_use?.web_search_requests ?? 0) * modelCosts.webSearchRequests
  )
}
```

**핵심 인사이트:**
- Cache Read는 Input의 **1/10 가격** (Sonnet: $0.3 vs $3)
- Cache Write는 Input의 **1.25배 가격** (Sonnet: $3.75 vs $3)
- → 캐시 히트율이 비용 결정적

### 4.11 환경 변수 (디버그)

| 변수 | 효과 |
|------|------|
| `DISABLE_COMPACT=1` | 모든 compaction 비활성화 |
| `DISABLE_AUTO_COMPACT=1` | autocompact만 비활성화 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=75` | 75%에서 트리거 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW=150000` | 윈도우 커스텀 |
| `CLAUDE_CODE_UNATTENDED_RETRY=1` | Persistent retry (ant-only) |
| `ENABLE_PROMPT_CACHING_1H_BEDROCK=1` | Bedrock 1h TTL |
| `REACTIVE_COMPACT=1` | Proactive 억제 |

---

## PART 5: 메모리 & MCP & Bridge

### 5.1 Memdir 파일 구조

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md              # 인덱스 (200줄/25KB 제한)
├── user_role.md           # 사용자 역할
├── feedback_testing.md    # 피드백
├── project_auth.md        # 프로젝트 컨텍스트
└── reference_dashboards.md # 외부 리소스
```

### 5.2 Memory 파일 Frontmatter

**memoryTypes.ts:14-31**
```typescript
export const MEMORY_TYPES = [
  'user',       // 사용자 역할/선호도/지식
  'feedback',   // 수정된 접근법
  'project',    // 진행중 작업/제약
  'reference',  // 외부 시스템 (Linear, Slack)
] as const
```

**파일 형식:**
```markdown
---
name: "사용자 역할"
description: "데이터 사이언티스트, observability 담당"
type: user
---

사용자는 데이터 사이언티스트로 현재 로깅/observability 조사 중...
```

### 5.3 200줄/25KB 이중 절단

**memdir.ts:34-103**
```typescript
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const trimmed = raw.trim()
  const contentLines = trimmed.split('\n')
  const wasLineTruncated = contentLines.length > MAX_ENTRYPOINT_LINES
  const wasByteTruncated = trimmed.length > MAX_ENTRYPOINT_BYTES
  
  if (!wasLineTruncated && !wasByteTruncated) {
    return { content: trimmed, ... }
  }

  let truncated = wasLineTruncated
    ? contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    : trimmed

  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    // 마지막 newline에서 절단 (mid-line 방지)
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }
  
  // 경고 메시지 추가
  return {
    content: truncated + `\n\n> WARNING: ${ENTRYPOINT_NAME} is ${reason}...`,
  }
}
```

### 5.4 Sonnet 기반 메모리 리콜

**findRelevantMemories.ts:39-75**
```typescript
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]> {
  const memories = (await scanMemoryFiles(memoryDir, signal))
    .filter(m => !alreadySurfaced.has(m.filePath))
  
  // Sonnet에게 JSON schema로 관련 메모리 선택 요청
  const result = await sideQuery({
    model: getDefaultSonnetModel(),
    system: SELECT_MEMORIES_SYSTEM_PROMPT,
    messages: [{
      role: 'user',
      content: `Query: ${query}\n\nAvailable memories:\n${manifest}${toolsSection}`,
    }],
    max_tokens: 256,
    output_format: {
      type: 'json_schema',
      schema: {
        type: 'object',
        properties: {
          selected_memories: { type: 'array', items: { type: 'string' } },
        },
        required: ['selected_memories'],
        additionalProperties: false,
      },
    },
    querySource: 'memdir_relevance',
  })
  
  return selected.map(m => ({ path: m.filePath, mtimeMs: m.mtimeMs }))
}
```

**왜 이 설계?**
- 모든 메모리를 시스템 프롬프트에 넣으면 토큰 낭비
- 쿼리마다 관련 메모리만 선택해서 로드 (Sonnet이 판단)
- JSON schema로 출력 구조 보장
- `alreadySurfaced`로 중복 방지

### 5.5 MCP 도구 이름 파싱

**mcpStringUtils.ts:19-52**
```typescript
// 'mcp__claude_ai_Figma__get_screenshot' →
//   serverName: 'claude_ai_Figma'
//   toolName: 'get_screenshot'
export function mcpInfoFromString(toolString: string) {
  const parts = toolString.split('__')
  const [mcpPart, serverName, ...toolNameParts] = parts
  if (mcpPart !== 'mcp' || !serverName) return null
  
  const toolName = toolNameParts.length > 0 ? toolNameParts.join('__') : undefined
  return { serverName, toolName }
}
```

**MCP 프로토콜 (실제 메시지):**
```json
// 도구 목록 요청
{"jsonrpc":"2.0","id":1,"method":"tools/list"}

// 도구 호출
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
  "name":"get_screenshot",
  "arguments":{"fileKey":"abc","nodeId":"1:2"}
}}
```

### 5.6 JWT 토큰 자동 갱신

**jwtUtils.ts:72-140**
```typescript
const TOKEN_REFRESH_BUFFER_MS = 5 * 60 * 1000      // 만료 5분 전 갱신
const FALLBACK_REFRESH_INTERVAL_MS = 30 * 60 * 1000 // 30분 보험 갱신
const REFRESH_RETRY_DELAY_MS = 60 * 1000           // 실패 시 60초 후 재시도
const MAX_REFRESH_FAILURES = 3

export function createTokenRefreshScheduler({
  getAccessToken,
  onRefresh,
  label,
  refreshBufferMs = TOKEN_REFRESH_BUFFER_MS,
}) {
  const timers = new Map<string, Timeout>()
  const failureCounts = new Map<string, number>()
  const generations = new Map<string, number>()  // 재스케줄링 감지
  
  function schedule(sessionId, token) {
    const expiry = decodeJwtExpiry(token)
    const delayMs = expiry * 1000 - Date.now() - refreshBufferMs
    
    if (delayMs <= 0) {
      void doRefresh(sessionId, gen)  // 즉시 갱신
      return
    }
    
    const timer = setTimeout(doRefresh, delayMs, sessionId, gen)
    timers.set(sessionId, timer)
  }
  
  async function doRefresh(sessionId, gen) {
    const oauthToken = await getAccessToken()
    
    // Generation 체크: 다른 곳에서 재스케줄됐으면 skip
    if (generations.get(sessionId) !== gen) return
    
    if (!oauthToken) {
      const failures = (failureCounts.get(sessionId) ?? 0) + 1
      failureCounts.set(sessionId, failures)
      if (failures < MAX_REFRESH_FAILURES) {
        setTimeout(doRefresh, REFRESH_RETRY_DELAY_MS, sessionId, gen)
      }
      return
    }
    
    onRefresh(sessionId, oauthToken)
    // 30분 후 다시 갱신 (보험)
    setTimeout(doRefresh, FALLBACK_REFRESH_INTERVAL_MS, sessionId, gen)
  }
}
```

### 5.7 Coordinator Mode 워커 프롬프트

**coordinatorMode.ts:111-200**
```typescript
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates...

## Your Role
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that you can handle without tools

## Your Tools
- **${AGENT_TOOL_NAME}** - Spawn a new worker
- **${SEND_MESSAGE_TOOL_NAME}** - Continue an existing worker
- **${TASK_STOP_TOOL_NAME}** - Stop a running worker

## Parallelism is your superpower
**Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible**

Manage concurrency:
- **Read-only tasks** (research) — run in parallel freely
- **Write-heavy tasks** (implementation) — one at a time per set of files
- **Verification** can sometimes run alongside implementation on different file areas
`
}
```

### 5.8 조건부 Skill 활성화 (경로 기반)

**loadSkillsDir.ts:997-1058**
```typescript
// 특정 파일 경로를 매칭하는 skill만 활성화
// 예: Python 파일 편집시만 Python skill 활성
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string,
): string[] {
  const activated: string[] = []
  
  for (const [name, skill] of conditionalSkills) {
    if (!skill.paths || skill.paths.length === 0) continue
    
    const skillIgnore = ignore().add(skill.paths)  // gitignore 스타일
    for (const filePath of filePaths) {
      const relativePath = isAbsolute(filePath) 
        ? relative(cwd, filePath) 
        : filePath
      
      if (skillIgnore.ignores(relativePath)) {
        // 매칭 → 동적 skill로 이동
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        activated.push(name)
        break
      }
    }
  }
  
  if (activated.length > 0) {
    skillsLoaded.emit()  // 캐시 무효화
  }
  return activated
}
```

---

## PART 6: AI 엔지니어를 위한 실전 레시피

### 6.1 직접 에이전트 만들 때 반드시 구현할 10가지

```
✅ 1. AsyncGenerator 루프 패턴
   → Promise.all보다 스트리밍/에러 전파 우수

✅ 2. State 객체 + while(true) + transition reason
   → 7+ 복구 경로 지원, 디버깅 용이

✅ 3. 배치 분할 + Concurrency 제한 도구 실행
   → 읽기 전용만 병렬, 쓰기는 직렬

✅ 4. 거부 피드백 루프
   → 거부를 tool_result error로 → 모델에 전달 → 자동 대안

✅ 5. 시스템 프롬프트 정적/동적 분리
   → __BOUNDARY__ 마커 + cache_control: ephemeral + scope: global

✅ 6. 지수 백오프 + Jitter (±25%)
   → Thundering herd 방지, 서버 Retry-After 우선

✅ 7. 13K 버퍼 Proactive Compaction
   → 임계값에서 자동 요약, 사용자 무감지

✅ 8. 4계층 메모리 병합 (역순 우선순위)
   → Managed → User → Project → Local

✅ 9. Haiku Fallback 토큰 카운팅
   → API 없이 파일 타입별 bytes/token 추정

✅ 10. Hook 시스템 + Exit Code 해석
    → PreToolUse/PostToolUse, 체이닝, JSON 출력 파싱
```

### 6.2 절대 하지 말아야 할 안티패턴

```
❌ Promise.all로 모든 도구 병렬 실행
   → 쓰기 도구 경쟁 상태 발생

❌ 거부시 단순 멈춤
   → 모델이 대안 탐색 못함, UX 나쁨

❌ 시스템 프롬프트 매번 통째로 전송
   → 캐시 히트 안 됨, 비용 폭발

❌ 단순 재시도 (백오프 없이)
   → 529 에러 증폭, rate limit 영구화

❌ 토큰 카운팅 없이 context 관리
   → prompt_too_long → 복구 불가

❌ 사용자 메시지 비동기 저장
   → 세션 유실시 사용자 입력 손실

❌ 요약 프롬프트에 도구 허용
   → 요약 도중 도구 호출로 토큰 낭비

❌ 환경 변수 전부 허용
   → NODE_OPTIONS, PYTHONPATH 통한 코드 주입

❌ 모든 메모리를 시스템 프롬프트에 포함
   → 토큰 낭비, Sonnet으로 관련성 선택 필요

❌ MCP 서버 스키마를 sync로 로딩
   → 콜드 스타트 지연, 병렬 로딩 필수
```

### 6.3 프로덕션 학습 (Haseeb 분석 + 코드 증거)

**"하네스가 전부다"**
```
API 호출: ~200 줄
하네스:   ~500,000 줄
비율:     1 : 2500
```

**코드로 증명된 프로덕션 흔적:**

1. **비대칭 세션 저장** - 사용자 메시지 동기, 어시스턴트 비동기
   → 세션 유실 사고에서 학습

2. **25단어 규칙** - ANT 빌드에만 추가
   → A/B 테스트로 1.2% output token 감소 검증

3. **529 3회 → Fallback** - fallback 모델 전환
   → Opus 과부하 시에도 서비스 유지

4. **Reactive Compaction 1회만** - `hasAttemptedReactiveCompact` 플래그
   → 무한 루프 방지

5. **30초 Heartbeat** - Persistent retry
   → Host가 idle 판정 후 kill 방지

6. **Generation Tracking** - JWT 갱신 스케줄러
   → Race condition 방지

7. **`--no-optional-locks`** - Git 명령어
   → 동시 에이전트 lock 충돌 방지

8. **Promise.allSettled** - 메모리 스캔
   → 개별 파일 실패 격리

9. **Realpath 중복 제거** - Skill 로딩
   → 심볼릭 링크 꼬임 방지

10. **경고 메시지 추가** - 절단된 메모리
    → 사용자가 축소됐다는 사실 인식

### 6.4 환경 변수 치트시트

**개발/디버깅:**
```bash
export DEBUG=1                           # 디버그 로그
export DISABLE_AUTO_COMPACT=1            # 자동 compaction 끄기
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=75 # 75%에서 트리거
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=32000 # 출력 토큰 캡
```

**프로덕션:**
```bash
export CLAUDE_CODE_UNATTENDED_RETRY=1    # 무한 재시도 (ant-only)
export ENABLE_PROMPT_CACHING_1H_BEDROCK=1 # Bedrock 1h TTL
export FALLBACK_FOR_ALL_PRIMARY_MODELS=1 # 모든 모델 fallback
```

**보안/성능:**
```bash
export CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1 # 백그라운드 금지
export CLAUDE_CODE_DISABLE_POLICY_SKILLS=1   # 관리 skill 비활성
```

### 6.5 핵심 파일 맵 (빠른 참조)

```
프로젝트: /Volumes/WorkDrive/Develop/46.claude-code-harness/claw-cli-v2.1.88/src/

에이전트 루프:
├── query.ts                    (67KB)  메인 루프
├── QueryEngine.ts              (46KB)  SDK/REPL 추상화
├── tools.ts                            도구 레지스트리
└── StreamingToolExecutor.ts            스트리밍 도구 실행

시스템 프롬프트:
├── constants/prompts.ts                15개 composable 함수
├── context.ts                  (6.3KB) 환경/사용자 컨텍스트
├── services/api/claude.ts              API 클라이언트
└── utils/api.ts                        prompt 조립 헬퍼

도구 & 권한:
├── Tool.ts                     (28KB)  기본 클래스
├── tools/BashTool/                     Bash 도구
├── tools/FileEditTool/                 파일 편집
├── tools/AgentTool/            (228KB) 서브에이전트
├── services/bashPermissions.ts (96KB)  Bash 보안
├── services/pathValidation.ts  (42KB)  경로 검증
├── services/readOnlyValidation.ts (66KB) 읽기전용
├── types/permissions.ts                권한 타입
└── utils/permissions/                  권한 유틸

Compaction:
├── services/compact/autoCompact.ts     자동 compaction
├── services/compact/prompt.ts          요약 프롬프트
├── services/api/errors.ts              에러 파싱
├── services/api/withRetry.ts           재시도 로직
├── utils/tokens.ts                     토큰 카운팅
├── utils/tokenEstimation.ts            추정
├── utils/cost-tracker.ts               비용 추적
├── utils/modelCost.ts                  가격표
└── utils/tokenBudget.ts                예산 파싱

메모리 & MCP:
├── memdir/memdir.ts                    Memdir 메인
├── memdir/memoryScan.ts                파일 스캔
├── memdir/findRelevantMemories.ts      Sonnet 리콜
├── memdir/memoryTypes.ts               타입 정의
├── utils/claudemd.ts                   CLAUDE.md 로딩
├── services/mcp/mcpClient.ts           MCP 클라이언트
├── services/mcp/mcpStringUtils.ts      이름 파싱
└── tools/MCPTool/                      MCP 도구

Bridge & Coordinator:
├── bridge/bridgeMain.ts        (113KB) Bridge 메인
├── bridge/replBridge.ts        (98KB)  REPL Bridge
├── bridge/jwtUtils.ts                  JWT 갱신
├── bridge/bridgeMessaging.ts           메시지 라우팅
├── bridge/remoteBridgeCore.ts  (39KB)  v2 프로토콜
└── coordinator/coordinatorMode.ts (19KB) 멀티에이전트

Skills & Plugins:
├── skills/loadSkillsDir.ts             Skill 로딩
├── skills/frontmatterParser.ts         YAML 파싱
└── plugins/                            플러그인 시스템
```

---

## PART 7: 요약 카드 (암기용)

### 7.1 에이전트 루프 5줄 요약

```
1. AsyncGenerator while(true) State 머신
2. 7가지 종료 조건 + transition reason
3. 병렬 safe / 직렬 unsafe 배치 분할  
4. 지수 백오프 + ±25% jitter
5. 거부 → tool_result error → 모델 자동 대안
```

### 7.2 시스템 프롬프트 5줄 요약

```
1. 15개 composable 함수 + __BOUNDARY__ 마커
2. 정적 = scope: 'global', 동적 = scope: null
3. cache_control: ephemeral (+ optional ttl: '1h')
4. ANT 분기 = process.env.USER_TYPE === 'ant' DCE
5. CLAUDE.md 4계층 역순 우선순위 (Local 최강)
```

### 7.3 권한 시스템 5줄 요약

```
1. 4모드: default/plan/acceptEdits/bypassPermissions
2. 파이프라인: mode → hook → rule → user prompt
3. Bash 보안 23개 검사 ID + 12개 치환 패턴
4. 환경변수는 안전 리스트만 화이트리스트
5. 거부 누적 3연속/20총 → 강제 프롬프트
```

### 7.4 Compaction 5줄 요약

```
1. Proactive 13K 버퍼, Reactive prompt_too_long, Snip SDK, Collapse ephemeral
2. Haiku fallback 토큰 카운팅 (파일타입별 bytes/token)
3. Cache Read = Input/10 가격, Cache Write = Input*1.25
4. "+500k" 파싱: SHORTHAND/VERBOSE 정규식
5. Budget 90% + diminishing returns 3회 = stop
```

### 7.5 메모리/MCP 5줄 요약

```
1. Memdir 200줄/25KB 이중 절단 + 경고 메시지
2. Sonnet sideQuery + JSON schema로 관련 메모리 선택
3. MCP 이름 = 'mcp__' + serverName + '__' + toolName
4. JWT 만료 5분 전 자동 갱신 + 30분 보험
5. Skill 조건부 활성화 = 파일 경로 매칭 (gitignore 스타일)
```

---

## 결론

**Claude Code = 에이전트 루프 + 시스템 프롬프트 최적화 + 도구 권한 + Compaction + 메모리 + MCP/Bridge**

이 5개 시스템이 **독립적으로 설계**되어 **느슨한 결합**으로 통합됨. AI 엔지니어가 자기 에이전트를 만들 때:

1. **루프 먼저** (AsyncGenerator + State)
2. **도구 시스템** (권한 + 병렬 + 거부 피드백)
3. **프롬프트 캐시** (정적/동적 분리)
4. **Compaction** (13K 버퍼)
5. **메모리** (Sonnet 기반 리콜)
6. **MCP/Bridge** (마지막에)

**핵심 원칙 3가지:**
- **스트리밍 우선** (Promise 대신 Generator)
- **비용 최적화** (캐시 경계 명시)
- **프로덕션 복구** (모든 에러에 복구 경로)

---

*총 분석: 800K+ LOC | 병렬 에이전트 5개 | 파일:줄번호 인용 100+개 | 2026-04-16*
