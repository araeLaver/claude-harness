# Claude Code 소스코드 유출 - 4개 레포 종합 분석 보고서
## 2026.03.31 npm sourcemap 유출 사건

---

## 사건 개요

2026년 3월 31일, Chaofan Shou가 npm 패키지에 포함된 sourcemap(.map) 파일을 통해 Claude Code 전체 소스를 발견.
- **규모**: ~1,900개 TypeScript 파일, ~512,000줄 (800K+ LOC)
- **원인**: Bun bundler의 기본 sourcemap 생성 설정을 끄지 않은 패키징 실수
- **결과**: Anthropic DMCA로 8,100+ GitHub 레포 삭제, 하지만 이미 확산

---

## 분석 대상 4개 레포

| # | 레포 | 유형 | 보고서 |
|---|------|------|--------|
| 1 | `claude-code-yasas` | 원본 유출 TypeScript 소스 | REPORT_1 |
| 2 | `claw-cli-v2.1.88` | v2.1.88 소스 재구성 (더 완전) | REPORT_2 |
| 3 | `haseeb-analysis-gist` | Haseeb Qureshi의 기술 분석 | REPORT_3 |
| 4 | `claurst` | Rust 완전 리라이트 + 행위 명세 990KB | REPORT_4 |

---

## 핵심 아키텍처 요약

### 기술 스택
```
런타임: Bun
언어: TypeScript (~800K LOC)
UI: React + Ink (터미널용 React)
CLI: Commander.js
스키마: Zod v4
API: @anthropic-ai/sdk
MCP: @modelcontextprotocol/sdk
Feature Flag: Bun build-time DCE + GrowthBook
```

### 에이전트 루프 (query.ts)
```
사용자 입력 → 시스템 프롬프트 조립 (15개 composable 함수)
  → API 호출 (SSE 스트리밍)
    → AsyncGenerator로 이벤트 스트림
      → tool_use 감지 → 도구 실행 (병렬/직렬)
        → 결과 피드백 → 다시 API 호출
          → 자동 컴팩션 (4단계)
            → 토큰 예산 추적
              → 종료 조건 체크
```

---

## 시스템 프롬프트 구조

### 정적/동적 분할 (캐시 최적화)
```
[정적 - 글로벌 캐시, Blake2b hash]
├── Intro: "You are an interactive agent..."
├── System: 도구/태그/권한 규칙
├── Doing Tasks: 코딩 스타일, 보안 주의
├── Actions: 되돌릴 수 없는 작업 확인
├── Using Tools: 전용 도구 우선
├── Tone & Style: 이모지 금지, 간결함
└── Output Efficiency

__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__

[동적 - 세션별]
├── Session Guidance
├── CLAUDE.md 메모리
├── 환경 정보 (Git, OS, 모델)
├── MCP 서버 지시
├── 출력 스타일
└── 함수 결과 정리 (FRC)
```

### ANT(내부) vs 외부 빌드 프롬프트 차이
- **내부**: ≤25단어 제한, ≤100단어 최종 응답, "모든 테스트 통과" 거짓 주장 금지
- **외부**: "Go straight to the point"

---

## 도구 시스템 (42+)

### 카테고리별 도구

| 카테고리 | 도구 |
|---------|------|
| **파일** | BashTool, FileRead/Edit/WriteTool, GlobTool, GrepTool |
| **에이전트** | AgentTool(228KB), SendMessageTool, TeamCreate/DeleteTool |
| **웹** | WebFetchTool, WebSearchTool, WebBrowserTool(flag) |
| **태스크** | TaskCreate/Get/Update/List/Output/StopTool |
| **플랜** | EnterPlanMode/ExitPlanModeV2Tool, VerifyPlanExecutionTool |
| **MCP** | MCPTool, McpAuthTool, ListMcpResources/ReadMcpResourceTool, ToolSearchTool |
| **Kairos** | SleepTool, PushNotificationTool, SendUserFileTool, SubscribePRTool |
| **Cron** | CronCreate/Delete/ListTool, RemoteTriggerTool |
| **노트북** | NotebookEditTool |
| **기타** | AskUserQuestionTool, SkillTool, EnterWorktreeTool, TodoWriteTool |

---

## 메모리 시스템 (3중 구조)

### 1. CLAUDE.md (4계층)
```
/etc/claude-code/CLAUDE.md       → 조직 관리 (Managed)
~/.claude/CLAUDE.md              → 사용자 전역 (User)
./CLAUDE.md, .claude/rules/*.md  → 프로젝트 (Project)
./CLAUDE.local.md                → 로컬 비공개 (Local)
```

### 2. Memdir (구조화된 메모리)
```
~/.claude/projects/<slug>/memory/MEMORY.md  (200줄/25KB 제한)
~/.claude/projects/<slug>/memory/topic.md   (개별 파일)
```
4가지 타입: `user`, `feedback`, `project`, `reference`

### 3. 자동 메모리 추출
백그라운드 포크 에이전트가 대화에서 자동으로 메모리 생성/업데이트.

---

## Compaction 4단계 전략

| 단계 | 트리거 | 동작 |
|------|--------|------|
| **Proactive** | 매 턴 토큰 모니터링, 한계 접근 | 자동 요약 (사용자 무감지) |
| **Reactive** | API `prompt_too_long` 반환 | 소급 컴팩션 후 재시도 |
| **Snip** | SDK/headless 모드 | 경계에서 잘라냄 |
| **Context Collapse** | `marble_origami` flag | tool 결과 선택적 압축 (복원 가능) |

---

## 권한 시스템

### 모드
`default` → `plan` → `acceptEdits` → `bypassPermissions`

### 파이프라인
```
Mode check → Hook evaluation → Rule matching → User prompt
```

### Bash 보안 (300KB+!)
- 5단계 위험도: Safe → Low → Medium → High → Critical
- Critical → `Forbidden` 레벨로 무조건 차단
- `bashPermissions.ts`(96KB), `pathValidation.ts`(42KB), `readOnlyValidation.ts`(66KB)

### 거부 피드백
거부가 tool result로 래핑되어 모델에게 전달 → 접근 방식 자동 조정.

---

## 숨겨진 기능 종합

| 기능 | Flag | 설명 |
|------|------|------|
| **KAIROS** | `KAIROS` | Always-on 자율 에이전트 데몬 |
| **Buddy System** | `BUDDY` | 18종 ASCII 펫 타마고치 (결정론적 가챠) |
| **Coordinator** | `COORDINATOR_MODE` | Multi-agent 오케스트레이션 |
| **Ultrathink** | `ULTRATHINK` | 확장 사고 모드 (무지개 애니메이션) |
| **Ultraplan** | `ULTRAPLAN` | 30분 원격 멀티에이전트 탐색 |
| **Voice Mode** | `VOICE_MODE` | 음성 STT + Whisper |
| **Dream System** | `KAIROS_DREAM` | 24h+5세션 메모리 통합 데몬 |
| **Bridge Mode** | `BRIDGE_MODE` | JWT+WebSocket 원격 제어 |
| **Undercover** | env var | ANT 개발자 모델/프로젝트명 은닉 |
| **Verification Agent** | `VERIFICATION_AGENT` | 적대적 sub-agent 검증 |
| **Context Collapse** | `marble_origami` | tool 결과 선택적 압축 |
| **Token Budget** | `TOKEN_BUDGET` | "+500k" 자동 계속 작업 |
| **Speculation** | 내장 | 입력 전 다음 응답 추측 계산 |
| **Penguin Mode** | 내부 코드명 | Fast Mode |
| **Advisor** | 서버측 | 암호화된 결과 지원 서버 도구 |
| **Magic Docs** | 내장 | `# MAGIC DOC:` 파일 자동 업데이트 |

### Buddy 18종
```
duck, goose, blob, cat, dragon, octopus, owl, penguin,
turtle, snail, ghost, axolotl, capybara, cactus, robot,
rabbit, mushroom, chonk
```
- Common(60%) → Uncommon(25%) → Rare(10%) → Epic(4%) → Legendary(1%)
- 1% Shiny 변형, 스탯: DEBUGGING/PATIENCE/CHAOS/WISDOM/SNARK

---

## 모델 설정

### 지원 모델
```
claude-opus-4-6 (프론티어)
claude-opus-4-5-20251101, claude-opus-4-1-20250805, claude-opus-4-20250514
claude-sonnet-4-6, claude-sonnet-4-5-20250929, claude-sonnet-4-20250514
claude-3-7-sonnet-20250219, claude-3-5-sonnet-20241022
claude-haiku-4-5-20251001, claude-3-5-haiku-20241022
```

### 모델 코드명
- **Capybara**: Claude 4.6 계열
- **Fennec**: Opus 4.6
- **Numbat**: 미확인

### 기본 선택
- Max/Team Premium → Opus 4.6 [1m]
- Pro/Enterprise/Standard → Sonnet 4.6
- ANT 내부 → Opus 4.6 [1m]

### Effort: `low` / `medium` / `high` / `max`
### Thinking: `adaptive` / `enabled(budgetTokens)` / `disabled`
### 프로바이더: `firstParty` / `bedrock` / `vertex` / `foundry`

---

## Haseeb Qureshi의 핵심 인사이트

> **"API 호출은 ~200줄. 나머지 ~500,000줄이 전부 harness.
> 아무도 harness를 이야기하지 않고 모두 모델 지능을 이야기하지만,
> 모델이야말로 가장 교체 가능한 부분이다."**

### 프로덕션 현장의 흔적
1. **비대칭 세션 영속화**: 세션 손실 사고 → 사용자 메시지 동기 저장
2. **Thinking 규칙**: 하루 디버깅 후 medieval English로 경고
3. **Anti-hallucination**: "모든 테스트 통과" 거짓말 방지 명시적 추가
4. **25단어 규칙**: A/B 테스트로 ~1.2% output token 감소 검증

### Claude Code vs Codex 비교
| | Claude Code | Codex |
|---|---|---|
| 언어 | TypeScript 500K LOC | Rust |
| UI | React + Ink | ratatui |
| 동시성 | AsyncGenerator (yield) | Async channel |
| Compaction | 4단계 | 2단계 + remote |
| 권한 | App-level + 거부 피드백 | OS-level 샌드박싱 |

---

## CLAURST Rust 리라이트

### 구조
11개 crate: `cli`, `core`, `api`, `tools`, `query`, `tui`, `commands`, `mcp`, `bridge`, `buddy`, `plugins`

### 주요 차이
- React/Ink → `ratatui` + `crossterm`
- Node.js → `tokio` async
- Zod → `serde` + `schemars`
- 1,902 파일 → ~100 .rs 파일
- 텔레메트리 완전 제거
- 실험적 기능 기본 활성화

### 행위 명세 (`spec/`, 990KB)
15개 문서에 원본 TypeScript 소스의 완전한 행위 명세 기록. 이것만으로도 독립적 가치가 있음.

---

## 파일 목록

```
/Volumes/WorkDrive/Develop/46.claude-code-harness/
├── REPORT_0_종합요약.md          (이 파일)
├── REPORT_1_claude-code-yasas.md  (원본 유출 소스 분석)
├── REPORT_2_claw-cli-v2.1.88.md  (v2.1.88 소스 분석)
├── REPORT_3_haseeb-analysis.md    (Haseeb Qureshi 분석)
├── REPORT_4_claurst.md            (Rust 리라이트 분석)
├── claude-code-yasas/             (원본 유출 소스)
├── claw-cli-v2.1.88/             (v2.1.88 소스)
├── haseeb-analysis-gist/          (분석 gist)
└── claurst/                       (Rust 리라이트)
```
