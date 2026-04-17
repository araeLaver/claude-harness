# Claude Code Harness 전체 엔지니어링 상세분석 보고서
## 25개 레포 완전 분석 | 2026-04-15

---

## 1. 전체 레포 지도

### 1.1 레포 분류표

| # | 레포 | 유형 | 언어 | 파일수 | 완성도 | 핵심 가치 |
|---|------|------|------|--------|--------|----------|
| 1 | `claw-cli-v2.1.88` | 원본 TS 소스 재구성 | TS/TSX | 1,884 | 95%+ | **가장 깔끔한 원본** |
| 2 | `exhen-claude-code-2.1.88` | 공식 npm 배포본 | TS→JS | 1,955 | 100% | **공식 배포 바이너리** |
| 3 | `hyper-sourcemap` | sourcemap 추출 | TS/TSX | 1,885 | 95%+ | **가장 완전한 추출** |
| 4 | `pm-shawn-source` | sourcemap 추출+분석 | TS/TSX | 1,885 | 95%+ | **아키텍처 분석 포함** |
| 5 | `claude-code-yasas` | 원본 유출 + 에셋 | TS/TSX | 1,906 | 95%+ | **브랜딩 자산 포함** |
| 6 | `claude-code-analysis` | 소스 아카이브 | TS/TSX | 1,908 | 95%+ | **zip 아카이브 포함** |
| 7 | `collection-claude-code` | 멀티파트 컬렉션 | TS/PY/JS | 4,012 | 95%+ | **다국어 분석 문서** |
| 8 | `leeyeel-sourcemap` | 최소 스텁 버전 | TS | 212 | 15% | **권한/도구 시그니처 참조** |
| 9 | `claurst` | Rust 완전 리라이트 | Rust | 155 | 85% | **990KB 행위 명세** |
| 10 | `claw-code` | Python 포트 | Python | 138 | 60% | **Discord 멀티에이전트** |
| 11 | `claw-code-agent` | Python 에이전트 | Python | 109 | 70% | **제로 의존성, Qwen3 지원** |
| 12 | `open-claude-code` | JS 리라이트 (v0.2+v2) | JS | 85 | 40% | **1,581 테스트 통과** |
| 13 | `deep-dive-claude-code` | 교육 플랫폼 | TS/MD | 134 | N/A | **13챕터 인터랙티브 학습** |
| 14 | `everything-claude-code` | 프로덕션 프레임워크 | MD/JS/SH | 1,510 | N/A | **38+ 에이전트, 50K★** |
| 15 | `haseeb-analysis-gist` | 전문 분석 | MD | 1 | N/A | **"하네스가 전부다"** |
| 16 | `claudecode-leak-autopsy` | 사건 조사 | MD | 7 | N/A | **보안 자문, 타임라인** |
| 17 | `sanbuphy-source` | 4개국어 분석 | MD | 20+ | N/A | **숨겨진 기능/텔레메트리** |
| 18 | `awesome-cli-coding-agents` | 생태계 인덱스 | MD | 2 | N/A | **80+ CLI 에이전트 목록** |
| 19 | `learn-claude-code` | 학습 자료 | MD | - | N/A | 교육용 |
| 20 | `codeaashu-claude-code` | 커뮤니티 포트 | - | - | N/A | 커뮤니티 |
| 21 | `tanbiralam-claude-code` | 커뮤니티 포트 | - | - | N/A | 커뮤니티 |
| 22 | `qnexai-claude-code` | 커뮤니티 포트 | - | - | N/A | 커뮤니티 |
| 23 | `ultraworkers-claw-code` | 커뮤니티 포트 | - | - | N/A | 커뮤니티 |
| 24 | `claude-code-harness-chachamaru` | 분석 | - | - | N/A | 커뮤니티 |
| 25 | `.claude/` | 설정 디렉토리 | - | - | N/A | 프로젝트 설정 |

### 1.2 추천 참조 순서

```
엔지니어링 이해:  claw-cli-v2.1.88 (원본) → claurst/spec/ (행위 명세)
아키텍처 분석:    haseeb-analysis-gist → pm-shawn-source (아키텍처 MD)
숨겨진 기능:      sanbuphy-source → claudecode-leak-autopsy
실용 활용:        everything-claude-code (에이전트/스킬 프레임워크)
학습:             deep-dive-claude-code (인터랙티브 13챕터)
생태계:           awesome-cli-coding-agents (80+ 도구 인덱스)
```

---

## 2. 코어 아키텍처 심층 분석

### 2.1 부트스트랩 시퀀스 (6단계)

```
┌──────────────────────────────────────────────────────────┐
│ 1. CLI Parse (main.tsx - 785KB)                          │
│    ├─ Commander.js 인수 파싱                              │
│    ├─ 환경 감지 (OS, Shell, Git, Node)                    │
│    └─ Feature Flag 로딩 (Bun DCE + GrowthBook)           │
├──────────────────────────────────────────────────────────┤
│ 2. Parallel I/O Prewarming                               │
│    ├─ CLAUDE.md 4계층 동시 로딩                           │
│    ├─ OAuth 토큰 검증                                     │
│    ├─ MCP 서버 연결 (stdio transport)                     │
│    ├─ Memdir 인덱스 로딩                                  │
│    └─ Git 상태 스냅샷                                     │
├──────────────────────────────────────────────────────────┤
│ 3. State Bootstrap (AppState.tsx - 23KB)                 │
│    ├─ 80+ 필드 초기화                                     │
│    ├─ 세션 ID 생성                                       │
│    ├─ 비용 추적기 초기화                                  │
│    └─ 권한 모드 결정                                      │
├──────────────────────────────────────────────────────────┤
│ 4. System Prompt Assembly (15개 composable 함수)         │
│    ├─ [정적] 캐시 가능 - Blake2b 해시                     │
│    ├─   Intro + System + Tasks + Actions + Tools         │
│    ├─   Tone + Output Efficiency                         │
│    ├─ ── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ──           │
│    ├─ [동적] 세션별                                       │
│    ├─   Session Guidance + CLAUDE.md                     │
│    ├─   환경 + MCP + Memdir                              │
│    └─   출력 스타일 + FRC                                │
├──────────────────────────────────────────────────────────┤
│ 5. React/Ink Mount                                       │
│    ├─ REPL.tsx (875KB) 마운트                             │
│    ├─ 커스텀 Ink reconciler (96개 파일)                    │
│    ├─ Yoga 레이아웃 엔진 (WASM)                          │
│    └─ 키바인딩 시스템 초기화                              │
├──────────────────────────────────────────────────────────┤
│ 6. Query Loop Enter                                      │
│    ├─ query.ts (67KB) AsyncGenerator 시작                │
│    ├─ QueryEngine.ts (46KB) SDK/REPL 추상화              │
│    └─ SSE 스트리밍 연결 대기                              │
└──────────────────────────────────────────────────────────┘
```

**핵심 포인트**: `main.tsx`가 785KB인 이유 — CLI 파싱, 환경 감지, OAuth, 세션 복원, 에러 핸들링, feature flag 분기가 **모두 한 파일**에 있음. 이는 Bun의 번들링 특성과 빠른 콜드 스타트를 위한 의도적 설계.

### 2.2 에이전트 루프 상세 흐름

```
사용자 입력
    │
    ▼
┌─────────────┐
│ query()     │  ← AsyncGenerator (yield로 이벤트 스트림)
│ 67KB        │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────┐
│ 시스템 프롬프트 조립              │
│ ├─ 정적 캐시 (3K tokens)        │  ← Blake2b 해시로 캐시 키
│ ├─ 동적 컨텍스트                 │
│ ├─ CLAUDE.md (4계층 병합)        │
│ ├─ Git Status 스냅샷             │
│ ├─ MCP 서버 도구 목록            │
│ └─ Memdir 메모리                 │
└──────────┬──────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│ API 호출 (SSE 스트리밍)          │
│ ├─ Provider: 1P / Bedrock /     │
│ │   Vertex / Foundry            │
│ ├─ Model: Opus/Sonnet/Haiku     │
│ ├─ Thinking: adaptive/enabled   │
│ └─ Effort: low~max              │
└──────────┬──────────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
[text_delta]  [tool_use]
     │           │
     │     ┌─────┴─────────────────┐
     │     │ 도구 실행 파이프라인    │
     │     │ ┌─ 권한 체크           │
     │     │ │  ├─ Mode check      │
     │     │ │  ├─ Hook eval       │
     │     │ │  ├─ Rule match      │
     │     │ │  └─ User prompt     │
     │     │ ├─ 입력 검증 (Zod)    │
     │     │ ├─ 실행              │
     │     │ ├─ 결과 변환          │
     │     │ └─ 거부시 → 에러로    │
     │     │   래핑 후 모델에 전달  │
     │     └───────────┬───────────┘
     │                 │
     │     ┌───────────┘
     ▼     ▼
┌─────────────────────────────────┐
│ 토큰 예산 체크                   │
│ ├─ 현재 사용량 계산              │
│ ├─ Compaction 필요성 판단        │
│ │  ├─ Proactive: 한계 접근시     │
│ │  ├─ Reactive: API 거부시       │
│ │  ├─ Snip: SDK 모드             │
│ │  └─ Collapse: marble_origami   │
│ └─ 종료 조건 체크                │
│    ├─ end_turn                   │
│    ├─ max_tokens 도달            │
│    ├─ +500K TOKEN_BUDGET         │
│    └─ 사용자 인터럽트            │
└──────────┬──────────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
 [계속 루프]   [종료 → 결과 반환]
```

### 2.3 시스템 프롬프트 캐시 최적화 (중요!)

이것은 Claude Code의 **가장 영리한 최적화** 중 하나:

```typescript
// 15개 composable 함수로 시스템 프롬프트 조립
const staticPart = [
  getIntroPrompt(),           // "You are Claude Code..."
  getSystemRules(),            // 도구/태그/권한
  getDoingTasksGuidance(),     // 코딩 스타일
  getActionRules(),            // 되돌릴 수 없는 작업
  getToolUsageRules(),         // 전용 도구 우선
  getToneAndStyle(),           // 이모지 금지
  getOutputEfficiency(),       // 간결함
].join('\n');

// Blake2b 해시로 캐시 키 생성 → 전역 공유
const cacheKey = blake2b(staticPart);

// 동적 부분은 세션마다 다름
const dynamicPart = [
  getSessionGuidance(),
  getClaudeMdMemory(),
  getEnvironmentInfo(),        // Git, OS, 모델
  getMcpInstructions(),
  getOutputStyle(),
  getFunctionResultClearing(),
].join('\n');

// __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__로 구분
// → Anthropic API의 prompt caching이 정적 부분만 캐시
// → 비용 절감: 정적 3K tokens × 수백만 요청 = 엄청난 절감
```

**왜 중요한가**: Anthropic API는 시스템 프롬프트를 prefix 기반으로 캐시. 정적/동적 경계를 명시적으로 분리함으로써, 모든 사용자가 같은 정적 캐시를 공유. 이는 **서버 비용 절감**과 **첫 토큰 레이턴시 감소** 모두에 기여.

---

## 3. 도구 시스템 심층 분석

### 3.1 도구 아키텍처

```
Tool.ts (28KB) - 기본 클래스
├── name: string
├── description: string
├── inputSchema: Zod schema
├── isReadOnly(): boolean          ← 읽기 전용이면 권한 체크 스킵
├── isEnabled(context): boolean    ← feature flag로 동적 활성화
├── validateInput(input): Result   ← Zod 검증
├── prompt(): string               ← 도구별 사용 가이드 (시스템 프롬프트에 삽입)
├── call(input, context): Result   ← 실제 실행
└── needsPermission(): boolean     ← 권한 필요 여부
```

### 3.2 도구별 상세 사양 (42+)

#### 파일 조작 도구

| 도구 | 크기 | 핵심 로직 |
|------|------|----------|
| **BashTool** | 96KB+ | 5단계 위험 분류, 샌드박스, 타임아웃(120s/600s), 인젝션 감지 |
| **FileReadTool** | ~15KB | 2000줄 기본, 이미지/PDF/노트북 지원, 멀티모달 |
| **FileEditTool** | ~20KB | exact string replacement, `old_string→new_string`, `replace_all` 옵션 |
| **FileWriteTool** | ~10KB | 새 파일 생성 전용, 기존 파일은 Read 선행 필수 |
| **GlobTool** | ~8KB | 수정시간 정렬, fast-glob 사용 |
| **GrepTool** | ~12KB | ripgrep 기반, `content/files_with_matches/count` 3모드 |

#### BashTool 보안 상세 (가장 복잡한 도구)

```
명령어 입력
    │
    ▼
┌─────────────────────────────────┐
│ 1. bashPermissions.ts (96KB)    │
│    ├─ 커맨드 파싱               │
│    ├─ 파이프/체인 분석          │
│    ├─ 위험도 5단계 분류:        │
│    │  Safe: echo, pwd, ls       │
│    │  Low: git status, npm list │
│    │  Medium: npm install       │
│    │  High: rm, git push        │
│    │  Critical: rm -rf /        │
│    └─ Forbidden → 즉시 차단     │
├─────────────────────────────────┤
│ 2. pathValidation.ts (42KB)     │
│    ├─ 5계층 경로 검증           │
│    ├─ 심볼릭 링크 해석          │
│    ├─ 홈 디렉토리 보호          │
│    └─ .env, credentials 감지    │
├─────────────────────────────────┤
│ 3. readOnlyValidation.ts (66KB) │
│    ├─ 읽기 전용 명령 허용 목록  │
│    ├─ 사이드이펙트 감지         │
│    └─ 출력 리다이렉션 차단      │
├─────────────────────────────────┤
│ 4. injectionCheck.ts            │
│    ├─ 환경변수 인젝션           │
│    ├─ 서브쉘 이스케이프          │
│    └─ 파이프 체인 위험도        │
├─────────────────────────────────┤
│ 5. sandbox.ts                   │
│    ├─ macOS: sandbox-exec       │
│    ├─ Linux: 네임스페이스       │
│    └─ 네트워크/파일시스템 격리  │
└─────────────────────────────────┘
    → 총 300KB+ 보안 코드!
```

#### AgentTool (228KB - 가장 큰 단일 도구)

```
AgentTool 아키텍처:
├─ 서브에이전트 생성 (Fork 방식)
│  ├─ subagent_type: general/Explore/Plan/claude-code-guide 등
│  ├─ model: sonnet/opus/haiku 오버라이드 가능
│  ├─ isolation: "worktree" → git worktree에서 격리 실행
│  └─ run_in_background: true → 백그라운드 실행
│
├─ 에이전트 간 통신
│  ├─ SendMessageTool → 기존 에이전트에 메시지 전송
│  ├─ TeamCreate/DeleteTool → 팀 관리
│  └─ 계보 추적 (lineage tracking)
│
├─ 컨텍스트 상속
│  ├─ 부모의 CLAUDE.md 상속
│  ├─ 도구 목록 제한 (type별)
│  └─ 독립 대화 히스토리
│
└─ 완료시 → 결과를 부모에게 반환
   (사용자에게는 보이지 않음 → 부모가 요약 필요)
```

#### 웹 도구

| 도구 | 기능 | 특이사항 |
|------|------|---------|
| **WebFetchTool** | URL→Markdown 변환 | Jina AI 프록시, 30초 타임아웃 |
| **WebSearchTool** | 웹 검색 | Brave Search API 사용 |
| **WebBrowserTool** | 헤드리스 브라우저 | Feature flag 뒤 (미출시) |

#### 태스크/플랜 도구

```
TaskCreate → TaskGet → TaskUpdate → TaskList → TaskOutput → TaskStop
    │
    └─ 각 태스크는 독립 에이전트로 실행 가능
       ├─ 의존성 추적
       ├─ 진행률 보고
       └─ 백그라운드 실행 지원

EnterPlanMode → 계획 수립 모드 (편집 불가)
ExitPlanMode  → 계획 확정 → 실행 모드
```

#### KAIROS 전용 도구 (미출시)

| 도구 | 기능 |
|------|------|
| **SleepTool** | 에이전트 일시정지 후 재개 |
| **PushNotificationTool** | 사용자에게 푸시 알림 전송 |
| **SendUserFileTool** | 파일을 사용자에게 전달 |
| **SubscribePRTool** | PR 이벤트 구독/모니터링 |

### 3.3 ToolSearchTool (Deferred Tools)

```
문제: 40+ 도구의 스키마를 모두 시스템 프롬프트에 넣으면 토큰 낭비
해결: Deferred Tools 패턴

1. 시스템 프롬프트에는 도구 이름만 나열
2. 모델이 ToolSearchTool 호출 → 필요한 도구 스키마 동적 로딩
3. 로딩된 스키마는 이후 호출 가능

쿼리 형식:
- "select:Read,Edit,Grep"  → 정확한 이름으로 로딩
- "notebook jupyter"        → 키워드 검색
- "+slack send"             → 이름 필수 + 키워드 랭킹
```

---

## 4. 권한 시스템 심층 분석

### 4.1 권한 모드 계층

```
┌─────────────────────────────────────────┐
│  bypassPermissions (YOLO 모드)          │ ← 모든 도구 자동 승인
├─────────────────────────────────────────┤
│  acceptEdits                            │ ← 파일 편집 자동 승인
├─────────────────────────────────────────┤
│  plan                                   │ ← 읽기만 자동 승인
├─────────────────────────────────────────┤
│  default                                │ ← 모든 것 확인 요청
└─────────────────────────────────────────┘
```

### 4.2 권한 파이프라인 상세

```
도구 호출 요청
    │
    ▼
[1] Mode Check
    ├─ bypassPermissions? → 즉시 승인
    ├─ isReadOnly()? → plan 이상이면 승인
    └─ acceptEdits? → 파일 편집이면 승인
    │
    ▼
[2] Hook Evaluation (PreToolUse)
    ├─ settings.json의 hooks 설정 확인
    ├─ 쉘 명령 실행 → exit code로 판단
    │  ├─ 0: 승인
    │  ├─ 2: 거부 (이유 포함)
    │  └─ 기타: 에러
    └─ 여러 훅 체인 가능
    │
    ▼
[3] Rule Matching
    ├─ permissions.ts (13KB) 규칙 엔진
    ├─ 패턴 매칭 (glob, regex)
    ├─ allow/deny 규칙 우선순위
    └─ CLAUDE.md에 정의 가능
    │
    ▼
[4] User Prompt (마지막 수단)
    ├─ 터미널에 승인/거부 묻기
    ├─ "Always allow" 옵션
    └─ 거부 → tool_result error로 래핑
         → 모델에게 전달 → 접근 방식 자동 변경
```

### 4.3 거부 피드백 루프 (핵심 설계)

```typescript
// 사용자가 도구 호출 거부시
const denialResult = {
  type: "tool_result",
  tool_use_id: toolCall.id,
  is_error: true,
  content: "User denied permission for this operation. " +
           "Consider an alternative approach."
};
// → 이 에러가 다음 API 호출에 포함
// → 모델이 대안을 찾아 재시도
// → "rm -rf" 거부 → "git clean -fd" 시도 → 그것도 거부 → 사용자에게 직접 요청
```

**왜 중요한가**: 대부분의 에이전트는 거부시 멈춤. Claude Code는 거부를 **학습 신호**로 사용하여 자동으로 더 안전한 대안을 탐색.

---

## 5. Compaction 시스템 심층 분석

### 5.1 4단계 전략 상세

```
┌────────────────────────────────────────────────────────────┐
│ Stage 1: PROACTIVE COMPACTION                              │
│ ─────────────────────────────                              │
│ 트리거: 매 턴마다 토큰 사용량 모니터링                       │
│ 조건: context_used / context_limit > threshold (보통 80%)   │
│ 동작:                                                      │
│   1. 대화 히스토리를 "핵심 결정 + 미해결 태스크" 중심 요약   │
│   2. tool_result 중 큰 것부터 요약/제거                     │
│   3. 사용자에게 투명 (알림 없음)                            │
│   4. 원본 보존 (복원 가능)                                  │
│ 장점: 가장 자연스러운 경험                                  │
├────────────────────────────────────────────────────────────┤
│ Stage 2: REACTIVE COMPACTION                               │
│ ─────────────────────────────                              │
│ 트리거: API가 `prompt_too_long` 에러 반환                   │
│ 동작:                                                      │
│   1. Proactive보다 더 공격적인 요약                         │
│   2. 이전 tool_result를 placeholder로 교체                  │
│   3. 재시도 (최대 3회)                                     │
│ 왜 존재하는가: Proactive가 토큰 계산을 잘못할 수 있음       │
├────────────────────────────────────────────────────────────┤
│ Stage 3: SNIP (SDK/headless 모드)                          │
│ ────────────────────────────────                           │
│ 트리거: SDK 모드 또는 headless 모드                         │
│ 동작:                                                      │
│   1. 단순 잘라내기 (요약 안 함)                             │
│   2. 오래된 메시지부터 제거                                 │
│   3. 빠르지만 정보 손실                                    │
│ 왜 존재하는가: SDK는 요약의 추가 API 호출 비용을 감당 불가  │
├────────────────────────────────────────────────────────────┤
│ Stage 4: CONTEXT COLLAPSE (marble_origami)                 │
│ ──────────────────────────────────────────                 │
│ 트리거: `marble_origami` feature flag 활성시                │
│ 동작:                                                      │
│   1. tool_result를 선택적으로 "접기" (fold)                 │
│   2. "[Collapsed: 파일 읽기 결과, 250줄]" placeholder       │
│   3. 모델이 필요시 "펼치기" 요청 가능                       │
│   4. 실질적인 "가상 메모리" 효과                            │
│ 왜 중요한가: 긴 대화에서 가장 효율적인 토큰 관리            │
└────────────────────────────────────────────────────────────┘
```

### 5.2 Token Budget 시스템

```
입력 토큰 계산:
  system_prompt_tokens + message_history_tokens + tool_schemas_tokens
  = total_input_tokens

예산 한계:
  model_context_window - reserved_output_tokens - safety_margin
  = available_input_budget

예: Opus 4.6 [1M context]
  1,000,000 - 32,000(출력) - 10,000(안전) = 958,000 입력 가능

TOKEN_BUDGET "+500k" 플래그:
  → 컴팩션 후에도 자동으로 작업 계속
  → 500K 토큰 소진 후에야 정지 고려
```

---

## 6. 메모리 시스템 심층 분석

### 6.1 3중 메모리 구조

```
┌─────────────────────────────────────────────────────────┐
│ Layer 1: CLAUDE.md (4계층)                               │
│ ────────────────────────                                │
│                                                         │
│ /etc/claude-code/CLAUDE.md       Managed (조직 관리자)   │
│   └─ 조직 전체 규칙, 보안 정책                           │
│                                                         │
│ ~/.claude/CLAUDE.md              User (사용자 전역)      │
│   └─ 개인 선호, 글로벌 설정                              │
│                                                         │
│ ./CLAUDE.md + .claude/rules/*.md Project (프로젝트)      │
│   └─ 프로젝트별 코딩 규칙, 아키텍처 가이드              │
│                                                         │
│ ./CLAUDE.local.md                Local (비공개)          │
│   └─ git-ignored, 개인 메모                             │
│                                                         │
│ 로딩: 부트시 4계층 병합 → 시스템 프롬프트 동적 부분      │
├─────────────────────────────────────────────────────────┤
│ Layer 2: Memdir (구조화된 메모리)                        │
│ ─────────────────────────────                           │
│                                                         │
│ ~/.claude/projects/<slug>/memory/                        │
│   ├─ MEMORY.md          인덱스 (200줄 제한)              │
│   ├─ user_role.md       사용자 정보                      │
│   ├─ feedback_*.md      행동 피드백                      │
│   ├─ project_*.md       프로젝트 컨텍스트                │
│   └─ reference_*.md     외부 리소스 참조                 │
│                                                         │
│ Frontmatter 형식:                                       │
│   ---                                                   │
│   name: "메모리 이름"                                    │
│   description: "한줄 설명"                               │
│   type: user|feedback|project|reference                  │
│   ---                                                   │
│   내용...                                                │
│                                                         │
│ 규칙:                                                   │
│   - MEMORY.md 항상 대화 컨텍스트에 로딩                  │
│   - 200줄 넘으면 잘림                                   │
│   - 각 항목 150자 이하                                   │
│   - 코드 패턴/아키텍처는 저장 금지 (코드에서 파악 가능)  │
├─────────────────────────────────────────────────────────┤
│ Layer 3: Auto Memory Extraction (자동 추출)              │
│ ──────────────────────────────────────────               │
│                                                         │
│ 동작 방식:                                               │
│   1. 대화 종료 후 백그라운드 포크 에이전트 실행           │
│   2. 대화에서 저장할 만한 정보 자동 식별                  │
│   3. Memdir에 자동 생성/업데이트                         │
│                                                         │
│ KAIROS Dream 시스템 (미출시):                            │
│   - 24시간 + 5세션 이상 경과시                           │
│   - 전체 메모리 통합/정리                                │
│   - "autoDream" 서비스                                   │
│   - 모순 해결, 오래된 메모리 제거                        │
└─────────────────────────────────────────────────────────┘
```

### 6.2 Magic Docs 시스템

```
파일에 `# MAGIC DOC:` 헤더가 있으면:
  → 에이전트가 자동으로 최신 상태로 업데이트
  → 일종의 "살아있는 문서"
  → 프로젝트 지식의 자동 갱신
```

---

## 7. 숨겨진 기능 & Feature Flag 상세

### 7.1 Feature Flag 시스템

```
Bun의 build-time DCE (Dead Code Elimination):
  feature("FLAG_NAME") → 빌드시 true/false로 교체
  → false인 코드는 번들에서 제거
  → 런타임 오버헤드 제로

GrowthBook (A/B 테스팅):
  → 사용자별 feature flag 원격 제어
  → 점진적 롤아웃
  → 실험 데이터 수집

Random Word Pair 이름 생성:
  → flag 이름이 "marble_origami", "velvet_thunder" 등
  → 코드 검색 방지 (obfuscation)
```

### 7.2 주요 미출시 기능 상세

#### KAIROS (Always-On 자율 에이전트 데몬)

```
┌──────────────────────────────────────┐
│ KAIROS 아키텍처                      │
│                                      │
│ ┌─ 백그라운드 데몬 ─────────┐        │
│ │  ├─ 레포 파일 변경 모니터  │        │
│ │  ├─ PR 이벤트 구독         │        │
│ │  ├─ 스케줄 작업 실행       │        │
│ │  └─ 메모리 통합 (Dream)    │        │
│ └────────────────────────────┘        │
│          │                            │
│          ▼                            │
│ ┌─ 사용자 인터페이스 ───────┐         │
│ │  ├─ 푸시 알림 전송         │         │
│ │  ├─ 파일 전달              │         │
│ │  └─ 대화 없이 작업 완료    │         │
│ └────────────────────────────┘        │
│                                      │
│ 전용 도구:                            │
│  SleepTool, PushNotificationTool,    │
│  SendUserFileTool, SubscribePRTool   │
└──────────────────────────────────────┘
```

#### COORDINATOR MODE (멀티에이전트 오케스트레이션)

```
coordinatorMode.ts (19KB):

┌─ Coordinator Agent ─────────────────────┐
│  ├─ 작업 분해 (Task Decomposition)       │
│  ├─ 에이전트 할당                         │
│  ├─ 진행 추적                             │
│  └─ 결과 통합                             │
├──────────────────────────────────────────┤
│  Worker Agent 1 (git worktree A)         │
│  Worker Agent 2 (git worktree B)         │
│  Worker Agent 3 (git worktree C)         │
│  ...                                     │
├──────────────────────────────────────────┤
│  각 Worker:                               │
│  ├─ 독립 git worktree에서 격리 작업      │
│  ├─ 부모와 SendMessage로 소통            │
│  ├─ 완료시 결과 반환 or 변경사항 브랜치   │
│  └─ 실패시 worktree 자동 정리            │
└──────────────────────────────────────────┘
```

#### BUDDY 시스템 (18종 ASCII 펫)

```
CompanionSprite.tsx (72KB):

종류: duck, goose, blob, cat, dragon, octopus, owl, penguin,
      turtle, snail, ghost, axolotl, capybara, cactus, robot,
      rabbit, mushroom, chonk

레어도 가챠:
  Common(60%) → Uncommon(25%) → Rare(10%) → Epic(4%) → Legendary(1%)
  + 1% Shiny 변형

스탯: DEBUGGING / PATIENCE / CHAOS / WISDOM / SNARK

결정론적 생성: 사용자 ID → 해시 → 항상 같은 버디
```

#### ULTRAPLAN (30분 딥 씽킹)

```
원격 컨테이너에서 30분간 멀티에이전트 탐색:
  1. 문제 분석 에이전트
  2. 코드 탐색 에이전트 (여러 개)
  3. 해결 방안 생성 에이전트
  4. 통합/평가 에이전트
  → 30분 후 상세 계획 반환
```

#### 기타

| 기능 | 설명 |
|------|------|
| **ULTRATHINK** | 확장 사고 모드, 무지개 애니메이션 표시 |
| **VOICE_MODE** | Whisper STT 통합, 음성 입력 |
| **BRIDGE_MODE** | JWT+WebSocket 원격 제어 (IDE 통합) |
| **VERIFICATION_AGENT** | 적대적 서브에이전트가 결과 검증 |
| **TRANSCRIPT_CLASSIFIER** | 대화 패턴 분석 → 자동 권한 모드 추천 |
| **Undercover Mode** | ANT 직원이 공개 레포 작업시 AI 사용 숨김 |
| **Speculation** | 사용자 입력 전 다음 응답 추측 사전 계산 |
| **Penguin Mode** | Fast Mode 내부 코드명 |
| **Advisor** | 서버측 암호화 도구 |

---

## 8. 멀티 언어 포트 비교

### 8.1 TypeScript (원본) vs Python vs Rust

| 측면 | TypeScript (원본) | Python (claw-code-agent) | Rust (claurst) |
|------|-------------------|--------------------------|----------------|
| **LOC** | ~800K | ~50K | ~48K |
| **파일 수** | 1,900+ | 109 | 155 |
| **런타임** | Bun | CPython | tokio |
| **UI** | React + Ink | readline | ratatui + crossterm |
| **스키마** | Zod v4 | 수동 검증 | serde + schemars |
| **비동기** | AsyncGenerator | asyncio | async/await + channels |
| **API SDK** | @anthropic-ai/sdk | 직접 HTTP | reqwest |
| **MCP** | @modelcontextprotocol/sdk | 직접 구현 (30KB) | 직접 구현 |
| **모델** | Claude 전용 | Qwen3, Ollama, LiteLLM | Claude + OpenAI compat |
| **의존성** | 192 npm 패키지 | **제로** (stdlib) | ~30 crates |
| **권한** | 4모드 + 5단계 bash | hook_policy.py | 동일 구조 |
| **메모리** | 3중 구조 | 기본 | 동일 구조 |
| **특이점** | 원본 | Discord 멀티에이전트, 무료 모델 | 텔레메트리 제거, 실험적 기능 활성 |

### 8.2 Python 포트의 독특한 점

```
claw-code (Discord 멀티에이전트):
  PHILOSOPHY: "Humans set direction; claws perform the labor"
  ├─ clawhip: 조율자
  ├─ oh-my-codex: 코드 에이전트
  └─ oh-my-openagent: 범용 에이전트
  → Discord를 통한 에이전트 간 소통

claw-code-agent (독립 에이전트):
  ├─ 제로 의존성 (순수 Python stdlib)
  ├─ Qwen3-Coder 30B (vLLM) 지원
  ├─ Ollama / LiteLLM / OpenRouter 지원
  ├─ OpenAI 호환 API 레이어
  └─ 1,581 테스트 통과 (open-claude-code v2)
```

### 8.3 Rust 포트의 독특한 점

```
claurst (11 crate 모듈):
  ├─ 990KB 행위 명세 (spec/ 디렉토리)
  │   → 원본 TS의 완전한 행위를 문서화
  │   → 이것만으로도 독립적 참고 가치
  ├─ 텔레메트리 완전 제거
  ├─ 실험적 기능 기본 활성화
  ├─ mock parity harness (10 시나리오, 19 API 요청)
  └─ bash_classifier.rs (18.5KB) / ps_classifier.rs (20.4KB)
      → 명령어 위험도 분류를 Rust로 재구현
```

---

## 9. 프로덕션 엔지니어링 인사이트

### 9.1 "하네스가 전부다" (Haseeb Qureshi)

> **"API 호출은 ~200줄. 나머지 ~500,000줄이 하네스.
> 아무도 하네스를 이야기하지 않고 모두 모델 지능을 이야기하지만,
> 모델이야말로 가장 교체 가능한 부분이다."**

이 말이 의미하는 것:

```
전체 코드 500,000줄의 구성:

API 호출 (200줄) ████ 0.04%
시스템 프롬프트 조립 ██████████ 2%
도구 시스템 (42+) ████████████████████████████████ 15%
권한/보안 (300KB+) ████████████████████████ 12%
UI/UX (React+Ink) ████████████████████████████████████████ 25%
상태 관리 ████████████████ 8%
메모리 시스템 ██████████ 5%
Compaction ████████ 4%
MCP/Bridge ████████████ 6%
유틸리티 ████████████████████████████████████████████ 23%
```

### 9.2 프로덕션에서 발견된 흔적

| 흔적 | 추론 |
|------|------|
| **비대칭 세션 영속화** | 세션 유실 사고 → 사용자 메시지만 동기 저장 |
| **Thinking 경고 (medieval English)** | 하루 디버깅 후 추가. thinking 블록이 API 미지원 형식일 때 |
| **Anti-hallucination** | "모든 테스트 통과" 거짓말 → 명시적 금지 규칙 |
| **25단어 규칙** | A/B 테스트로 ~1.2% output token 감소 검증 |
| **ANT 빌드 차이** | 내부 사용자에게 더 엄격한 규칙 (≤25단어 제한) |
| **marble_origami** | 난수 단어쌍 = 소스코드 검색 방지 |
| **Hook 시스템** | 권한 시스템만으로 불충분 → 사용자 커스텀 셸 훅 추가 |
| **rate-limiter.ts** | 429/529 에러 → 지수 백오프 + jitter |

### 9.3 텔레메트리 & 원격 제어

```
텔레메트리 (sanbuphy 분석):
  2개 수집 파이프라인:
    ├─ 1P (Anthropic 자체)
    └─ Datadog
  수집 데이터:
    ├─ 환경 핑거프린트
    ├─ 도구 사용 패턴
    ├─ 에러/충돌 보고
    └─ 세션 통계
  UI 옵트아웃 없음 (환경변수만 가능)

원격 제어:
  매시간 폴링: /api/claude_code/settings
  ├─ 위험한 변경 → 블로킹 다이얼로그 표시
  ├─ 6+ 킬스위치
  └─ 특정 기능 원격 비활성화 가능
```

---

## 10. 생태계 & 영향

### 10.1 유출 타임라인 (2026.03.31)

```
00:21 UTC  Axios npm 하이잭 (UNC1069/북한, RAT 악성코드)
04:00 UTC  Claude Code v2.1.88 npm 배포 (.npmignore 미설정)
04:23 UTC  Chaofan Shou가 sourcemap에서 소스 발견
06:00 UTC  바이럴 확산, 2시간 내 41,500 포크
08:00 UTC  Anthropic 패키지 철회, "인적 오류" 발표
09:00 UTC  커뮤니티 리라이트 시작 (claw-code 등)
이후       DMCA로 8,100+ GitHub 레포 삭제, 하지만 이미 확산

⚠️ 동일 날 Axios RAT 공격과 동시 발생
   → 같은 날 npm install 했다면 장비 오염 가능성
```

### 10.2 유출 후 생태계 폭발

```
awesome-cli-coding-agents 기준 (80+ 프로젝트):

오픈소스 주요 에이전트:
  OpenCode (122K★)    → 프라이버시 중심, 75+ 프로바이더
  Claw Code (110K★)   → 유출 소스 Python/Rust 리라이트
  Gemini CLI (98K★)   → Google의 에이전트
  OpenHands (69K★)    → 웹 + CLI
  Codex CLI (66K★)    → OpenAI의 에이전트
  ...

OpenClaw 생태계 (유출 기반 파생):
  OpenClaw (322K★)    → 원본 로컬 AI 어시스턴트
  nanobot (34.6K★)    → 4,000줄 Python 리라이트
  ZeroClaw (27.8K★)   → Rust, <5MB RAM
  PicoClaw (25.3K★)   → Go, $10 하드웨어에서 구동

오케스트레이션 도구:
  claude-flow (21.6K★) → 스웜 오케스트레이션
  claude-code-router (29.9K★) → 대체 프로바이더 지원
  vibe-kanban (23.4K★) → 세션 관리
```

### 10.3 everything-claude-code (프로덕션 프레임워크)

```
50K+ GitHub ★, 6K+ 포크, 30 기여자
Anthropic Hackathon 우승

포함 내용:
  ├─ 38+ 사전 빌드 에이전트
  │   ├─ architect, code-reviewer, tdd-guide, planner
  │   ├─ security-reviewer, performance-optimizer
  │   ├─ cpp/java/rust/python-reviewer (언어별)
  │   └─ go/dart/kotlin-build-resolver (빌드 해결)
  ├─ 스킬 시스템 (Python, Spring Boot, Django 등)
  ├─ 80+ 슬래시 명령어
  ├─ 멀티 언어 규칙 (TS, Python, Go, Java, PHP, Kotlin, C++, Rust, Perl)
  ├─ MCP 서버 설정
  └─ 997+ 내부 테스트 통과
```

---

## 11. 핵심 요약: 반드시 알아야 할 것

### 11.1 아키텍처 핵심 5가지

1. **시스템 프롬프트 캐시 분할** — 정적/동적 경계(`__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`)로 서버 비용 절감
2. **거부 피드백 루프** — 도구 거부가 에러로 래핑되어 모델에 전달 → 자동 대안 탐색
3. **4단계 Compaction** — Proactive → Reactive → Snip → Context Collapse
4. **Deferred Tools** — 도구 스키마 지연 로딩으로 시스템 프롬프트 토큰 절약
5. **3중 메모리** — CLAUDE.md(규칙) + Memdir(구조화) + Auto-extraction(자동)

### 11.2 보안 핵심 3가지

1. **Bash 300KB 보안 코드** — 5단계 위험 분류 + 경로 검증 + 샌드박스
2. **4모드 권한 계층** — default → plan → acceptEdits → bypass
3. **Hook 시스템** — PreToolUse/PostToolUse 셸 훅으로 커스텀 정책

### 11.3 숨겨진 기능 핵심 5가지

1. **KAIROS** — Always-on 자율 데몬 + Dream 메모리 통합
2. **COORDINATOR** — 멀티에이전트 git worktree 격리 오케스트레이션
3. **ULTRAPLAN** — 30분 원격 멀티에이전트 딥 플래닝
4. **BUDDY** — 18종 ASCII 펫 타마고치 (결정론적 가챠)
5. **marble_origami** — 가상 메모리 스타일 컨텍스트 접기/펼치기

### 11.4 프로덕션 교훈

```
"모델은 교체 가능, 하네스가 진짜 가치"
  → API 200줄, 하네스 500,000줄
  → 수년간의 프로덕션 경험이 하네스에 응축

"비대칭 저장은 사고에서 배운 것"
  → 사용자 메시지: 동기 저장 (절대 유실 금지)
  → 어시스턴트 메시지: 비동기 저장 (약간 유실 허용)

"25단어 규칙은 A/B 테스트 결과"
  → 1.2% 출력 토큰 감소 → 수백만 요청에서 막대한 비용 절감

"거부를 학습 신호로 사용"
  → 대부분 에이전트는 거부시 멈춤
  → Claude Code는 자동으로 더 안전한 대안 탐색
```

---

## 부록: 파일 크기 기준 핵심 파일 TOP 20

| # | 파일 | 크기 | 역할 |
|---|------|------|------|
| 1 | `screens/REPL.tsx` | 875KB | 메인 REPL 화면 |
| 2 | `main.tsx` | 785KB | CLI 진입점 |
| 3 | `tools/AgentTool/` | 228KB | 서브에이전트 시스템 |
| 4 | `hooks/useTypeahead.tsx` | 208KB | 자동완성 |
| 5 | `hooks/useReplBridge.tsx` | 113KB | REPL-Bridge 연결 |
| 6 | `bridge/bridgeMain.ts` | 113KB | 원격 세션 메인 |
| 7 | `hooks/useVoiceIntegration.tsx` | 97KB | 음성 통합 |
| 8 | `bridge/replBridge.ts` | 98KB | REPL 브릿지 |
| 9 | `services/bashPermissions.ts` | 96KB | Bash 보안 |
| 10 | `buddy/CompanionSprite.tsx` | 72KB | 버디 스프라이트 |
| 11 | `query.ts` | 67KB | 에이전트 루프 |
| 12 | `services/readOnlyValidation.ts` | 66KB | 읽기전용 검증 |
| 13 | `utils/interactiveHelpers.tsx` | 56KB | 인터랙티브 UI |
| 14 | `QueryEngine.ts` | 46KB | 쿼리 엔진 |
| 15 | `services/pathValidation.ts` | 42KB | 경로 검증 |
| 16 | `bridge/remoteBridgeCore.ts` | 39KB | 원격 브릿지 코어 |
| 17 | `Tool.ts` | 29KB | 도구 기본 클래스 |
| 18 | `commands.ts` | 25KB | 명령 레지스트리 |
| 19 | `state/AppState.tsx` | 23KB | 상태 관리 |
| 20 | `coordinator/coordinatorMode.ts` | 19KB | 코디네이터 |

---

*보고서 생성: 2026-04-15 | 25개 레포 분석 기반 | 총 분석 소스: ~800K+ LOC TypeScript + 50K Python + 48K Rust*
