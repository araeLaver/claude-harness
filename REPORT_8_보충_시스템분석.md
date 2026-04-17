# Claude Code Harness 보충 시스템 분석
## Hooks / Telemetry / React Ink / OAuth / Voice / Error Recovery / Worktree / Cron

> 이전 REPORT_7에서 커버하지 못한 11개 서브시스템의 실제 코드 분석

---

## 1. Hooks 시스템

### 1.1 이벤트 타입 (20+)

```typescript
type HookEvent =
  | 'PreToolUse'          // 도구 사용 전 필터링/승인
  | 'PostToolUse'         // 도구 결과 후처리
  | 'PostToolUseFailure'  // 도구 실행 실패
  | 'SessionStart'        // 세션 초기화
  | 'SessionEnd'          // 세션 종료
  | 'Setup'               // 초기 설정
  | 'PermissionRequest'   // 권한 요청
  | 'PermissionDenied'    // 권한 거부
  | 'UserPromptSubmit'    // 사용자 입력 처리
  | 'Notification'        // 알림
  | 'FileChanged'         // 파일 변경 감시
  | 'CwdChanged'          // 작업 디렉토리 변경
  | 'SubagentStart'       // 서브에이전트 시작
  | 'SubagentStop'        // 서브에이전트 종료
  | ...                   // 총 20+
```

### 1.2 settings.json 훅 설정 구조

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "if": "Write(*.ts)",
        "hooks": [
          {
            "type": "command",
            "command": "bash check-ts-syntax.sh",
            "shell": "bash",
            "timeout": 10,
            "async": false
          },
          {
            "type": "prompt",
            "prompt": "Warn if using deprecated APIs"
          },
          {
            "type": "http",
            "url": "https://hook-server/pre-tool-use"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "resume",
        "hooks": [{ "type": "command", "command": "source ~/.bashrc" }]
      }
    ]
  }
}
```

**3가지 훅 타입:**
- `command`: 쉘 명령 실행, exit code + JSON stdout 해석
- `prompt`: 모델에게 프롬프트 주입
- `http`: 외부 웹훅 POST 호출

### 1.3 JSON 응답 포맷

```typescript
type SyncHookJSONOutput = {
  decision?: 'approve' | 'block'
  reason?: string
  systemMessage?: string      // 모델에게 주입할 메시지
  updatedInput?: unknown      // 수정된 도구 입력
  outputToAppend?: string     // 결과에 추가할 텍스트
  suppressOutput?: boolean    // 원래 결과 숨김
}
```

### 1.4 실행 흐름

```
도구 호출 → PreToolUse 훅 매칭 → 쉘 실행 (env 주입)
  → exit code 해석:
    0 = 승인 (JSON으로 decision/systemMessage 포함 가능)
    1 = 거부 (reason 포함)
    2+ = 비차단 에러
  → PostToolUse 훅 → 결과 후처리
```

**환경 변수 주입:**
```
CLAUDE_SESSION_ID, CLAUDE_TRANSCRIPT_PATH, CLAUDE_CWD,
CLAUDE_PERMISSION_MODE, CLAUDE_AGENT_ID, CLAUDE_AGENT_TYPE,
CLAUDE_TOOL_NAME, CLAUDE_TOOL_INPUT (JSON)
```

---

## 2. Telemetry (2개 파이프라인)

### 2.1 1P (OpenTelemetry)

```
엔드포인트: /api/event_logging/batch
배치 지연: 5000ms
최대 배치: 200 이벤트
최대 큐: 8192 이벤트
타임아웃: 10초
재시도: Quadratic backoff (base * attempts²)
디스크 캐시: ~/.claude/sessions/<id>/events_batch_<uuid>.json
```

**Quadratic Backoff (firstPartyEventLoggingExporter.ts:445):**
```typescript
const delay = Math.min(
  this.baseBackoffDelayMs * this.attempts * this.attempts,
  this.maxBackoffDelayMs,
)
```

### 2.2 Datadog

```
엔드포인트: https://http-intake.logs.us5.datadoghq.com/api/v2/logs
클라이언트 토큰: pubbbf48e6d78dae54bceaa4acf463299bf
플러시 간격: 15초
최대 배치: 100 이벤트
타임아웃: 5초
사용자 버킷: SHA-256(userId) % NUM_USER_BUCKETS
```

**이벤트 화이트리스트 (50+):**
```typescript
const DATADOG_ALLOWED_EVENTS = new Set([
  'tengu_api_error', 'tengu_api_success',
  'tengu_brief_mode_enabled', 'tengu_oauth_error',
  'tengu_session_file_read', 'tengu_streaming_idle_timeout',
  // ... 50+ 이벤트
])
```

### 2.3 옵트아웃

```typescript
export function isAnalyticsDisabled(): boolean {
  return (
    process.env.NODE_ENV === 'test' ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isTelemetryDisabled()
  )
}
```

**핵심:** 3P 프로바이더 사용자는 자동 비활성화. UI 옵트아웃 없음 (환경변수만).

### 2.4 킬스위치 6종

| 킬스위치 | 대상 | 타입 |
|---------|------|------|
| `tengu_frond_boric` | Datadog/1P sink | `{ datadog?: bool, firstParty?: bool }` |
| `tengu_max_version_config` | 버전 업데이트 | `{ external?: string, ant?: string }` |
| `tengu_log_datadog_events` | Datadog 이벤트 | boolean |
| `tengu_kairos_brief` | Brief 모드 | `{ enable_slash_command: bool }` |
| `tengu_harbor` | 채널 알림 | boolean |
| `tengu_attribution_header` | Attribution 헤더 | boolean |

---

## 3. Feature Flags (GrowthBook)

### 3.1 feature() 함수 (Bun DCE)

```typescript
// 빌드타임에 feature("FLAG_NAME") → true/false로 치환
// false인 코드는 번들에서 제거 (Dead Code Elimination)
feature('KAIROS')          // → false (외부 빌드)
feature('COORDINATOR_MODE') // → false (외부 빌드)
```

### 3.2 getFeatureValue_CACHED_MAY_BE_STALE (3계층 캐시)

```typescript
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  feature: string,
  defaultValue: T,
): T {
  // 1. 환경변수 오버라이드
  const overrides = getEnvOverrides()
  if (overrides && feature in overrides) return overrides[feature] as T
  
  // 2. 설정 파일 오버라이드
  const configOverrides = getConfigOverrides()
  if (configOverrides && feature in configOverrides) return configOverrides[feature] as T
  
  // 3. 메모리 캐시 (in-memory payload)
  if (remoteEvalFeatureValues.has(feature)) return remoteEvalFeatureValues.get(feature) as T
  
  // 4. 디스크 캐시 (프로세스 재시작 생존)
  try {
    const cached = getGlobalConfig().cachedGrowthBookFeatures?.[feature]
    return cached !== undefined ? (cached as T) : defaultValue
  } catch { return defaultValue }
}
```

### 3.3 GrowthBook 새로고침 주기

```typescript
const GROWTHBOOK_REFRESH_INTERVAL_MS =
  process.env.USER_TYPE !== 'ant'
    ? 6 * 60 * 60 * 1000  // 외부: 6시간
    : 20 * 60 * 1000      // ANT: 20분
```

### 3.4 사용자 속성 (타게팅)

```typescript
type GrowthBookUserAttributes = {
  id: string               // device UUID
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  organizationUUID?: string
  accountUUID?: string
  userType?: string         // 'ant' | 'external'
  subscriptionType?: string // 'free' | 'pro' | 'team' | 'enterprise'
  rateLimitTier?: string
  email?: string
  appVersion?: string
}
```

### 3.5 A/B 실험 트래킹

```typescript
// Feature 접근 시 실험 데이터 캡처
if (f.source === 'experiment' && f.experimentResult) {
  experimentDataByFeature.set(key, {
    experimentId: exp.key,
    variationId: expResult.variationId,
  })
}

// 1P에 노출 로깅 (세션별 중복 제거)
function logExposureForFeature(feature: string): void {
  if (loggedExposures.has(feature)) return
  const expData = experimentDataByFeature.get(feature)
  if (expData) {
    loggedExposures.add(feature)
    logGrowthBookExperimentTo1P({...})
  }
}
```

---

## 4. Remote Settings

### 4.1 매시간 폴링

```typescript
const POLLING_INTERVAL_MS = 60 * 60 * 1000  // 1시간

export function startBackgroundPolling(): void {
  if (!isRemoteManagedSettingsEligible()) return
  pollingIntervalId = setInterval(() => {
    void pollRemoteSettings()
  }, POLLING_INTERVAL_MS)
  pollingIntervalId.unref()
}
```

### 4.2 체크섬 기반 ETag 캐싱

```typescript
export function computeChecksumFromSettings(settings): string {
  const sorted = sortKeysDeep(settings)
  const normalized = jsonStringify(sorted)
  const hash = createHash('sha256').update(normalized).digest('hex')
  return `sha256:${hash}`
}

// If-None-Match 헤더로 304 Not Modified 지원
if (cachedChecksum) headers['If-None-Match'] = `"${cachedChecksum}"`
// 304 응답 → 캐시 사용 (불필요한 다운로드 방지)
```

### 4.3 자격 확인

```typescript
// 적용 대상:
// - Enterprise/Team OAuth 사용자
// - Console API 키 사용자
// 제외 대상:
// - Bedrock/Vertex/Foundry (3P provider)
// - Cowork (local-agent)
// - 커스텀 base URL
```

### 4.4 위험 설정 변경 → 블로킹 다이얼로그

```typescript
const securityResult = await checkManagedSettingsSecurity(
  cachedSettings,
  newSettings,
)
if (!handleSecurityCheckResult(securityResult)) {
  // 사용자 거부 → 이전 캐시 유지
  return cachedSettings
}
```

---

## 5. React/Ink Terminal UI

### 5.1 왜 React?

- **컴포지션:** Web 팀이 CLI에도 기여 가능 (같은 컴포넌트 모델)
- **타입 안전성:** TypeScript + React Props 검증
- **성능:** 더티 플래그 + WeakMap 캐시 + 메모이제이션
- **접근성:** DOM 구조 + 포커스 관리

### 5.2 렌더링 파이프라인

```
React 컴포넌트
    │
    ▼
reconciler.ts (커스텀 React reconciler)
    ├─ createInstance() → DOMElement 생성
    ├─ diff() → Props 비교 (불필요 업데이트 방지)
    └─ resetAfterCommit() → 레이아웃 + 렌더
    │
    ▼
dom.ts (터미널 DOM 추상화)
    ├─ DOMElement: ink-root, ink-box, ink-text, ink-link, ink-raw-ansi
    ├─ yogaNode: Flexbox 레이아웃 노드
    ├─ markDirty() → 부모까지 전파
    └─ appendChildNode() → Yoga 트리 동기화
    │
    ▼
layout/yoga.ts (Yoga WASM)
    ├─ calculateLayout(width) → Flexbox 계산
    ├─ setMeasureFunc() → 텍스트 측정
    └─ setFlexDirection/Grow/Basis() → 속성 설정
    │
    ▼
renderer.ts → render-node-to-output.ts
    ├─ DOM 트리 → Screen 버퍼
    ├─ ANSI 색상/스타일 적용
    ├─ 이전 프레임(prevScreen) diff → 변경된 부분만 출력
    └─ 더블 버퍼링 (frontFrame/backFrame)
```

### 5.3 성능 최적화

```typescript
// WeakMap 캐시 (메모리 자동 정리)
const nodeCache = new WeakMap<DOMElement, CachedLayout>()
const pendingClears = new WeakMap<DOMElement, Rectangle[]>()

// 더티 플래그 전파 (부모까지)
function markDirty(node: DOMElement): void {
  let current = node
  while (current) {
    current.dirty = true
    if (current.yogaNode) current.yogaNode.markDirty()
    current = current.parentNode
  }
}

// throttled 렌더 (FRAME_INTERVAL_MS)
const scheduleRender = throttle(() => {
  /* render() */
}, FRAME_INTERVAL_MS)
```

### 5.4 Focus 관리

```typescript
class FocusManager {
  activeElement: DOMElement | null
  focusStack: DOMElement[]  // 최대 32개
  
  focus(node) {
    // 이전: blur 이벤트
    // 현재: focus 이벤트
    // 스택에 push
  }
  
  handleNodeRemoved() {
    // 제거된 노드/자손 → 스택에서 복구
  }
}
```

---

## 6. OAuth PKCE

### 6.1 전체 흐름

```
1. code_verifier = base64URL(randomBytes(32))
2. code_challenge = base64URL(SHA-256(code_verifier))
3. state = base64URL(randomBytes(32))
4. auth_url = buildAuthUrl(code_challenge, state, port)
5. 브라우저 열기 → 사용자 인증
6. localhost:{port}/callback?code=xxx&state=yyy 수신
7. state 검증 (CSRF 방어)
8. POST /token { code, code_verifier, client_id }
9. access_token + refresh_token 수신
```

### 6.2 핵심 코드

```typescript
// crypto.ts
export const generateCodeVerifier = () => base64URLEncode(randomBytes(32))
export const generateCodeChallenge = (v) => base64URLEncode(createHash('sha256').update(v).digest())
export const generateState = () => base64URLEncode(randomBytes(32))

// client.ts - token exchange
const requestBody = {
  grant_type: 'authorization_code',
  code: authorizationCode,
  redirect_uri: `http://localhost:${port}/callback`,
  client_id: getOauthConfig().CLIENT_ID,
  code_verifier: codeVerifier,  // 서버가 challenge와 검증
  state,
}

// refresh
const refreshBody = {
  grant_type: 'refresh_token',
  refresh_token: refreshToken,
  client_id: getOauthConfig().CLIENT_ID,
  scope: CLAUDE_AI_OAUTH_SCOPES.join(' '),
}
```

### 6.3 프로필 캐시 최적화

```typescript
// 매 refresh마다 프로필 fetch 방지
const haveProfileAlready = config.oauthAccount?.billingType !== undefined
const profileInfo = haveProfileAlready ? null : await fetchProfileInfo(accessToken)
```

---

## 7. Voice Mode

### 7.1 아키텍처

```
키 누르기 (hold-to-talk)
    │
    ▼
마이크 캡처 (PCM 16-bit, 16kHz, mono)
    │
    ▼
WebSocket → /api/ws/speech_to_text/voice_stream
    │
    ├─ JSON 메시지: KeepAlive, CloseStream
    └─ Binary 메시지: 오디오 프레임
    │
    ▼
서버 → TranscriptText (중간) → TranscriptEndpoint (최종)
    │
    ▼
텍스트 입력으로 주입
```

### 7.2 42개 언어

```
en, es, fr, ja, de, pt, it, ko, hi, id,
ru, pl, tr, nl, uk, el, cs, da, sv, no
(+ 네이티브 이름: 한국어, 日本語, español 등)
```

### 7.3 Hold-to-Talk 키 감지

```
첫 키 누르기 → 즉시 녹음 시작
macOS 키 반복 → 600ms 대기 (반복 딜레이 ~500ms)
200ms 갭 감지 → "키 릴리스" 판정
릴리스 → STT 종료 → 텍스트 주입
```

### 7.4 Focus Mode (자동 녹음)

```
터미널 포커스 획득 → 자동 녹음 시작
5초 침묵 → 자동 종료
중간 transcript → 즉시 주입 (연속 전사)
포커스 잃음 → 녹음 종료
```

### 7.5 Silent-Drop Replay (~1% CE pod 버그 우회)

```typescript
// 오디오 있었는데 transcript 안 옴 → 새 WS로 재전송
if (finalizeSource === 'no_data_timeout' && hadAudioSignal &&
    wsConnected && accumulatedRef.current.trim() === '') {
  silentDropRetriedRef.current = true
  await sleep(250)  // 동일 pod 경쟁 클리어
  // 새 WS 연결 → fullAudioRef 재전송
}
```

### 7.6 오디오 레벨 시각화 (RMS + sqrt 곡선)

```typescript
export function computeLevel(chunk: Buffer): number {
  let sumSq = 0
  for (let i = 0; i < chunk.length - 1; i += 2) {
    const sample = ((chunk[i]! | (chunk[i + 1]! << 8)) << 16) >> 16
    sumSq += sample * sample
  }
  const rms = Math.sqrt(sumSq / (chunk.length >> 1))
  return Math.sqrt(Math.min(rms / 2000, 1))
}
```

---

## 8. Error Recovery

### 8.1 Max Output Tokens 에스컬레이션

```
기본: 8,000 (CAPPED_DEFAULT_MAX_TOKENS)
에스컬레이션: 64,000 (ESCALATED_MAX_TOKENS)
최대 재시도: 3회 (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT)

이유: BQ p99 출력 = 4,911토큰 → 8k 기본값으로 슬롯 예약 최적화
     64k는 한 번만 재시도 허용
```

### 8.2 Stop Reason 분기

| stop_reason | 동작 |
|-------------|------|
| `end_turn` | 정상 완료 |
| `max_tokens` | 에스컬레이션 재시도 (8k→64k) |
| `model_context_window_exceeded` | Reactive Compact 시도 |
| `stop_sequence` | 사용자 정의 중단 |

### 8.3 Stream Idle Watchdog

```
기본 타임아웃: 90초
경고: 45초
동작: abort → non-streaming fallback 재시도
환경변수: CLAUDE_STREAM_IDLE_TIMEOUT_MS, CLAUDE_ENABLE_STREAM_WATCHDOG
```

### 8.4 Fast Mode Cooldown (캐시 보존)

```
짧은 retry-after → 같은 모델로 재시도 (캐시 유지)
긴 retry-after → fast mode 비활성화 (cooldown 진입)
이유: 모델 전환하면 프롬프트 캐시 무효화 → 비용 증가
```

### 8.5 복구 경로 요약

```
에러 → 1차: 지수 백오프 재시도 (500ms * 2^n, ±25% jitter)
     → 2차: max_tokens 에스컬레이션 (8k→64k)
     → 3차: fast mode cooldown
     → 4차: 529 3연속 → fallback 모델
     → 5차: reactive compact → 재시도
     → 6차: persistent retry (무한, ant-only)
```

---

## 9. Worktree 격리

### 9.1 EnterWorktree

```
1. 중복 체크 (이미 worktree 세션이면 에러)
2. 메인 repo root 이동
3. git worktree add <path> -b <branch>
4. node_modules 등 심링크 (디스크 절약)
5. CWD 전환 + 세션 상태 저장
6. 캐시 무효화 (시스템 프롬프트, 메모리, 플랜)
```

### 9.2 ExitWorktree

```
validate:
  - 세션 존재 확인
  - remove 시: git status --porcelain + rev-list --count
  - 변경사항 있으면 discard_changes 필수

call:
  - keep: worktree 보관, CWD 복원
  - remove: tmux kill + worktree 삭제 + 브랜치 삭제 + CWD 복원
```

### 9.3 안전장치

```typescript
// 경로 탈출 검증
function validateWorktreeSlug(slug: string): void {
  if (slug.length > 64) throw new Error('...')
  for (const segment of slug.split('/')) {
    if (segment === '.' || segment === '..') throw new Error('...')
    if (!VALID_WORKTREE_SLUG_SEGMENT.test(segment)) throw new Error('...')
  }
}

// 변경사항 없이 remove 방지
if (changedFiles > 0 || commits > 0) {
  return {
    result: false,
    message: `Worktree has ${changedFiles} uncommitted files and ${commits} commits.`
  }
}
```

---

## 10. Cron 스케줄링

### 10.1 구조

```
.claude/scheduled_tasks.json

{
  "tasks": [
    {
      "id": "uuid",
      "expression": "0 9 * * 1-5",  // 5-field cron (M H DoM Mon DoW)
      "command": "/review-pr 123",
      "durable": true,              // 파일에 저장 (세션 간 생존)
      "recurring": true,            // 재스케줄
      "nextFireAt": 1713340800000,
      "lastFiredAt": null,
      "createdAt": 1713254400000,
      "recurringMaxAgeMs": 604800000  // 1주 후 만료
    }
  ]
}
```

### 10.2 스케줄러 루프

```typescript
// 1초 체크 간격
const SCHEDULER_CHECK_INTERVAL_MS = 1000

setInterval(() => {
  const now = Date.now()
  for (const task of tasks) {
    if (task.nextFireAt <= now) {
      executeTask(task)
      if (task.recurring) {
        task.nextFireAt = computeNextFireAt(task.expression, now)
        task.lastFiredAt = now
      } else {
        deleteTask(task.id)  // one-shot 자동 삭제
      }
    }
  }
}, SCHEDULER_CHECK_INTERVAL_MS)
```

### 10.3 멀티세션 Lock

```
chokidar 파일 감시 → scheduled_tasks.json 변경 감지
lock 기반 조율 → takeover detection
5-10% jitter → 동시 재시작 방지
```

---

## 11. Session 영속화

### 11.1 저장 경로

```
~/.claude/sessions/<sessionId>/
├── transcript.ndjson    # 메시지 스트리밍 (줄 단위 JSON)
├── sessionState.json    # 세션 상태
└── events_batch_*.json  # 텔레메트리 배치 캐시
```

### 11.2 transcript.ndjson 형식

```json
{"type":"user","message":{"role":"user","content":"Hello"},"timestamp":1713254400000}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"Hi"}]},"timestamp":1713254401000}
{"type":"tool_use","id":"toolu_xxx","name":"BashTool","input":{"command":"ls"}}
{"type":"tool_result","tool_use_id":"toolu_xxx","content":"file1.txt\nfile2.txt"}
```

### 11.3 비대칭 저장 (프로덕션 흔적!)

```typescript
// 사용자 메시지: 동기 저장 (절대 유실 안 됨)
await fs.writeFile(transcriptPath, serialize(userMessage))

// 어시스턴트 메시지: 비동기 저장 (약간 유실 허용)
void fs.appendFile(transcriptPath, serialize(assistantMessage))
```

**왜?** 세션 유실 사고에서 학습 → 사용자 입력은 절대 잃으면 안 됨.

### 11.4 세션 재개 (resume)

```
/resume 명령 또는 claude --resume <sessionId>
  → transcript.ndjson 파싱
    → 메시지 히스토리 복원
      → 마지막 상태에서 계속
```

---

## 12. API 클라이언트 상세

### 12.1 Provider 다형성

```typescript
// 4개 프로바이더 동적 로딩
if (CLAUDE_CODE_USE_BEDROCK) {
  const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk')
  // AWS 자격증명
}
if (CLAUDE_CODE_USE_FOUNDRY) {
  const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk')
  // Azure AD 토큰
}
if (CLAUDE_CODE_USE_VERTEX) {
  const { AnthropicVertex } = await import('@anthropic-ai/vertex-sdk')
  // Google Auth
}
// 기본: new Anthropic({ apiKey })
```

### 12.2 Thinking Config

```
adaptive: Opus 4.6, Sonnet 4.6 (무제한 사고)
enabled(budget): 다른 모델 (토큰 예산 지정)
disabled: 사고 없음

구현:
  adaptive → { type: 'adaptive' }
  budget → { type: 'enabled', budget_tokens: min(maxOutput-1, modelDefault) }
```

### 12.3 Effort Value

```
undefined → 자동 노력
"low"|"medium"|"high" → output_config.effort
0-1 숫자 → anthropic_internal.effort_override (ant-only)
```

### 12.4 Beta 헤더 (21+)

```
afk-mode-2025-03-13
1m-context-2025-03-13
context-management-2025-03-13
effort-2025-04-08
fast-mode-2025-03-13
prompt-caching-scope
redact-thinking
structured-outputs
task-budgets
... 등 21+ 동적 추가
```

---

## 요약: 전체 시스템 맵 (보충)

```
REPORT_7 (코어):
├── 에이전트 루프 (AsyncGenerator + State)
├── 시스템 프롬프트 (15 함수 + 캐시 경계)
├── 도구/권한 (42+ 도구 + 4모드 + 23 보안체크)
├── Compaction (4단계 + 13K 버퍼)
└── 메모리 + MCP + Bridge

REPORT_8 (보충):
├── Hooks (20+ 이벤트, 3타입, JSON 응답)
├── Telemetry (1P OTel + Datadog, 6 킬스위치)
├── Feature Flags (GrowthBook 3계층 캐시, A/B 트래킹)
├── Remote Settings (1h 폴링, ETag, 보안 다이얼로그)
├── React/Ink UI (커스텀 reconciler + Yoga WASM)
├── OAuth PKCE (code_verifier/challenge + localhost)
├── Voice Mode (hold-to-talk, 42언어, silent-drop 우회)
├── Error Recovery (8k→64k, 90s watchdog, 529 fallback)
├── Worktree (격리 생성/정리 + 안전장치)
├── Cron (5-field, durable, 멀티세션 lock)
├── Session 영속화 (ndjson, 비대칭 저장)
└── API 클라이언트 (4 프로바이더, adaptive thinking, effort)
```

*총 분석: 11개 보충 서브시스템 | 병렬 에이전트 4개 | 2026-04-17*
