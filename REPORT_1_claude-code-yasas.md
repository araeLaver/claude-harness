# Claude Code 원본 유출 소스 종합 분석 보고서
## 레포: yasasbanukaofficial/claude-code

---

## 1. 전체 디렉토리 구조

```
claude-code-yasas/
├── README.md
├── assets/
│   ├── claude-logo.png
│   ├── claude-npm-img.png
│   └── x-post.png
└── src/
    ├── main.tsx                    # 메인 엔트리포인트
    ├── setup.ts                    # 세션 초기화
    ├── query.ts                    # API 쿼리 루프 (핵심 에이전트 루프)
    ├── QueryEngine.ts              # 쿼리 엔진 (SDK/REPL 공용)
    ├── Tool.ts                     # 도구 타입 정의
    ├── tools.ts                    # 도구 등록/팩토리
    ├── commands.ts                 # 슬래시 명령어 등록
    ├── context.ts                  # 시스템/사용자 컨텍스트 관리
    ├── cost-tracker.ts             # 비용 추적
    ├── history.ts                  # 대화 기록 관리
    ├── ink.ts                      # Ink 터미널 UI 루트
    ├── replLauncher.tsx            # REPL 런처
    ├── assistant/                  # KAIROS (어시스턴트 모드) - 세션 히스토리
    ├── bootstrap/                  # 부트스트랩 상태 관리
    ├── bridge/                     # Bridge 모드 (원격 세션)
    ├── buddy/                      # 반려동물(컴패니언) 시스템
    ├── cli/                        # CLI 전송/핸들러
    ├── commands/                   # 80+ 슬래시 명령어
    ├── components/                 # Ink React UI 컴포넌트
    ├── constants/                  # 상수/프롬프트/API 제한
    ├── context/                    # 컨텍스트 관리 세부
    ├── coordinator/                # 코디네이터 모드 (다중 워커)
    ├── entrypoints/                # CLI/SDK/MCP 엔트리포인트
    ├── hooks/                      # React 훅 (50+)
    ├── ink/                        # 커스텀 Ink 프레임워크
    ├── keybindings/                # 키바인딩 관리
    ├── memdir/                     # 구조화된 메모리 디렉토리 시스템
    ├── migrations/                 # 데이터 마이그레이션
    ├── moreright/                  # 추가 기능
    ├── native-ts/                  # 네이티브 TS 유틸리티
    ├── outputStyles/               # 출력 스타일 설정
    ├── plugins/                    # 플러그인 시스템
    ├── query/                      # 쿼리 구성/종속성/전환
    ├── remote/                     # 원격 모드
    ├── schemas/                    # Zod 스키마
    ├── screens/                    # UI 화면
    ├── server/                     # 서버 컴포넌트
    ├── services/                   # 핵심 서비스 (API, MCP, 컴팩트 등)
    ├── skills/                     # 스킬 시스템
    ├── state/                      # AppState 관리 (zustand-like)
    ├── tasks/                      # 태스크 관리
    ├── tools/                      # 40+ 도구 구현
    ├── types/                      # TypeScript 타입 정의
    ├── upstreamproxy/              # 업스트림 프록시
    ├── utils/                      # 유틸리티 (300+ 파일)
    ├── vim/                        # Vim 모드 지원
    └── voice/                      # 음성 모드
```

---

## 2. 아키텍처 개요

### 엔트리포인트
메인 엔트리포인트는 `/src/main.tsx`로, `bun:bundle` 기능 플래그와 함께 Bun 런타임을 사용한다. Commander.js를 CLI 프레임워크로, React + Ink를 터미널 UI 렌더링에 사용한다.

```typescript
// main.tsx - 시작 시퀀스
profileCheckpoint('main_tsx_entry');
startMdmRawRead();        // MDM 설정 병렬 읽기
startKeychainPrefetch();  // macOS 키체인 프리페치
```

### 빌드 시스템
- **Bun 번들러** 사용 (`bun:bundle`의 `feature()` 함수로 빌드 타임 기능 플래그 제어)
- **Dead Code Elimination (DCE)**: `feature('KAIROS')`, `feature('COORDINATOR_MODE')` 등으로 빌드별 코드 분기
- `process.env.USER_TYPE === 'ant'` 로 내부(Anthropic) 빌드와 외부 빌드 구분

### 핵심 프레임워크/라이브러리
- **React + Ink**: 터미널 UI
- **@anthropic-ai/sdk**: Anthropic API 클라이언트
- **@modelcontextprotocol/sdk**: MCP 클라이언트
- **Commander.js** (`@commander-js/extra-typings`): CLI 파서
- **Zod v4**: 스키마 검증
- **lodash-es**: 유틸리티 함수
- **GrowthBook**: 피처 플래그/A/B 테스팅
- **Axios**: HTTP 클라이언트
- **@anthropic-ai/sandbox-runtime**: 샌드박스 실행

---

## 3. 핵심 모듈 및 역할

### query.ts - 에이전트 루프의 심장부
에이전트 루프의 핵심 파일로, `query()` 비동기 제너레이터 함수가 메시지 스트리밍, 도구 호출, 자동 컴팩션을 관리한다.

```typescript
export async function* query(params: QueryParams): AsyncGenerator<StreamEvent | Message, Terminal> {
  // 쿼리 루프 상태: messages, toolUseContext, autoCompactTracking 등
  let state: State = {
    messages: params.messages,
    turnCount: 1,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    // ...
  }
  // 반복 루프: API 호출 -> 도구 실행 -> 결과 피드백
}
```

핵심 동작:
- `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`: 최대 출력 토큰 복구 재시도
- `StreamingToolExecutor`: 스트리밍 중 도구 실행
- `autoCompact`: 컨텍스트 한계 접근 시 자동 요약
- `reactiveCompact`: 프롬프트 너무 긴 경우 반응형 컴팩션
- `tokenBudget`: `+500k` 같은 토큰 예산 자동 계속 기능

### QueryEngine.ts - SDK/REPL 통합 엔진
`query.ts`를 감싸는 상위 레이어로, 메시지 관리, 시스템 프롬프트 구축, 파일 히스토리 스냅샷 등을 담당한다.

### context.ts - 시스템/사용자 컨텍스트
Git 상태(브랜치, 최근 커밋, 사용자), CLAUDE.md 내용을 수집하여 시스템 프롬프트에 주입한다.

```typescript
// git status, branch, log, user.name 을 병렬로 가져옴
const [branch, mainBranch, status, log, userName] = await Promise.all([...])
```

---

## 4. 시스템 프롬프트 (매우 중요)

### 프롬프트 구조 (`/src/constants/prompts.ts`)

시스템 프롬프트는 **정적 부분**(캐시 가능)과 **동적 부분**(세션별)으로 나뉜다:

```
[정적 - 전역 캐시 가능]
1. getSimpleIntroSection()     - "You are an interactive agent..."
2. getSimpleSystemSection()    - 시스템 규칙/도구 설명
3. getSimpleDoingTasksSection() - 작업 수행 규칙
4. getActionsSection()         - 주의 깊은 행동 실행 규칙
5. getUsingYourToolsSection()  - 도구 사용 가이드
6. getSimpleToneAndStyleSection() - 톤/스타일
7. getOutputEfficiencySection() - 출력 효율성

__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ (캐시 경계)

[동적 - 세션별]
8. session_guidance              - 세션별 가이드
9. memory (CLAUDE.md)            - 메모리 파일
10. env_info_simple              - 환경 정보
11. language                     - 언어 설정
12. output_style                 - 출력 스타일
13. mcp_instructions             - MCP 서버 지시
14. scratchpad                   - 스크래치패드
15. frc (Function Result Clearing) - 함수 결과 정리
16. summarize_tool_results       - 도구 결과 요약
```

### 핵심 프롬프트 내용

**Intro 섹션**:
```
You are an interactive agent that helps users with software engineering tasks.
IMPORTANT: Assist with authorized security testing... Refuse requests for destructive techniques...
IMPORTANT: You must NEVER generate or guess URLs for the user unless confident...
```

**CYBER_RISK_INSTRUCTION** (보안 가이드):
```
Assist with authorized security testing, defensive security, CTF challenges...
Refuse requests for destructive techniques, DoS attacks, mass targeting...
```

**코드 스타일 규칙 (Ant 빌드 전용 추가 규칙)**:
```
- 주석을 기본적으로 작성하지 말 것
- WHY가 자명하지 않을 때만 주석
- 작업 완료 보고 전 실제 동작 확인
- 결과를 충실히 보고 (허위 보고 금지)
```

**출력 효율성 (외부)**:
```
IMPORTANT: Go straight to the point. Try the simplest approach first without going in circles.
Keep your text output brief and direct.
```

**출력 효율성 (Ant 내부)**:
```
Length limits: keep text between tool calls to ≤25 words.
Keep final responses to ≤100 words unless the task requires more detail.
```

### 모델 이름/정보

```typescript
const FRONTIER_MODEL_NAME = 'Claude Opus 4.6'
const CLAUDE_4_5_OR_4_6_MODEL_IDS = {
  opus: 'claude-opus-4-6',
  sonnet: 'claude-sonnet-4-6',
  haiku: 'claude-haiku-4-5-20251001',
}
```

### Proactive 모드 프롬프트
```
You are an autonomous agent. Use the available tools to do useful work.
```
(KAIROS/Proactive 모드에서는 간소화된 프롬프트 사용)

---

## 5. 도구 시스템

### 도구 정의 패턴
각 도구는 `src/tools/<ToolName>/` 디렉토리에 다음 파일들을 가진다:
- `<ToolName>.ts(x)` - 메인 도구 구현 (Tool 인터페이스 구현)
- `prompt.ts` - 도구 설명/프롬프트
- `constants.ts` - 상수

### 등록된 전체 도구 목록 (`tools.ts`)

**핵심 도구**:

| 도구명 | 역할 |
|--------|------|
| `AgentTool` | 서브에이전트 생성/위임 |
| `BashTool` | 셸 명령 실행 |
| `FileReadTool` | 파일 읽기 |
| `FileEditTool` | 파일 편집 (문자열 교체) |
| `FileWriteTool` | 파일 쓰기 |
| `GlobTool` | 파일 패턴 검색 |
| `GrepTool` | 콘텐츠 검색 (ripgrep) |
| `NotebookEditTool` | Jupyter 노트북 편집 |
| `WebFetchTool` | 웹 콘텐츠 가져오기 |
| `WebSearchTool` | 웹 검색 |
| `TodoWriteTool` | TODO 관리 |
| `TaskOutputTool` | 백그라운드 태스크 출력 |
| `SkillTool` | 스킬 실행 |
| `AskUserQuestionTool` | 사용자에게 질문 |
| `EnterPlanModeTool` / `ExitPlanModeV2Tool` | 플랜 모드 전환 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git 워크트리 관리 |
| `TaskCreateTool` / `TaskGetTool` / `TaskUpdateTool` / `TaskListTool` | 태스크 CRUD (v2) |
| `ToolSearchTool` | 지연 로드된 도구 검색 |
| `ListMcpResourcesTool` / `ReadMcpResourceTool` | MCP 리소스 관리 |

**피처 플래그 도구** (조건부):

| 도구명 | 플래그 | 역할 |
|--------|--------|------|
| `SleepTool` | PROACTIVE/KAIROS | 대기 기능 |
| `CronCreate/Delete/ListTool` | AGENT_TRIGGERS | 크론 작업 관리 |
| `RemoteTriggerTool` | AGENT_TRIGGERS_REMOTE | 원격 트리거 |
| `MonitorTool` | MONITOR_TOOL | 모니터링 |
| `SendUserFileTool` | KAIROS | 사용자에게 파일 전송 |
| `PushNotificationTool` | KAIROS | 푸시 알림 |
| `SubscribePRTool` | KAIROS_GITHUB_WEBHOOKS | PR 구독 |
| `WebBrowserTool` | WEB_BROWSER_TOOL | 웹 브라우저 |
| `TerminalCaptureTool` | TERMINAL_PANEL | 터미널 캡처 |
| `WorkflowTool` | WORKFLOW_SCRIPTS | 워크플로우 스크립트 |
| `SnipTool` | HISTORY_SNIP | 히스토리 스닙 |
| `ListPeersTool` | UDS_INBOX | 피어 목록 |

**Ant 전용 도구**:
- `REPLTool`: REPL 모드 (Bash/Read/Edit를 VM 안에서 실행)
- `SuggestBackgroundPRTool`: 백그라운드 PR 제안
- `ConfigTool`: 설정 도구
- `TungstenTool`: 텅스텐 도구
- `VerifyPlanExecutionTool`: 플랜 실행 검증

### 도구 권한 컨텍스트
```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean
}>
```

---

## 6. 에이전트 워크플로우

### 메인 루프 흐름
```
main.tsx (CLI 파싱)
  → setup.ts (환경 초기화, CWD, Git, 세션)
    → replLauncher.tsx (REPL 모드) / print mode (비대화형)
      → QueryEngine.ts (쿼리 준비)
        → query.ts (에이전트 루프)
          → API 호출 (claude.ts)
          → StreamingToolExecutor (도구 실행)
          → 자동 컴팩션 (임계값 도달 시)
          → 토큰 예산 추적
          → max_output_tokens 복구 (최대 3회)
```

### 코디네이터 모드 (`COORDINATOR_MODE`)
```
코디네이터 (사용자 대면)
  ├── AgentTool → Worker 1 (연구)
  ├── AgentTool → Worker 2 (구현)
  └── SendMessageTool → 기존 Worker에게 후속 지시
```

코디네이터는 직접 코드를 작성하지 않고, 워커들을 오케스트레이션한다. 워커 결과는 `<task-notification>` XML로 전달된다.

### 포크 서브에이전트 (`forkSubagent`)
메인 스레드의 캐시를 공유하는 배경 에이전트를 포크하여, 연구/구현 작업을 위임하면서 사용자와의 대화를 계속할 수 있다.

### 검증 에이전트 (`VERIFICATION_AGENT`)
비사소한 구현(3+ 파일 편집, 백엔드/API 변경) 후 독립적 적대적 검증을 수행하는 전용 에이전트.

---

## 7. 메모리 시스템

### CLAUDE.md 시스템 (`claudemd.ts`)
4단계 메모리 파일 체계:
1. **관리형 메모리** (`/etc/claude-code/CLAUDE.md`) - 조직 전체 지시
2. **사용자 메모리** (`~/.claude/CLAUDE.md`) - 개인 전역 지시
3. **프로젝트 메모리** (`CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`) - 저장소 체크인
4. **로컬 메모리** (`CLAUDE.local.md`) - 비공개 프로젝트별 지시

`@include` 지시자로 다른 파일을 포함할 수 있다.

### 구조화된 메모리 디렉토리 (`memdir/`)
```
~/.claude/projects/<slug>/memory/MEMORY.md  (인덱스, 200줄/25KB 제한)
~/.claude/projects/<slug>/memory/topic.md    (개별 메모리 파일)
```

4가지 메모리 타입 분류:
- **user**: 사용자 선호도/역할
- **feedback**: 사용자 피드백/수정 지시
- **project**: 프로젝트별 규칙
- **reference**: 참조 정보

### 자동 메모리 추출 (`extractMemories/`)
백그라운드 메모리 추출 에이전트가 대화에서 중요 정보를 자동으로 구조화된 메모리 파일로 저장한다.

### 자동 컴팩션 (`services/compact/`)
```typescript
// 컨텍스트 한계 접근 시 대화를 요약
const BASE_COMPACT_PROMPT = `Your task is to create a detailed summary of the conversation...`
```
9가지 섹션으로 구조화된 요약을 생성:
1. 주요 요청 및 의도
2. 핵심 기술 개념
3. 파일 및 코드 섹션
4. 에러 및 수정
5. 문제 해결
6. 모든 사용자 메시지
7. 대기 중인 작업
8. 현재 작업
9. 선택적 다음 단계

---

## 8. 권한 시스템

### 권한 모드 (`PermissionMode`)
```
- default: 기본 (도구별 확인)
- plan: 플랜 모드 (읽기만 허용)
- acceptEdits: 편집 자동 수락
- bypassPermissions: 모든 권한 우회
```

### 권한 규칙 체계
- **alwaysAllow**: 항상 허용 규칙
- **alwaysDeny**: 항상 거부 규칙
- **alwaysAsk**: 항상 확인 규칙

규칙 소스: 관리형 설정, 사용자 설정, 프로젝트 설정

### 분류기 시스템 (`TRANSCRIPT_CLASSIFIER`)
- `bashClassifier.ts`: Bash 명령어 위험도 분류
- `yoloClassifier.ts`: 자동 모드 분류기
- `dangerousPatterns.ts`: 위험한 패턴 탐지
- `classifierDecision.ts`: 분류 결정 로직

### 샌드박스 (`sandbox-adapter.ts`)
`@anthropic-ai/sandbox-runtime`을 래핑하여 파일시스템, 네트워크 제한을 적용한다.
- 파일시스템 읽기/쓰기 제한
- 네트워크 호스트 패턴 제한
- 위반 이벤트 추적

---

## 9. 숨겨진/흥미로운 기능

### 버디 시스템 (Buddy/Companion) - `buddy/`
사용자 입력 옆에 ASCII 아트 반려동물이 표시되는 이스터에그 기능!

**종류**: duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, **capybara**, cactus, robot, rabbit, mushroom (18종)

**희귀도 체계**:
```typescript
const RARITY_WEIGHTS = {
  common: 60, uncommon: 25, rare: 10, epic: 4, legendary: 1
}
```

각 동물에는 스탯(StatName), 눈 스타일(Eye), 모자(Hat)가 있으며, 사용자 ID 해시 기반 시드 PRNG(Mulberry32)로 결정적으로 생성된다.

```
    __
  <(o )___
   (  ._>
    `--´
```

### KAIROS 모드 (어시스턴트 데몬)
`feature('KAIROS')`로 게이트된 자율 에이전트 모드:
- `SleepTool`: 대기 기능
- `PushNotificationTool`: 푸시 알림
- `SendUserFileTool`: 파일 전송
- `SubscribePRTool`: PR 이벤트 구독 (GitHub Webhooks)
- 원격 어시스턴트 세션 히스토리
- "Dream" 스킬 (`KAIROS_DREAM`)
- "Brief" 도구 (`KAIROS_BRIEF`)
- 크론 작업 스케줄링 (`AGENT_TRIGGERS`)

### 언더커버 모드 (Undercover)
```typescript
export function getUndercoverInstructions(): string {
  return `## UNDERCOVER MODE — CRITICAL
You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository.
NEVER include: Internal model codenames, unreleased model versions,
internal repo names, the phrase "Claude Code", any hint of what model you are...`
}
```

### 울트라싱크 (Ultrathink)
`ultrathink` 키워드를 입력하면 확장된 thinking 모드가 활성화된다. 무지개 색상 애니메이션으로 UI에 표시된다.

### 보이스 모드 (`VOICE_MODE`)
`voice_stream` 엔드포인트를 사용한 음성 입력 모드. Anthropic OAuth가 필수이며, GrowthBook kill-switch(`tengu_amber_quartz_disabled`)로 제어된다.

### 토큰 예산 (`TOKEN_BUDGET`)
사용자가 `+500k`, "spend 2M tokens" 같은 지시를 하면 자동으로 계속 작업하는 기능.

---

## 10. 모델 설정

### 지원 모델 전체 목록 (`model/configs.ts`)
```typescript
ALL_MODEL_CONFIGS = {
  haiku35:  'claude-3-5-haiku-20241022',
  haiku45:  'claude-haiku-4-5-20251001',
  sonnet35: 'claude-3-5-sonnet-20241022',
  sonnet37: 'claude-3-7-sonnet-20250219',
  sonnet40: 'claude-sonnet-4-20250514',
  sonnet45: 'claude-sonnet-4-5-20250929',
  sonnet46: 'claude-sonnet-4-6',
  opus40:   'claude-opus-4-20250514',
  opus41:   'claude-opus-4-1-20250805',
  opus45:   'claude-opus-4-5-20251101',
  opus46:   'claude-opus-4-6',
}
```

### 기본 모델 선택 로직
```
- Max 구독자 → Opus 4.6 [1m]
- Team Premium → Opus 4.6 [1m]
- Pro/Enterprise/Team Standard → Sonnet 4.6
- PAYG (3P) → Sonnet 4.5 (3P는 4.6 미지원 가능)
- Ant 내부 → Opus 4.6 [1m] (또는 flag config)
```

### 특수 모델 설정
- `opusplan`: 플랜 모드에서만 Opus 사용
- `haiku` (플랜 모드): 자동으로 Sonnet으로 업그레이드
- `[1m]` 접미사: 1M 컨텍스트 윈도우 활성화

### Effort 레벨
```typescript
const EFFORT_LEVELS = ['low', 'medium', 'high', 'max']
// 'max' effort는 Opus 4.6에서만 지원
```

### Thinking 설정
```typescript
type ThinkingConfig =
  | { type: 'adaptive' }       // 적응형 (기본)
  | { type: 'enabled'; budgetTokens: number }  // 명시적 예산
  | { type: 'disabled' }       // 비활성화
```

---

## 11. MCP (Model Context Protocol) 통합

### MCP 클라이언트 (`services/mcp/client.ts`)
```typescript
// 3가지 전송 지원
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js'
```

### MCP 기능
- **도구**: MCP 서버의 도구를 Claude Code 도구로 변환 (`MCPTool`)
- **리소스**: `ListMcpResourcesTool`, `ReadMcpResourceTool`로 리소스 접근
- **인증**: OAuth, 커스텀 헤더, XAA(X-API-Auth) 지원
- **알림**: 채널 알림, 허용 목록 관리
- **지시**: MCP 서버별 지시를 시스템 프롬프트에 주입

---

## 12. 핵심 상수 및 설정

### 베타 헤더 (`constants/betas.ts`)
```typescript
'claude-code-20250219'              // Claude Code 기본
'interleaved-thinking-2025-05-14'   // 인터리브드 thinking
'context-1m-2025-08-07'             // 1M 컨텍스트
'context-management-2025-06-27'     // 컨텍스트 관리
'web-search-2025-03-05'             // 웹 검색
'advanced-tool-use-2025-11-20'      // 도구 검색 (1P)
'effort-2025-11-24'                 // Effort 파라미터
'task-budgets-2026-03-13'           // 태스크 예산
'prompt-caching-scope-2026-01-05'   // 프롬프트 캐싱 스코프
'fast-mode-2026-02-01'              // 패스트 모드
'redact-thinking-2026-02-12'        // Thinking 수정
'token-efficient-tools-2026-03-28'  // 토큰 효율적 도구
'afk-mode-2026-01-31'               // AFK 모드 (분류기)
'advisor-tool-2026-03-01'           // 어드바이저 도구
```

### 피처 플래그 (빌드 타임) - 30+
```
KAIROS, KAIROS_BRIEF, KAIROS_DREAM, KAIROS_PUSH_NOTIFICATION,
KAIROS_GITHUB_WEBHOOKS, PROACTIVE, COORDINATOR_MODE, BRIDGE_MODE,
DAEMON, VOICE_MODE, BUDDY, ULTRATHINK, ULTRAPLAN, AGENT_TRIGGERS,
AGENT_TRIGGERS_REMOTE, MONITOR_TOOL, WEB_BROWSER_TOOL,
TERMINAL_PANEL, HISTORY_SNIP, UDS_INBOX, WORKFLOW_SCRIPTS,
CONTEXT_COLLAPSE, REACTIVE_COMPACT, CACHED_MICROCOMPACT,
TOKEN_BUDGET, TRANSCRIPT_CLASSIFIER, VERIFICATION_AGENT,
FORK_SUBAGENT, TEAMMEM, BG_SESSIONS, CONNECTOR_TEXT,
EXPERIMENTAL_SKILL_SEARCH, BUILDING_CLAUDE_APPS,
RUN_SKILL_GENERATOR, REVIEW_ARTIFACT, OVERFLOW_TEST_TOOL,
CCR_REMOTE_SETUP, TEMPLATES
```

### 슬래시 명령어 (80+)
주요: `/help`, `/clear`, `/compact`, `/config`, `/cost`, `/diff`, `/commit`, `/memory`, `/context`, `/model`, `/fast`, `/effort`, `/plugin`, `/mcp`, `/skills`, `/permissions`, `/agents`, `/tasks`, `/doctor`, `/bridge`, `/voice`, `/vim`, `/theme`, `/login`, `/export`, `/resume`, `/session`, `/worktree`

---

## 핵심 발견 요약

1. **Claude Code는 Bun 런타임 + React/Ink 터미널 UI 기반의 대규모 TypeScript 애플리케이션**이다 (~800K+ LOC, ~1,900 파일).

2. **시스템 프롬프트는 정적/동적 경계로 나뉘어** 프롬프트 캐시 효율성을 극대화한다.

3. **KAIROS는 미출시 자율 에이전트 데몬 모드**로, 크론 작업, 푸시 알림, PR 구독 등을 포함한다.

4. **코디네이터 모드는 다중 워커 오케스트레이션**으로, 코디네이터가 워커들에게 작업을 병렬 위임한다.

5. **버디 시스템은 ASCII 아트 반려동물 이스터에그**로, 18종 동물, 5단계 희귀도를 가진다.

6. **메모리는 4계층 CLAUDE.md + 구조화된 memdir + 자동 추출**의 삼중 시스템이다.

7. **40+ 도구가 등록**되어 있으며, MCP 통합을 통해 확장 가능하다.

8. **언더커버 모드는 Anthropic 내부 개발자가 공개 저장소에 기여 시** 모델/프로젝트 정보 누출을 방지한다.
