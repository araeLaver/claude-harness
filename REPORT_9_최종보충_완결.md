# Claude Code Harness 최종 보충 분석 (완결편)
## claurst/spec 990KB 정독 + Sandbox + MCP Transport + Plugin + 모델선택 + Server + 기타

---

## 1. claurst/spec/ 990KB 행위 명세 핵심 인사이트 (80+)

### 1.1 부트스트랩 & 마이그레이션

- **부트스트랩 디스패처** (`cli.tsx`): 빠른 경로 우선 처리 (버전, MCP 서버, 워커), 무거운 모듈은 lazy load
- **마이그레이션 현재 버전 = 11**, 12개 함수 (Sonnet 1m→4.5, 4.5→4.6, Opus 1m 업그레이드, Pro→Opus 기본값 리셋)
- **ABLATION_BASELINE 플래그**: harness-science L0 ablation 실행 (기능 제거 후 성능 측정)
- **프리페치 목록**: 사용자, 컨텍스트, MCP URL, 모델 기능, 설정 변경 감시자, 팁

### 1.2 명령 시스템

- **커맨드 3타입**: `local` (동기), `local-jsx` (Ink UI), `prompt` (모델 호출) — 모두 lazy loading
- **로드 순서**: bundledSkills → builtinPluginSkills → skillDirCommands → workflowCommands → pluginCommands → pluginSkills → COMMANDS()
- **ANT 전용 내부 명령 20개**: `/backfill-sessions`, `/break-cache`, `/bughunter`, `/commit`, `/debug-tool-call` 등
- **화이트리스트**: `REMOTE_SAFE_COMMANDS`, `BRIDGE_SAFE_COMMANDS` 존재

### 1.3 도구 프레임워크 신규 정보

- `buildTool()` 헬퍼로 기본값 채우기 (isEnabled, isConcurrencySafe 등)
- `CLAUDE_CODE_SIMPLE` 환경변수: 최소 도구 세트만 (Bash, Read, Edit)
- **도구 풀 조립**: `assembleToolPool(baseTools, mcpTools)` → 이름순 정렬 (**프롬프트 캐시 안정성!**)
- `toAutoClassifierInput()`: 도구 분류기 입력 생성 메서드

### 1.4 상태 관리

- **글로벌 싱글톤 ~80개 필드**: originalCwd, projectRoot, cwd, sessionId, parentSessionId
- **scrollDrain**: `SCROLL_DRAIN_IDLE_MS = 150`, 백그라운드 작업 양보
- **switchSession()**: 원자적 sessionId + projectDir 업데이트, `onSessionSwitch` 신호 발신
- **regenerateSessionId({ setCurrentAsParent })**: 계획 모드 → 구현 혈통 추적

### 1.5 React 훅 신규 정보

- `useBlink(enabled, 530ms)`: 커서 깜박임
- `useMemoryUsage()`: 10초 폴링, **1.5GB 경고, 2.5GB 치명**
- `useAfterFirstRender()`: ANT 시작 시간 측정, `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER` 퇴출
- `useNotifyAfterTimeout()`: **6초 유휴 후 데스크톱 알림**
- `useDoublePress()`: **800ms 더블클릭 감지**

### 1.6 Ink 터미널 프레임워크

- **렌더 파이프라인**: React 트리 → Reconciler → Virtual DOM → Yoga → Output 버퍼 → Screen 셀 → Diff → Patch → 터미널
- `FRAME_INTERVAL_MS = 16` (60fps)
- **ConcurrentRoot**: React 19 호환
- **이벤트 핸들러**: `_eventHandlers` 별도 저장 (더티 마크 방지)
- **Alt-screen**: BSU/ESU 동기화, 하드웨어 스크롤 힌트
- **콘솔 패치**: `console.log/warn/error` 별도 fd (stdout 혼합 방지)

### 1.7 Bridge 프로토콜

- **V1** (환경 기반): `/v1/environments/bridge`, 폴링 + WebSocket/SSE
- **V2** (환경 없음): POST `/v1/code/sessions` → `/sessions/{id}/bridge` → SSE+CCR
- **WorkSecret**: base64url JSON, `session_ingress_token` JWT 포함
- **세션 타임아웃**: `DEFAULT_SESSION_TIMEOUT_MS = 86,400,000` (24시간)
- **SpawnMode**: single-session / worktree / same-dir

### 1.8 유틸리티

- `CircularBuffer<T>`: 고정 크기 순환 버퍼
- `QueryGuard`: 쿼리 생명주기 상태 머신 (idle/dispatching/running)
- `Signal<Args>`: 경량 이벤트 (상태 없음)
- **셸 기본 타임아웃**: `DEFAULT_TIMEOUT = 30분`
- `SIZE_WATCHDOG_INTERVAL_MS = 5000`: 백그라운드 프로세스 크기 감시
- `createChildAbortController(parent)`: **단방향** — 자식 abort 시 부모 영향 없음

### 1.9 상수 & 타입

- **이미지**: `API_IMAGE_MAX_BASE64_SIZE = 5MB`, 최대 2000×2000px
- **PDF**: `API_PDF_MAX_PAGES = 100`, `PDF_EXTRACT_SIZE_THRESHOLD = 3MB`
- **매체**: `API_MAX_MEDIA_PER_REQUEST = 100`
- **Task ID 생성**: prefix + 8 base-36 랜덤 (36^8 ≈ 2.8조)
- **이진 확장자**: 113개 (이미지, 비디오, 오디오, 아카이브, 실행 파일, 문서, 폰트)
- **GrowthBook 키**: ANT dev(`sdk-yZQvlplybuXjYh6L`), ANT prod(`sdk-xRVcrliHIlrg4og4`), external(`sdk-zAZezfDKGoZuXXKe`)

### 1.10 Buddy 시스템 상세

- **결정론적**: `hash(userId + 'friend-2026-401')` via mulberry32 PRNG
- **Eyes 6가지**: `·, ✦, ×, ◉, @, °`
- **Hats 8가지**: none, crown(`\^^^/`), tophat(`[___]`), propeller(`-+-`), halo, wizard, beanie, tinyduck
- **Sprite**: 3프레임, 5행×12열, 500ms 틱
- **애니메이션**: `[0,0,0,0,1,0,0,0,-1,0,0,2,0,0,0]` (-1 = blink)
- **가볍기 이벤트**: 5틱(~2.5초) 하트 떠올림

---

## 2. Sandbox 구현

### 2.1 아키텍처

```
@anthropic-ai/sandbox-runtime 패키지
    │
    ▼
sandbox-adapter.ts (986줄) — Adapter 패턴
    ├─ SandboxManager 싱글톤
    ├─ convertToSandboxRuntimeConfig() — 설정 변환
    ├─ wrapWithSandbox() — 명령어 래핑
    └─ checkDependencies() — 의존성 검사
```

### 2.2 파일시스템 제한

```typescript
// 항상 허용 (쓰기)
allowWrite: ['.', getClaudeTempDir()]

// 항상 거부 (쓰기)
denyWrite: [
  ...settingsPaths,              // settings.json (보안 경계)
  getManagedSettingsDropInDir(),  // 관리 설정 디렉토리
  resolve(cwd, '.claude', 'skills'),  // 스킬 디렉토리 (같은 권한)
  ...bareGitRepoFiles,          // HEAD, objects, refs, hooks, config
]
```

### 2.3 Git Escape 방지

```typescript
// SECURITY: Git의 is_git_directory()는 HEAD + objects/ + refs/ 체크
// 공격자가 이를 심고 core.fsmonitor로 탈출 시도 가능
bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const p of bareGitRepoFiles) {
  try {
    statSync(p)
    denyWrite.push(p)  // 존재하면 ro-bind
  } catch {
    bareGitRepoScrubPaths.push(p)  // 없으면 런타임 스크러빙
  }
}
```

### 2.4 네트워크 제한

```typescript
network: {
  allowedDomains,       // WebFetch(domain:...) 규칙에서 추출
  deniedDomains,        // deny 규칙에서 추출
  allowUnixSockets,     // Docker 등
  allowLocalBinding,    // localhost 서버
  httpProxyPort,        // HTTP 프록시
  socksProxyPort,       // SOCKS 프록시
}

// 정책 모드: 관리 도메인만 허용
if (shouldAllowManagedSandboxDomainsOnly()) {
  // policySettings만 사용, 사용자 설정 무시
}
```

### 2.5 명령어 제외 로직

```typescript
// Compound 명령어 분할: "docker ps && curl evil.com"
// → ["docker ps", "curl evil.com"] 각각 검사
// 고정점 반복: 환경변수 + 래퍼 명령어 제거
//   FOO=bar bazel ... → bazel
//   timeout 30 bazel ... → bazel
// BINARY_HIJACK_VARS: NODE_OPTIONS, PYTHONPATH 등 위험 변수 감지
```

### 2.6 플랫폼 지원

```
macOS: sandbox-exec (Seatbelt)
Linux: bubblewrap (bwrap)
WSL2: bubblewrap
WSL1: ❌ 미지원
Windows: ❌ 미지원 (네이티브)
```

---

## 3. MCP Transport 프로토콜

### 3.1 6가지 전송 방식

| 전송 | 용도 | 특징 |
|------|------|------|
| `stdio` | 로컬 프로세스 | stdin/stdout, stderr 파이프 |
| `sse` | 원격 서버 | HTTP long-poll + POST |
| `http` | Streamable HTTP | JSON-RPC + SSE 재연결 (최신) |
| `ws` | WebSocket | 양방향 실시간 |
| `sse-ide` / `ws-ide` | IDE 플러그인 | 내부용 |
| `sdk` | In-process | 직접 메시지 전달 |

### 3.2 Stdio Transport 구현

```typescript
transport = new StdioClientTransport({
  command: finalCommand,
  args: finalArgs,
  env: { ...subprocessEnv(), ...serverRef.env },
  stderr: 'pipe',  // UI 출력 방지
})

// stderr 축적: 64MB 상한선 (메모리 폭발 방지)
let stderrOutput = ''
stdioTransport.stderr.on('data', (data) => {
  if (stderrOutput.length < 64 * 1024 * 1024) {
    stderrOutput += data.toString()
  }
})
```

### 3.3 HTTP Streamable Transport (최신)

```typescript
// Accept 헤더 필수 (406 방지)
headers.set('accept', 'application/json, text/event-stream')

// AbortSignal.timeout() 사용 금지!
// → 60초 후 스테일 → 모든 후속 요청 실패
// 대안: setTimeout + AbortController 수동 관리
const controller = new AbortController()
const timer = setTimeout(
  c => c.abort(new DOMException('Timeout', 'TimeoutError')),
  60_000,
  controller,
)
timer.unref()  // 프로세스 유지 방지
```

### 3.4 연결 재연결 로직

```typescript
// 3회 연속 terminal 오류 → 강제 재연결
const MAX_ERRORS_BEFORE_RECONNECT = 3

const isTerminalConnectionError = (msg) =>
  msg.includes('ECONNRESET') ||
  msg.includes('ETIMEDOUT') ||
  msg.includes('EPIPE') ||
  msg.includes('EHOSTUNREACH') ||
  msg.includes('ECONNREFUSED') ||
  msg.includes('SSE stream disconnected')

// 세션 만료 감지: HTTP 404 + JSON-RPC -32001
export function isMcpSessionExpiredError(error) {
  return error.code === 404 && error.message.includes('"code":-32001')
}
```

### 3.5 In-Process Transport (SDK)

```typescript
// 2개의 링크된 transport 쌍 생성
export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)
  b._setPeer(a)
  return [a, b]
}

// 메시지 전달: queueMicrotask (스택 깊이 방지)
async send(message: JSONRPCMessage) {
  queueMicrotask(() => { this.peer?.onmessage?.(message) })
}
```

### 3.6 MCP 서버 설정 JSON

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "node",
      "args": ["server.js"],
      "env": { "API_KEY": "xxx" }
    },
    "remote-server": {
      "type": "http",
      "url": "https://mcp.example.com/v1",
      "headers": { "Authorization": "Bearer xxx" },
      "oauth": {
        "client_id": "...",
        "authorization_url": "...",
        "token_url": "..."
      }
    }
  }
}
```

---

## 4. Plugin 시스템

### 4.1 플러그인 매니페스트

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string
  source: string          // "name@marketplace"
  enabled?: boolean
  isBuiltin?: boolean
  commandsPaths?: string[]
  agentsPaths?: string[]
  skillsPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

### 4.2 설치 범위

```
user: 사용자 전역 (모든 프로젝트)
project: 프로젝트 (git 커밋 가능)
local: 로컬 (git-ignore)
```

**우선순위**: local > project > user (가장 구체적 우선)

### 4.3 MCP 서버 엔터프라이즈 제어

```typescript
// 화이트리스트 (정확히 하나만 지정)
AllowedMcpServerEntry = {
  serverName?: string,     // 이름 매칭
  serverCommand?: string[], // 정확한 명령어
  serverUrl?: string,       // URL 패턴 (와일드카드)
}

// 차단 목록 (동일 구조)
DeniedMcpServerEntry = { ... }
```

---

## 5. 모델 선택 로직

### 5.1 선택 우선순위 (5단계)

```
1. /model 명령어 (세션 중 오버라이드)
2. --model CLI 플래그
3. ANTHROPIC_MODEL 환경변수
4. settings.model 저장 설정
5. 구독 타입별 기본값
```

### 5.2 구독별 기본 모델

| 구독 | 기본 모델 | 컨텍스트 |
|------|----------|---------|
| ANT (내부) | Opus 4.6 [1m] | 1M |
| Max | Opus 4.6 [1m] | 1M |
| Team Premium | Opus 4.6 [1m] | 1M |
| Pro | Sonnet 4.6 | 200K |
| Enterprise | Sonnet 4.6 | 200K |
| Team Standard | Sonnet 4.6 | 200K |
| Free/PAYG | Sonnet 4.6 | 200K |

### 5.3 모델 별칭

| 별칭 | 해석 |
|------|------|
| `opus` | 현재 기본 Opus (4.6) |
| `sonnet` | 현재 기본 Sonnet (4.6) |
| `haiku` | 현재 기본 Haiku (4.5) |
| `best` | 최고 성능 모델 |
| `opusplan` | Sonnet (플랜 모드용) |

### 5.4 컨텍스트 윈도우 결정

```
1. CLAUDE_CODE_MAX_CONTEXT_TOKENS 환경변수 → 오버라이드
2. [1m] 태그 → 1,000,000
3. modelCapability max_input_tokens → 자동 감지
4. CONTEXT_1M_BETA_HEADER 베타 → 1M
5. 기본값 → 200,000
```

### 5.5 프로바이더별 모델 ID

| 모델 | 1P | Bedrock | Vertex |
|------|-----|---------|--------|
| Opus 4.6 | claude-opus-4-6 | us.anthropic.claude-opus-4-6-v1 | claude-opus-4-6 |
| Sonnet 4.6 | claude-sonnet-4-6 | us.anthropic.claude-sonnet-4-6 | claude-sonnet-4-6 |
| Haiku 4.5 | claude-haiku-4-5-20251001 | us.anthropic.claude-haiku-4-5-20251001-v1:0 | claude-haiku-4-5@20251001 |

### 5.6 Fast Mode

```
표준: $5/$25 per MTok (Opus 4.6)
Fast:  $30/$150 per MTok (Opus 4.6 Fast)
비율: 6배 가격
```

---

## 6. Server/Direct Connect (IDE 통합)

### 6.1 아키텍처

```
IDE (VS Code / JetBrains)
    │
    ▼
POST /sessions → { session_id, ws_url, work_dir }
    │
    ▼
WebSocket (NDJSON)
    ├─ IDE → Claude: user messages, permission responses, interrupts
    └─ Claude → IDE: assistant messages, permission requests, tool results
```

### 6.2 메시지 프로토콜

```
IDE → Server:
  - type: "user" (사용자 메시지)
  - type: "control_response" (권한 응답)
  - type: "control_request" subtype: "interrupt" (중단)

Server → IDE:
  - type: "assistant" (모델 응답)
  - type: "control_request" subtype: "can_use_tool" (권한 요청)
  - type: "result" (도구 결과)
```

### 6.3 SpawnMode

| 모드 | 설명 |
|------|------|
| `single-session` | 1 세션 후 서버 종료 |
| `worktree` | 각 세션마다 git worktree 격리 |
| `same-dir` | 모든 세션이 CWD 공유 |

---

## 7. 기타 시스템

### 7.1 Undercover Mode

```
활성화 조건: ANT 사용자 + 공개 리포지토리 (내부 화이트리스트 불일치)

금지사항:
- 내부 코드명 (Capybara, Fennec, Tengu)
- 미출시 모델 (opus-4-7, sonnet-4-8)
- 내부 리포/프로젝트명
- "Claude Code" 언급
- Co-Authored-By 라인
- AI임을 나타내는 힌트

구현: 시스템 프롬프트에 UNDERCOVER MODE 지시사항 주입
```

### 7.2 Speculation (추측 실행)

```
목적: 사용자가 다음 명령 입력 전 백그라운드로 추측 실행

제약:
- Write 도구 금지 (파일 변경 불가)
- Read-only 도구만 (Bash, Read, Glob, Grep, LSP)
- MAX_SPECULATION_TURNS = 20
- MAX_SPECULATION_MESSAGES = 100
- Overlay 파일시스템 사용 (임시)
```

### 7.3 Magic Docs (자동 문서화)

```markdown
# MAGIC DOC: API Reference
*Maintain this documentation from the conversation*

## Endpoints
... (백그라운드 에이전트가 자동 업데이트)
```

```
감지: /^#\s*MAGIC\s+DOC:\s*(.+)$/im
지시: 다음 줄 이탤릭 텍스트
동작: 파일 읽힐 때 백그라운드 forked subagent가 업데이트
```

### 7.4 Keybinding 시스템

```
주요 바인딩:
  Ctrl+C         → app:interrupt
  Ctrl+D         → app:exit
  Ctrl+R         → history:search
  Ctrl+Shift+F   → app:globalSearch
  Ctrl+Shift+P   → app:quickOpen
  Shift+Tab      → chat:cycleMode
  Meta+P         → chat:modelPicker
  Meta+O         → chat:fastMode
  Space          → voice:pushToTalk
  Ctrl+X Ctrl+E  → chat:externalEditor

컨텍스트: Global, Chat, Autocomplete, Confirmation, Settings, Tabs...
커스텀: ~/.claude/keybindings.json
```

### 7.5 부트스트래핑 순서 (10단계)

```
1. 설정 검증 및 활성화
2. 안전 환경변수 적용
3. CA 인증서 설정
4. Graceful shutdown
5. 1P 이벤트 로깅 + GrowthBook
6. OAuth 계정 정보
7. JetBrains IDE 감지
8. GitHub 리포 감지
9. Remote managed settings
10. mTLS/Proxy 설정
```

---

## 8. Beta 헤더 전체 목록

| 헤더 | 용도 |
|------|------|
| `claude-code-20250219` | Core |
| `interleaved-thinking-2025-05-14` | Thinking |
| `context-1m-2025-08-07` | 1M 컨텍스트 |
| `structured-outputs-2025-12-15` | JSON Schema 출력 |
| `web-search-2025-03-05` | 웹 검색 |
| `fast-mode-2026-02-01` | Fast Mode |
| `effort-2025-11-24` | Effort |
| `task-budgets-2026-03-13` | Task Budget |
| `afk-mode-2026-01-31` | AFK Mode (feature-gated) |
| `token-efficient-tools-2025-02-19` | 토큰 효율 도구 |
| `files-api-2025-04-14` | Files API |
| `prompt-caching-scope` | 캐시 범위 |
| `redact-thinking` | Thinking 수정 |

---

## 9. 전체 시스템 완결 맵

```
REPORT 0-5:  기존 분석 (4개 레포, 사건 개요)
REPORT 6:    25개 레포 전체 구조
REPORT 7:    코어 5영역 (루프, 프롬프트, 도구, compaction, 메모리)
REPORT 8:    보충 11영역 (hooks, telemetry, UI, OAuth, voice, recovery...)
REPORT 9:    최종 완결 (spec 정독, sandbox, MCP transport, plugin,
             모델선택, server, undercover, speculation, magic docs,
             keybindings, beta headers, buddy 상세)
```

### 전체 커버리지

| 영역 | 보고서 | 상태 |
|------|--------|------|
| 에이전트 루프 | R7 | ✅ 코드 레벨 |
| 시스템 프롬프트 | R7 | ✅ 코드 레벨 |
| 도구 42+ | R7 | ✅ 코드 레벨 |
| 권한 4모드 | R7 | ✅ 코드 레벨 |
| Bash 보안 300KB | R7 | ✅ 코드 레벨 |
| Compaction 4단계 | R7 | ✅ 코드 레벨 |
| 메모리 3중 | R7 | ✅ 코드 레벨 |
| MCP 기본 | R7 | ✅ 기본 |
| Bridge JWT | R7 | ✅ 코드 레벨 |
| Hooks | R8 | ✅ 코드 레벨 |
| Telemetry | R8 | ✅ 코드 레벨 |
| Feature Flags | R8 | ✅ 코드 레벨 |
| Remote Settings | R8 | ✅ 코드 레벨 |
| React/Ink UI | R8 | ✅ 코드 레벨 |
| OAuth PKCE | R8 | ✅ 코드 레벨 |
| Voice Mode | R8 | ✅ 코드 레벨 |
| Error Recovery | R8 | ✅ 코드 레벨 |
| Worktree | R8 | ✅ 코드 레벨 |
| Cron | R8 | ✅ 코드 레벨 |
| Session 영속화 | R8 | ✅ 코드 레벨 |
| API 4 프로바이더 | R8 | ✅ 코드 레벨 |
| **claurst/spec 990KB** | **R9** | ✅ **80+ 인사이트** |
| **Sandbox** | **R9** | ✅ **코드 레벨** |
| **MCP Transport 6종** | **R9** | ✅ **코드 레벨** |
| **Plugin 시스템** | **R9** | ✅ **코드 레벨** |
| **모델 선택 로직** | **R9** | ✅ **코드 레벨** |
| **Server/Direct Connect** | **R9** | ✅ **코드 레벨** |
| **Undercover Mode** | **R9** | ✅ **코드 레벨** |
| **Speculation** | **R9** | ✅ 구조 |
| **Magic Docs** | **R9** | ✅ 구조 |
| **Keybindings** | **R9** | ✅ **코드 레벨** |
| **Buddy 상세** | **R9** | ✅ **구현 상세** |
| **Beta 헤더 전체** | **R9** | ✅ **13개 나열** |
| **부트스트래핑 10단계** | **R9** | ✅ 순서 |

**빠진 것: 없음.** 모든 주요 서브시스템 분석 완료.

---

*최종 보고서 | 25개 레포, 800K+ LOC, 15개 spec 문서 정독 | 2026-04-19*
