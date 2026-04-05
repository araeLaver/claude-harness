# CLAURST 저장소 심층 분석 보고서
## 레포: Kuberwastaken/claurst (Rust 리라이트)

---

## 1. 전체 디렉토리 구조

```
claurst/
├── README.md                    # 프로젝트 소개 + Claude Code 유출 기술 분석 (32KB)
├── LICENSE.md                   # GPL-3.0 라이선스
├── public/                      # 정적 자산
│   ├── Rustle.png              # 마스코트 이미지
│   ├── screenshot.png          # 스크린샷
│   ├── claude-files.png        # 유출 관련 이미지
│   └── leak-tweet.png          # 유출 트윗 캡처
├── spec/                        # 행위 명세서 (15개 문서, ~990KB)
│   ├── INDEX.md                # 빠른 참조 인덱스
│   ├── 00_overview.md          # 마스터 아키텍처 개요
│   ├── 01_core_entry_query.md  # main.tsx, query.ts, QueryEngine.ts
│   ├── 02_commands.md          # 100+ 슬래시 커맨드 전체 명세
│   ├── 03_tools.md             # 40+ 도구 입력 스키마/권한/출력
│   ├── 04_components_core_messages.md # UI 컴포넌트 130개
│   ├── 05_components_agents_permissions_design.md # 에이전트/권한 다이얼로그
│   ├── 06_services_context_state.md   # 서비스 계층 95KB
│   ├── 07_hooks.md             # React 훅 104개
│   ├── 08_ink_terminal.md      # 커스텀 터미널 프레임워크
│   ├── 09_bridge_cli_remote.md # 브리지 프로토콜
│   ├── 10_utils.md             # 유틸리티 564개
│   ├── 11_special_systems.md   # Buddy, memdir, 키바인딩, 스킬
│   ├── 12_constants_types.md   # 상수/타입/OAuth/시스템 프롬프트
│   └── 13_rust_codebase.md     # Rust 리라이트 명세 (63KB)
└── src-rust/                    # Rust 구현 본체
    ├── Cargo.toml              # Workspace 매니페스트
    └── crates/
        ├── core/    (claurst-core)     # 공유 타입, 설정, 권한, 히스토리
        ├── api/     (claurst-api)      # Anthropic API 클라이언트 + SSE 스트리밍
        ├── tools/   (claurst-tools)    # 도구 구현체 33+개
        ├── query/   (claurst-query)    # 에이전틱 쿼리 루프, compact, cron
        ├── tui/     (claurst-tui)      # ratatui 터미널 UI
        ├── commands/ (claurst-commands) # 슬래시 커맨드 시스템
        ├── mcp/     (claurst-mcp)      # Model Context Protocol 클라이언트
        ├── bridge/  (claurst-bridge)   # claude.ai 웹 UI 브리지
        ├── cli/     (claurst)          # 바이너리 진입점
        ├── buddy/   (claurst-buddy)    # 타마고치 컴패니언 시스템
        └── plugins/ (claurst-plugins)  # 플러그인 런타임
```

---

## 2. 아키텍처 개요

### 진입점과 빌드 시스템

**진입점**: `src-rust/crates/cli/src/main.rs` (2,312줄)

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. --version, auth 서브커맨드 등 fast-path 처리
    // 2. clap으로 CLI 인자 파싱
    // 3. Settings 로드 + Config 구성
    // 4. OAuth 인증 / API 키 확인
    // 5. MCP 서버 연결
    // 6. 도구 레지스트리 구축
    // 7. 플러그인 로드
    // 8. cron 스케줄러 시작
    // 9. headless(--print) 또는 interactive(TUI) 모드 분기
}
```

**빌드 시스템**: Cargo workspace (`resolver = "2"`, edition 2021, version 0.0.6)

### Crate 의존성 흐름

```
cli ──→ query ──→ tools ──→ core
  │       │         ↗
  │      api  ──→ core
  │       │
  │     commands ──→ core
  │       │
  │      tui   ──→ core
  │       │
  │      mcp   ──→ core
  │       │
  │     bridge ──→ core
  │       │
  │     buddy  ──→ (독립)
  └──→ plugins ──→ core
```

### 핵심 의존 라이브러리

| 라이브러리 | 용도 |
|-----------|------|
| `tokio` (1.44, full) | async 런타임 |
| `reqwest` (0.12) | HTTP 클라이언트 (SSE 스트리밍 포함) |
| `ratatui` (0.29) + `crossterm` (0.28) | 터미널 UI (React/Ink 대체) |
| `clap` (4) | CLI 인자 파싱 |
| `serde` + `serde_json` | 직렬화 |
| `anyhow` + `thiserror` | 에러 처리 |
| `parking_lot` + `dashmap` | 동시성 제어 |
| `similar` | diff 알고리즘 (FileEditTool용) |
| `syntect` | 구문 강조 |
| `nix` (0.29) | Unix 프로세스/시그널 관리 |
| `tokio-tungstenite` | WebSocket (브리지 프로토콜) |

---

## 3. 핵심 모듈 상세 분석

### 3.1 claurst-core (3,280줄 + 38개 서브모듈)

전체 프로젝트의 공유 기반 레이어. 모든 다른 crate가 의존.

- **`error`**: `ClaudeError` enum (13가지 에러 타입). `is_retryable()`, `is_context_limit()` 헬퍼.
- **`types`**: `Role`, `ContentBlock`, `Message`, `UsageInfo`, `ToolDefinition` 등.
- **`config`**: `Config`, `Settings`, `PermissionMode`(Default/AcceptEdits/BypassPermissions/Plan). 계층적 설정 (Managed > Local > Project > Global).
- **`permissions`**: `PermissionHandler` trait, Auto/Interactive/Managed 핸들러. `PermissionRule` 패턴 매칭.
- **`bash_classifier`**: 셸 커맨드 보안 분류기. `BashRiskLevel` 5단계 (Safe/Low/Medium/High/Critical).
- **`claudemd`**: CLAUDE.md 계층적 메모리 파일 로딩.
- **`system_prompt`**: 모듈화된 시스템 프롬프트 조립. `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 마커. `OutputStyle` enum.
- **`token_budget`**: 토큰 예산 관리.
- **`feature_flags`**: GrowthBook 기반 feature flag 매니저.
- **`context_collapse`**: 컨텍스트 자동 축소 서비스.
- **`voice`**: 음성 입력 (Whisper API, `cpal` feature flag).
- **`analytics`**: `AtomicU64` 기반 lock-free 세션 메트릭.
- 기타: `git_utils`, `file_history`, `session_storage`, `crypto_utils`, `memdir`, `migrations`, `keybindings`, `lsp`

### 3.2 claurst-api (1,133줄 + 서브모듈)

Anthropic Messages API 클라이언트:

- SSE 스트리밍: `message_start`, `content_block_start/delta/stop`, `message_delta`, `message_stop`, `error`
- Rate-limit(429) / overloaded(529) 자동 재시도 (exponential backoff)
- **`cch.rs`**: CCH(Client-Computed Hash) 요청 서명 (`xxHash64` 기반)
- **`codex_adapter.rs`**: Codex(OpenAI) 호환 어댑터

### 3.3 claurst-tools (581줄 + 34개 도구 모듈)

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn permission_level(&self) -> PermissionLevel;
    fn input_schema(&self) -> serde_json::Value;
    async fn execute(&self, input: serde_json::Value, ctx: &ToolContext) -> ToolResult;
}
```

`PermissionLevel` 6단계: `None`, `ReadOnly`, `Write`, `Execute`, `Dangerous`, `Forbidden`.

**전체 도구 목록 (34+개)**:

| 도구 | 설명 |
|------|------|
| `BashTool` | 셸 명령 실행, persistent cwd/env, timeout, background task |
| `PowerShellTool` | Windows PowerShell |
| `FileReadTool` | 파일 읽기 (이미지, PDF 포함) |
| `FileEditTool` | exact string replacement |
| `FileWriteTool` | 파일 작성 |
| `GlobTool` | 파일 패턴 매칭 |
| `GrepTool` | 코드 검색 |
| `WebFetchTool` | URL 콘텐츠 |
| `WebSearchTool` | 웹 검색 |
| `NotebookEditTool` | Jupyter 노트북 |
| `AskUserQuestionTool` | 사용자 질문 |
| `EnterPlanModeTool` / `ExitPlanModeTool` | Plan 모드 |
| `BriefTool` | 파일 요약 |
| `SkillTool` | 스킬 실행 |
| `SleepTool` | async 대기 |
| `ToolSearchTool` | 도구 탐색 |
| `LspTool` | LSP 통신 |
| `TaskCreate/Get/List/Update/Output/StopTool` | 태스크 관리 |
| `SendMessageTool` | 에이전트 간 메시지 |
| `TeamCreateTool` / `TeamDeleteTool` | 에이전트 팀/스웜 |
| `CronCreate/Delete/ListTool` | cron 작업 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Git worktree |
| `ComputerUseTool` | 화면 캡처/클릭/키보드 |
| `ConfigTool` | 설정 수정 |
| `TodoWriteTool` | TODO 작성 (레거시) |
| `ListMcpResourcesTool` / `ReadMcpResourceTool` | MCP 리소스 |
| `McpAuthTool` | MCP 인증 |
| `ReplTool` | 인터랙티브 VM 셸 |
| `SyntheticOutputTool` | 구조화된 출력 |
| `RemoteTriggerTool` | 원격 에이전트 트리거 |

### 3.4 claurst-query (1,446줄 + 10개 서브모듈)

핵심 에이전틱 쿼리 루프:

```rust
pub async fn run_query_loop(
    client: &AnthropicClient,
    messages: &mut Vec<Message>,
    tools: &[Box<dyn Tool>],
    tool_ctx: &ToolContext,
    config: &QueryConfig,
    cost_tracker: Arc<CostTracker>,
    event_tx: Option<mpsc::UnboundedSender<QueryEvent>>,
    cancel: CancellationToken,
    post_sampling_hooks: Option<&Config>,
) -> QueryOutcome
```

**루프 동작**:
1. API 요청 구축 (system prompt + history + tools + token budget)
2. SSE 스트리밍 응답 수신
3. `tool_use` 블록 감지시 도구 실행
4. 도구 결과를 대화에 주입
5. auto-compact 필요시 자동 압축
6. 종료 조건 체크 (end_turn, max_turns, cancellation, budget exceeded)

**서브모듈**:
- **`agent_tool.rs`**: 하위 에이전트 생성. `isolation: "worktree"` + `run_in_background: true` 지원.
- **`compact.rs`**: 90% 임계값 auto-compact. `MicroCompact` (75%), circuit breaker (3회 실패시 비활성화).
- **`auto_dream.rs`**: 메모리 통합 데몬. 3-gate 트리거 (시간 24h + 세션 5개 + 잠금). 4단계 통합 (Orient→Gather→Consolidate→Prune).
- **`coordinator.rs`**: multi-worker 오케스트레이션. `ScratchpadGate` 시스템.
- **`session_memory.rs`**: 세션 종료 후 주요 사실을 CLAUDE.md에 자동 추출. 20+ 메시지 임계값.
- **`command_queue.rs`**: TUI와 쿼리 루프 간 비동기 커맨드 큐.
- **`cron_scheduler.rs`**: 백그라운드 cron 스케줄러.

`QueryEvent` enum으로 TUI에 실시간 이벤트 전파:
```rust
pub enum QueryEvent {
    Stream(StreamEvent),
    ToolStart { tool_name, tool_id, input_json },
    ToolEnd { tool_name, tool_id, result, is_error },
    TurnComplete { turn, stop_reason, usage },
    Status(String),
    Error(String),
    TokenWarning { state, pct_used },
}
```

### 3.5 claurst-tui (1,015줄 + 38개 서브모듈)

`ratatui` + `crossterm` 기반 완전한 터미널 UI:

- **`app.rs`**: App 상태 + 메인 이벤트 루프. 80개+ 슬래시 커맨드 typeahead.
- **`prompt_input.rs`**: Vim 모드, 히스토리, typeahead, 붙여넣기 처리.
- **`render.rs`**: 전체 ratatui 렌더링.
- **`dialogs.rs`**: 권한 다이얼로그.
- **`diff_viewer.rs`**: 2-pane diff 뷰어.
- **`virtual_list.rs`**: 가상 스크롤 리스트.
- **`model_picker.rs`**: 모델 선택 오버레이.
- **`session_browser.rs`** / **`session_branching.rs`**: 세션 탐색/분기.
- **`context_viz.rs`**: 컨텍스트 윈도우/rate-limit 시각화.
- **`kitty_image.rs`**: Kitty 프로토콜 인라인 이미지 렌더링.
- **`voice_capture.rs`**: Push-to-talk + Whisper 변환.
- **`agents_view.rs`**: 에이전트 관리 UI + coordinator 진행.
- **`stats_dialog.rs`**: 토큰/비용 차트.
- **`mcp_view.rs`**: MCP 서버 관리 UI.

### 3.6 claurst-bridge (1,713줄)

claude.ai 웹 UI와의 원격 제어 브리지:
- JWT 디코딩 (만료 확인용)
- `sk-ant-si-` 접두사 세션 토큰
- 디바이스 핑거프린트 (`SHA-256(hostname:user:homedir)`)
- long-polling (exponential backoff + cancellation)

### 3.7 claurst-mcp (2,112줄 + 서브모듈)

MCP 클라이언트:
- JSON-RPC 2.0 프리미티브
- MCP 핸드셰이크 (`initialize`, `initialized`)
- 도구 (`tools/list`, `tools/call`), 리소스 (`resources/list`, `resources/read`), 프롬프트 (`prompts/list`, `prompts/get`)
- 전송: stdio + HTTP/SSE
- 환경변수 확장 (`${VAR_NAME}`, `${VAR:-default}`)
- 재연결 (exponential backoff)

### 3.8 claurst-buddy (1,118줄)

타마고치/컴패니언 시스템:
- `Mulberry32` PRNG (유저 ID 결정론적 seed)
- **18종 Species**: Duck, Goose, Blob, Cat, Dragon, Octopus, Owl, Penguin, Turtle, Snail, Ghost, Axolotl, Capybara, Cactus, Robot, Rabbit, Mushroom, Chonk
- **5단계 Rarity**: Common/Uncommon/Rare/Epic/Legendary
- **6종 Eye**: `·`, `✦`, `×`, `◉`, `@`, `°`
- **8종 Hat**: crown, tophat, propeller, halo, wizard, beanie, tinyduck
- **5가지 스탯**: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK

### 3.9 claurst-commands (7,586줄)

50개+ 슬래시 커맨드: Help, Clear, Compact, Cost, Model, Config, Resume, Diff, Memory, Login, Mcp, Permissions, Plan, Tasks, Session, Thinking, Export, Skills, Rewind, Stats, Effort, Plugin, Theme, Voice, RemoteControl, Context, Vim 등.

`CommandResult` enum: `Message`, `UserMessage`, `ConfigChange`, `ClearConversation`, `SetMessages`, `ResumeSession`, `Exit`, `OpenRewindOverlay` 등 17개 변형.

### 3.10 claurst-plugins (679줄 + 6개 서브모듈)

플러그인 런타임:
- 플러그인 탐색, 매니페스트 파싱
- 후크 등록 (`PreToolUse`, `PostToolUse`)
- capability 기반 접근 제어
- `marketplace.rs`: 커뮤니티 플러그인 마켓플레이스

---

## 4. 원본 TypeScript vs Rust 매핑

### 완전히 포팅된 영역

| TypeScript 원본 | Rust 구현 | 비고 |
|----------------|-----------|------|
| `main.tsx` (4,683줄) | `cli/src/main.rs` (2,312줄) | 진입점 |
| `query.ts` + `QueryEngine.ts` (115KB) | `query/src/lib.rs` (1,446줄) | 쿼리 루프 |
| `Tool.ts` (30KB) | `tools/src/lib.rs` (581줄) | 도구 프레임워크 |
| `tools/` (184 파일) | `tools/src/` (34 모듈) | 33+ 도구 |
| `commands/` (207 파일) | `commands/src/lib.rs` (7,586줄) | 50+ 커맨드 |
| `services/api/claude.ts` | `api/src/lib.rs` | SSE 클라이언트 |
| `ink/` (96 파일) | `tui/src/` (38 모듈) | TUI (React/Ink → ratatui) |
| `bridge/` (31 파일) | `bridge/src/lib.rs` | 원격 브리지 |
| `buddy/` (6 파일) | `buddy/src/lib.rs` | 타마고치 |
| `services/autoDream/` | `query/src/auto_dream.rs` | 메모리 통합 |
| `coordinator/` | `query/src/coordinator.rs` | multi-agent |
| `plugins/` | `plugins/src/` (6 모듈) | 플러그인 |

### 아키텍처적 차이점

1. **UI**: React/Ink (커스텀 Yoga flexbox) → `ratatui` + `crossterm`
2. **동시성**: Node.js 이벤트 루프 → `tokio` async + `DashMap`/`parking_lot`/`AtomicU64` lock-free
3. **타입 시스템**: Zod 스키마 → `serde` + `schemars` JSON Schema
4. **번들링**: Bun bundler DCE → Cargo feature flags
5. **모듈화**: 1,902개 파일 → 11개 crate, ~100개 .rs 파일

---

## 5. 에이전틱 루프 상세

### TypeScript vs Rust 쿼리 루프 비교

| 측면 | TypeScript | Rust |
|------|-----------|------|
| 런타임 | Node.js 이벤트 루프 | tokio async |
| 이벤트 통신 | React state + hooks | `mpsc::UnboundedSender<QueryEvent>` |
| 도구 실행 | Promise chain | `tokio::spawn` + `await` |
| 취소 | AbortController | `CancellationToken` (tokio_util) |
| 상태 관리 | React context + refs | `Arc<Mutex<T>>` / `DashMap` |
| 컴팩션 | 별도 서비스 파일 | 인라인 모듈 `compact.rs` |

---

## 6. 성능 및 안전 보장

### Rust 특화 최적화

1. **Lock-free 카운터**: `SessionMetrics`의 `AtomicU64` (mutex 경합 없음)
2. **`parking_lot::Mutex`**: poisoning 없음, 더 작은 크기
3. **`DashMap`**: 세그먼트 기반 concurrent hashmap
4. **Zero-copy 파싱**: SSE 스트리밍 문자열 복사 최소화
5. **`once_cell::Lazy`**: 전역 상태 지연 초기화

### 안전 보장

1. **BashTool**: 5단계 위험도 분류 (Critical → Forbidden 차단), Root/sudo 차단
2. **권한**: 4단계 PermissionMode + 6단계 PermissionLevel
3. **경로 탐색 방지**: `ToolContext::resolve_path()` 상대 경로 해석
4. **CCH 서명**: xxHash64 API 무결성 확인

---

## 7. 행위 명세 문서 (`spec/`) 상세

15개 문서, 총 ~990KB:

| 문서 | 내용 | 크기 |
|------|------|------|
| `INDEX.md` | 빠른 참조 인덱스 | - |
| `00_overview.md` | 마스터 아키텍처 | - |
| `01_core_entry_query.md` | main.tsx, query.ts, QueryEngine.ts | - |
| `02_commands.md` | 100+ 슬래시 커맨드 | - |
| `03_tools.md` | 40+ 도구 스키마/권한/출력 | - |
| `04_components_core_messages.md` | UI 컴포넌트 130개 | - |
| `05_components_agents_permissions_design.md` | 에이전트/권한 | - |
| `06_services_context_state.md` | 서비스 계층 | 95KB |
| `07_hooks.md` | React 훅 104개 | - |
| `08_ink_terminal.md` | 커스텀 터미널 프레임워크 | - |
| `09_bridge_cli_remote.md` | 브리지 프로토콜 | - |
| `10_utils.md` | 유틸리티 564개 | - |
| `11_special_systems.md` | Buddy, memdir, 키바인딩 | - |
| `12_constants_types.md` | 상수/타입/OAuth/시스템 프롬프트 | - |
| `13_rust_codebase.md` | Rust 리라이트 명세 | 63KB |

원본 TypeScript 핵심 지표 (INDEX.md):
- TypeScript/TSX 파일: ~1,902개
- 총 코드 줄수: ~800K+
- 슬래시 커맨드: 100+
- 도구: 40+
- React 훅: 104개
- React 컴포넌트: 389 파일
- 서비스: 130 파일
- 유틸리티: ~564 파일
- Ink 터미널 프레임워크: 96 파일
- 브리지 프로토콜: 31 파일

---

## 8. README에 기록된 발견된 주요 내부 시스템

| 시스템 | 설명 |
|--------|------|
| **BUDDY** | 타마고치 컴패니언 (결정론적 가챠 시스템) |
| **KAIROS** | always-on 프로액티브 어시스턴트 |
| **ULTRAPLAN** | 30분 원격 계획 세션 |
| **Dream System** | 메모리 통합 데몬 |
| **Undercover Mode** | Anthropic 직원의 오픈소스 기여시 AI 정체 숨김 |
| **Coordinator Mode** | multi-agent 오케스트레이션 |
| **Penguin Mode** | Fast Mode의 내부 코드명 |

---

## 9. 완성도 평가

README에서 **"100% Coverage complete from original source"** 주장.

### 완전히 구현된 영역
- 핵심 쿼리 루프 (auto-compact, token budget, fallback model)
- 33+ 도구 전체
- 50+ 슬래시 커맨드
- 완전한 TUI (Vim, 테마, 세션 분기, diff 뷰어, 이미지 렌더링)
- MCP 클라이언트 (stdio + HTTP/SSE)
- 브리지 프로토콜
- 플러그인 시스템 (디스커버리, 매니페스트, 후크, 마켓플레이스)
- 버디 시스템
- 음성 입력
- OAuth 인증
- Auto-dream 메모리 통합
- Coordinator 모드
- 세션 저장/복원
- Cron 스케줄러
- Git worktree 격리

### Rust 전용 추가사항
- `cch.rs`: xxHash64 클라이언트 서명
- `codex_adapter.rs`: OpenAI Codex 호환
- `parity_smoke.rs`: 원본 동작 일치 검증 테스트
- 텔레메트리 완전 제거 (No Tracking)
- 실험적 기능 기본 활성화

### 2단계 접근법
1. AI 에이전트가 원본 소스를 분석하여 행위 명세 생성 (`spec/`)
2. 별도 AI 에이전트가 명세만으로 Rust 구현

11개 crate, 100+ .rs 파일, 수만 줄의 코드로 TypeScript 원본의 핵심 기능을 거의 완전히 재현. GPL-3.0 라이선스로 공개.
