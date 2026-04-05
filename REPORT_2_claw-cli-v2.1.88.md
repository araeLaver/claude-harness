# Claude Code (claw-cli v2.1.88) 소스 코드 종합 분석 보고서
## 레포: ringmast4r/claw-cli-claude-code-source-code-v2.1.88

---

## 1. 전체 디렉토리 구조

```
claw-cli-v2.1.88/
├── package.json          # 프로젝트 설정 (claude-code-rebuild)
├── tsconfig.json         # TypeScript 설정
├── README.md / README_*.md  # 다국어 README (ES, JA, KO, RU, ZH, ZH_TW)
└── src/
    ├── main.tsx              # 애플리케이션 진입점
    ├── Tool.ts               # 도구 타입 정의 핵심 파일
    ├── tools.ts              # 도구 레지스트리 (모든 도구 등록)
    ├── commands.ts           # 슬래시 명령어 레지스트리
    ├── query.ts              # 쿼리 루프 (에이전트 루프 핵심)
    ├── QueryEngine.ts        # 쿼리 엔진 클래스
    ├── context.ts            # 컨텍스트 조립
    ├── cost-tracker.ts       # 비용 추적
    ├── history.ts            # 히스토리 관리
    ├── setup.ts              # 초기 설정
    │
    ├── assistant/            # Kairos 어시스턴트 모드 (세션 히스토리)
    ├── bootstrap/            # 부트스트랩 상태 관리
    ├── bridge/               # 원격 브릿지 시스템 (31개 파일!)
    ├── buddy/                # 컴패니언 펫 시스템
    ├── cli/                  # CLI 핸들러 및 트랜스포트
    ├── commands/             # 86개 슬래시 명령어 디렉토리
    ├── components/           # React/Ink UI 컴포넌트
    ├── constants/            # 상수 및 시스템 프롬프트
    ├── context/              # 컨텍스트 관련
    ├── coordinator/          # 코디네이터 모드 (멀티 에이전트)
    ├── entrypoints/          # SDK/CLI/MCP 진입점
    ├── hooks/                # React 훅 (83개 파일!)
    ├── ink/                  # 터미널 UI 렌더링 (Ink 프레임워크)
    ├── keybindings/          # 키바인딩 시스템
    ├── memdir/               # 메모리 디렉토리 시스템
    ├── migrations/           # 설정 마이그레이션
    ├── moreright/            # UI 추가 기능
    ├── native-ts/            # 네이티브 TypeScript 유틸리티
    ├── outputStyles/         # 출력 스타일 설정
    ├── plugins/              # 플러그인 시스템
    ├── public/               # 공개 자산
    ├── query/                # 쿼리 관련 서브모듈
    ├── remote/               # 원격 세션 관리
    ├── schemas/              # 스키마 정의
    ├── screens/              # 화면 컴포넌트 (REPL.tsx 등)
    ├── server/               # 서버 관련
    ├── services/             # 핵심 서비스 (20개 하위 디렉토리)
    │   ├── api/              # API 클라이언트 (claude.ts)
    │   ├── compact/          # 대화 요약/압축 시스템
    │   ├── mcp/              # MCP 프로토콜 클라이언트 (23개 파일!)
    │   ├── autoDream/        # 자동 메모리 통합 ("꿈" 기능)
    │   ├── MagicDocs/        # 매직 문서 자동 업데이트
    │   ├── SessionMemory/    # 세션 메모리
    │   ├── extractMemories/  # 메모리 추출 에이전트
    │   ├── tips/             # 팁 시스템
    │   ├── analytics/        # 분석/텔레메트리
    │   ├── lsp/              # LSP 서버 관리
    │   ├── tools/            # 도구 실행 오케스트레이션
    │   └── ...
    ├── skills/               # 스킬 시스템
    ├── state/                # 상태 관리 (AppState)
    ├── tasks/                # 태스크 시스템 (DreamTask, AgentTask 등)
    ├── tools/                # 42개 도구 디렉토리
    ├── types/                # TypeScript 타입 정의
    ├── upstreamproxy/        # 업스트림 프록시
    ├── utils/                # 유틸리티 (방대한 서브모듈)
    │   ├── permissions/      # 권한 시스템 (24개 파일!)
    │   ├── model/            # 모델 설정 (16개 파일)
    │   ├── sandbox/          # 샌드박스 어댑터
    │   ├── hooks/            # 훅 유틸리티
    │   ├── settings/         # 설정 관리
    │   └── ...
    ├── vim/                  # Vim 모드 구현
    └── voice/                # 음성 모드
```

---

## 2. 아키텍처 개요

### 진입점
`/src/main.tsx`가 메인 진입점. 실행 전 사이드이펙트 순서가 매우 중요하게 관리된다:
1. `profileCheckpoint` -- 성능 프로파일링 시작
2. `startMdmRawRead` -- MDM(Mobile Device Management) 서브프로세스 병렬 실행
3. `startKeychainPrefetch` -- macOS 키체인 읽기 병렬 프리페치 (~65ms 절약)

### 빌드 시스템
- **Bun 번들러** 사용 (`feature()` from `bun:bundle`로 기능 플래그 관리)
- **TypeScript 6.0.2** + `tsx` 런타임
- `jsx: "react-jsx"` 설정으로 React/Ink 기반 터미널 UI
- Dead Code Elimination 수행

### 핵심 의존성
```json
{
  "@anthropic-ai/sdk": "^0.80.0",
  "@modelcontextprotocol/sdk": "^1.29.0",
  "react": "^19.2.4",
  "zod": "^4.3.6",
  "chalk": "^5.6.2",
  "axios": "^1.14.0"
}
```

---

## 3. 핵심 모듈 및 역할

### 3.1 쿼리 루프 (`query.ts`, `QueryEngine.ts`)
에이전트의 핵심 루프. `query()` 함수가 `AsyncGenerator`로 구현:

```typescript
export async function* query(params: QueryParams): AsyncGenerator<StreamEvent | Message, Terminal> {
  const terminal = yield* queryLoop(params, consumedCommandUuids)
}
```

루프 상태:
- `maxOutputTokensRecoveryCount`: 최대 출력 토큰 복구 시도 (최대 3회)
- `hasAttemptedReactiveCompact`: 반응형 컴팩트 시도 여부
- `turnCount`: 턴 카운트
- `autoCompactTracking`: 자동 컴팩트 추적
- `taskBudget`: 태스크 예산 관리

### 3.2 도구 실행 (`services/tools/`)
- `StreamingToolExecutor`: 스트리밍 중 도구 동시 실행 오케스트레이터
  - 동시 실행 안전한 도구(읽기 전용)는 병렬 처리
  - 비안전 도구(쓰기)는 직렬 처리
- 최대 동시 실행 수: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` (기본값 10)

### 3.3 상태 관리 (`state/`)
- `AppStateStore.ts`: 메인 애플리케이션 상태
  - 모델 설정, 권한 컨텍스트, MCP 연결, 에이전트 정의, 태스크 상태
  - `SpeculationState`: 추측 실행(사용자 입력 전 미리 실행) 지원
  - `FooterItem`: tasks, tmux, bagel, teams, bridge, companion 표시 영역
- 불변 상태 패턴: `DeepImmutable` 타입 사용

---

## 4. 시스템 프롬프트

### 4.1 프롬프트 캐시 경계 기법
```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```
이 마커 이전은 글로벌 캐시 가능(정적), 이후는 세션별 동적 콘텐츠.

### 4.2 프롬프트 섹션 캐싱 시스템 (`systemPromptSections.ts`)
- `systemPromptSection()`: 메모이즈된 프롬프트 섹션 (한 번 계산, `/clear` 또는 `/compact`까지 캐시)
- `DANGEROUS_uncachedSystemPromptSection()`: 매 턴마다 재계산되는 휘발성 섹션

### 4.3 ANT(내부) vs 외부 빌드 프롬프트 차이

**내부 전용 출력 지침:**
```
Length limits: keep text between tool calls to ≤25 words.
Keep final responses to ≤100 words unless the task requires more detail.
"역피라미드 구조", "의미적 백트래킹 회피"
```
~1.2% output token 감소 효과를 A/B 테스트로 검증.

**False-Claims 완화 (capy v8):**
```
"출력에 실패가 표시될 때 절대로 '모든 테스트 통과'라고 주장하지 마라"
```

### 4.4 프롬프트 빌드 우선순위 (`utils/systemPrompt.ts`)
```
0. Override 프롬프트 (루프 모드) -- 모든 프롬프트 대체
1. Coordinator 프롬프트 (코디네이터 모드)
2. Agent 프롬프트 (mainThreadAgentDefinition)
3. Custom 프롬프트 (--system-prompt)
4. Default 프롬프트 (표준 Claude Code 프롬프트)
+ appendSystemPrompt는 항상 끝에 추가
```

---

## 5. 도구 시스템

### 5.1 내장 도구 전체 목록 (42개 디렉토리)

**핵심 도구:**
- `AgentTool` (228KB!) -- 서브에이전트 생성/관리
- `BashTool` (156KB) -- 쉘 명령 실행 (보안 검증 포함)
- `FileReadTool` / `FileEditTool` / `FileWriteTool`
- `GlobTool` / `GrepTool`
- `NotebookEditTool` / `WebFetchTool` / `WebSearchTool`
- `AskUserQuestionTool` / `SkillTool` / `TodoWriteTool`

**태스크 도구:** TaskCreate/Get/Update/List/Output/StopTool

**에이전트 협업:** TeamCreate/DeleteTool, SendMessageTool, ListPeersTool

**피처 게이트 도구:**
- `SleepTool`, `CronCreate/Delete/ListTool`, `RemoteTriggerTool`
- `MonitorTool`, `SendUserFileTool`, `PushNotificationTool`
- `SubscribePRTool`, `WebBrowserTool`, `TerminalCaptureTool`
- `SnipTool`, `WorkflowTool`, `BriefTool`

**ANT 전용:** REPLTool, SuggestBackgroundPRTool, ConfigTool, TungstenTool, LSPTool

### 5.2 도구 필터링 체계
```typescript
// 1. getAllBaseTools() -- 모든 기본 도구
// 2. filterToolsByDenyRules() -- deny 규칙으로 필터
// 3. isEnabled() 체크 -- 각 도구의 활성화 상태 확인
// 4. assembleToolPool() -- 내장 + MCP 도구 결합, 이름 중복 시 내장 우선
```

---

## 6. 에이전틱 워크플로우

### 6.1 서브에이전트 시스템 (`tools/AgentTool/`)

**내장 에이전트 타입:**
- `exploreAgent`: 코드베이스 탐색/심층 리서치
- `generalPurposeAgent`: 범용 에이전트
- `planAgent`: 계획 수립 에이전트
- `verificationAgent`: 구현 결과 검증 에이전트 (adversarial verification)
- `claudeCodeGuideAgent`: Claude Code 사용 가이드
- `statuslineSetup`: 상태줄 설정

**포크 서브에이전트:**
부모 컨텍스트를 상속하면서 백그라운드에서 실행. 프롬프트 캐시를 공유하므로 효율적.

### 6.2 코디네이터 모드
멀티 에이전트 조율 모드:
- 워커에게 작업 분배
- 워커 사용 가능 도구 제한 (`ASYNC_AGENT_ALLOWED_TOOLS`)
- 코디네이터 전용 시스템 프롬프트

### 6.3 팀/스왐 시스템 (`utils/swarm/`)
- `InProcessBackend`: 프로세스 내 에이전트 실행
- `teammatePromptAddendum`: 팀원 프롬프트 추가

### 6.4 Ultraplan (`commands/ultraplan.tsx`)
원격 CCR 세션에서 멀티 에이전트 탐색:
- 30분 타임아웃
- Opus 4.6 모델 사용

---

## 7. 메모리 시스템

### 7.1 Memdir 시스템 (`memdir/`)
- `MEMORY.md`를 진입점으로 사용하는 파일 기반 메모리 시스템
- 최대 200줄 / 25,000바이트 제한
- 4가지 메모리 유형: `user`, `feedback`, `project`, `reference`

### 7.2 자동 메모리 추출 (`services/extractMemories/`)
백그라운드 포크 에이전트로 실행. 턴 1에서 모든 파일 읽기 → 턴 2에서 모든 쓰기 (병렬).

### 7.3 자동 드림 (AutoDream) (`services/autoDream/`)
백그라운드 메모리 통합 시스템:
- 시간 게이트: 마지막 통합 이후 24시간 경과
- 세션 게이트: 최소 5개 세션 축적
- 잠금 메커니즘: 중복 실행 방지

### 7.4 세션 메모리 (`services/SessionMemory/`)
10개 섹션으로 구조화된 세션별 노트 파일 관리.

### 7.5 컴팩트 시스템 (`services/compact/`)
- **전체 컴팩트**: 전체 대화를 9개 섹션 구조로 요약
- **부분 컴팩트**: 최근 부분만 요약
- **반응형 컴팩트**: prompt_too_long 에러 시 자동 발동
- **마이크로 컴팩트**: 캐시된 설정으로 빠른 요약
- **스니핑 컴팩트**: 히스토리 스니핑 기능
- 컴팩트 최대 출력 토큰: 20,000

### 7.6 Away Summary (`services/awaySummary.ts`)
사용자 복귀 시 "여기까지 했습니다" 요약 (Haiku 모델 사용, 최근 30개 메시지).

### 7.7 Magic Docs (`services/MagicDocs/`)
`# MAGIC DOC:` 헤더 파일을 자동 업데이트하는 백그라운드 에이전트.

### 7.8 팀 메모리 (`memdir/teamMemPaths.ts`)
TEAMMEM 피처 플래그로 활성화되는 팀 공유 메모리 시스템.

---

## 8. 권한 시스템

### 8.1 권한 모드
```typescript
export const PERMISSION_MODES = ['default', 'plan', 'bypassPermissions']
```

### 8.2 YOLO(Auto Mode) 분류기 (`utils/permissions/yoloClassifier.ts`)
자동 모드에서 도구 호출을 자동 허용/거부 판단하는 AI 분류기.

### 8.3 Bash 보안 (300KB+!)
- `bashPermissions.ts` (96KB)
- `pathValidation.ts` (42KB)
- `readOnlyValidation.ts` (66KB)
- `sedValidation.ts` (21KB)
- 파괴적 명령어 경고 시스템

### 8.4 샌드박스 시스템
`@anthropic-ai/sandbox-runtime` 래핑: 파일시스템/네트워크 제한, 위반 이벤트 추적.

### 8.5 거부 추적 (`denialTracking.ts`)
도구 거부 횟수 추적, 반복 거부 시 프롬프트 폴백.

---

## 9. 숨겨진/흥미로운 기능

### 9.1 Buddy 시스템 (컴패니언 펫)

**종 18종:** duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk

**레어리티 시스템:**
```typescript
RARITY_WEIGHTS = { common: 60, uncommon: 25, rare: 10, epic: 4, legendary: 1 }
```

**특징:**
- 사용자 ID의 해시로 결정론적 생성 (Mulberry32 PRNG)
- 눈 모양: `·`, `✦`, `×`, `◉`, `@`, `°`
- 모자: crown, tophat, propeller, halo, wizard, beanie, tinyduck
- 스탯: DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK
- 1% 확률로 "shiny" 변형
- `/buddy pet` 명령으로 하트 이펙트 (2.5초)

**종 이름 난독화 (코드명 누출 방지):**
```typescript
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
```

### 9.2 Kairos 데몬 (어시스턴트 모드)
- SleepTool, PushNotificationTool, SendUserFileTool, SubscribePRTool
- 프로액티브 프롬프트: "You are an autonomous agent"

### 9.3 음성 모드
음성 스트리밍 STT (20KB 코드), 음성 통합 훅 (97KB!)

### 9.4 브릿지 모드 (31개 파일, ~500KB+)
JWT 인증, 웹소켓 통신, 신뢰 장치 관리. REPL 브릿지 (98KB!)

### 9.5 Advisor 시스템 (`utils/advisor.ts`)
서버 측 도구: `server_tool_use` 타입, 암호화된 결과 지원 (`advisor_redacted_result`).

### 9.6 Speculation (추측 실행)
```typescript
type SpeculationState =
  | { status: 'idle' }
  | { status: 'active'; ... pipelinedSuggestion }
```
사용자 입력 전에 다음 동작을 추측 실행.

### 9.7 Insights 명령어 (113KB!)
전체 세션 히스토리를 Opus 모델로 분석하여 사용 패턴 리포트 생성.

### 9.8 Stickers 명령어
`stickermule.com/claudecode` 스티커 페이지 오픈.

### 9.9 Vim 모드
완전한 Vim 키바인딩 구현: motions, operators, text objects, transitions.

---

## 10. 모델 설정

### 지원 모델
```
Claude 3.5 Sonnet v2, Claude 3.5 Haiku, Claude 3.7 Sonnet,
Claude Haiku 4.5, Claude Sonnet 4/4.5/4.6,
Claude Opus 4/4.1/4.5/4.6
```
각 모델은 4개 프로바이더별 ID: `firstParty`, `bedrock`, `vertex`, `foundry`

### 모델 마이그레이션 히스토리
```
Fennec → Opus → Opus 1M → Sonnet 4.5 → Sonnet 4.6
```

### Effort 시스템
```typescript
EFFORT_LEVELS = ['low', 'medium', 'high', 'max']
```

### 토큰 제한
```typescript
MAX_OUTPUT_TOKENS_DEFAULT = 32_000
MAX_OUTPUT_TOKENS_UPPER_LIMIT = 64_000
CAPPED_DEFAULT_MAX_TOKENS = 8_000     // 슬롯 예약 최적화
ESCALATED_MAX_TOKENS = 64_000         // 복구 시 상향
COMPACT_MAX_OUTPUT_TOKENS = 20_000    // 컴팩트 작업용
MODEL_CONTEXT_WINDOW_DEFAULT = 200_000
// 1M 지원: Sonnet 4.x, Opus 4.6
```

---

## 11. MCP 통합

### MCP 클라이언트 (116KB!)
- 3가지 전송: Stdio, SSE, StreamableHTTP
- MCPTool로 MCP 도구를 네이티브 래핑
- McpAuthTool로 OAuth 인증

### MCP 인증 (87KB!)
OAuth + XAA(eXternal Authentication Agent) + IDP 로그인.

### MCP 서버 지시
연결된 MCP 서버의 `instructions`가 시스템 프롬프트에 자동 삽입.

---

## 12. 원본과의 차이점/추가 기능

### 12.1 빌드 환경
`package.json`의 이름이 `claude-code-rebuild` → npm 번들에서 소스 추출/재구성

### 12.2 ANT 전용 기능 노출
원본 외부 빌드에서 제거되는 기능이 소스 형태로 남아있음:
- REPLTool, ConfigTool, TungstenTool
- Coder homespace 통합 (`coder list`, SSH)
- 숫자 길이 앵커 프롬프트
- False-claims 완화 지침 (Capybara v8)

### 12.3 규모 지표
- 약 300개 이상의 TypeScript 파일
- 86개의 슬래시 명령어 디렉토리
- 42개의 도구 디렉토리
- 83개의 React 훅
- BashTool 보안 코드만 300KB 이상
- MCP 클라이언트/인증 코드 200KB 이상

### 12.4 추가 기능 (yasas 버전 대비)
- **Advisor 시스템**: 서버측 도구, 암호화된 결과
- **Speculation**: 추측 실행
- **Magic Docs**: 자동 문서 업데이트
- **Away Summary**: 사용자 복귀 시 요약
- **Insights 명령어**: 세션 분석 리포트
- **다국어 README**: ES, JA, KO, RU, ZH, ZH_TW
