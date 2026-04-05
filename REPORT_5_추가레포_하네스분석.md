# Claude Code Harness 추가 레포 11개 종합 분석 보고서
## + 주요 블로그 분석글 3편 수집

---

## 추가 수집 레포 현황

| # | 레포 | 유형 | Star | 상태 |
|---|------|------|------|------|
| 5 | `learn-claude-code` | 교육 (12세션 하네스 구축) | - | 클론 완료 |
| 6 | `open-claude-code` | Clean-room 재구현 (npm 배포) | - | 클론 완료 |
| 7 | `everything-claude-code` | 하네스 최적화 (158 스킬) | 50K+ | 클론 완료 |
| 8 | `claw-code` | GitHub 역사상 최고속 100K star | 110K+ | 클론 완료 |
| 9 | `claudecode-leak-autopsy` | 기술적 부검 아카이브 | - | 클론 완료 |
| 10 | `collection-claude-code` | 모든 소스+분석 원스톱 아카이브 | - | 클론 완료 |
| 11 | `claw-code-agent` | Pure Python 재구현 (zero deps) | - | 클론 완료 |
| 12 | `deep-dive-claude-code` | 13챕터 분석 + 웹 시뮬레이터 | - | 클론 완료 |
| 13 | `claude-code-analysis` | 역공학 분석 문서 (35K 단일 문서) | - | 클론 완료 |
| 14 | `awesome-cli-coding-agents` | 80+ CLI agent 큐레이션 디렉토리 | - | 클론 완료 |
| 15 | `claude-code-harness-chachamaru` | Plan→Work→Review 하네스 플러그인 | - | 클론 완료 |
| - | `clawd-codex` | DMCA/계정 비활성화 | - | 실패 |
| - | `SinghCoder/claude-code` | DMCA 삭제 | - | 실패 |

---

## 유형별 분류

### A. 교육/학습형

#### learn-claude-code (shareAI-lab) — "Bash is All You Need"

**핵심 철학**: "모델이 agent이고, 코드는 harness다"

12단계 점진적 세션으로 agent harness를 바닥부터 구축:

```
s01: agent_loop.py      (121줄)  - while True + 1개 tool
s02: tool_use.py                  - tool dispatch map 4개 tools
s03: todo_write.py                - TodoWrite 계획 시스템
s04: subagent.py                  - 독립 messages[] 서브에이전트
s05: skill_loading.py             - SKILL.md on-demand 주입
s06: context_compact.py           - 3-layer 압축 전략
s07: task_system.py               - 파일 기반 task graph + 의존성
s08: background_tasks.py          - daemon thread + 알림 큐
s09: agent_teams.py               - 팀원 + JSONL mailbox
s10: team_protocols.py            - shutdown + plan approval FSM
s11: autonomous_agents.py         - idle cycle + auto-claim
s12: worktree_task_isolation.py   - worktree 격리
s_full.py (35.6K)                 - 전체 통합 capstone
```

**에이전트 루프의 본질** (121줄):
```python
def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({"type": "tool_result",
                                "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})
```

> **핵심 통찰**: 매 세션에서 정확히 하나의 harness 메커니즘만 추가. 루프 자체는 절대 변경하지 않음.

#### deep-dive-claude-code — 13챕터 시각적 분석

```
Ch01: Agent Loop (QueryEngine.ts + query.ts)
Ch02: Tool System (Tool.ts + 42개 도구)
Ch03: Prompt Engineering (prompts.ts + claudemd.ts)
Ch04: Shell Security (bashSecurity.ts 300KB+)
Ch05: Permission Engine (permissions.ts)
Ch06: Context Management (compact.ts + SessionMemory)
Ch07: MCP Protocol (mcp/client.ts + mcp/auth.ts)
Ch08: Plugin Ecosystem (pluginLoader.ts)
Ch09: Multi-Agent (AgentTool.tsx + swarm/)
Ch10: CLI Transport (cli/transports/)
Ch11: Bootstrap Optimization (진입점 체인)
Ch12: Production Patterns (sessionStorage + analytics)
Ch13: Hidden Features (buddy + ultraplan + undercover)
```

**4-tab 웹 플랫폼** 포함: Visualization(애니메이션), Simulator(agent loop 리플레이), Source(구문 강조), Deep Dive(Markdown)

---

### B. 완전 재구현형

#### open-claude-code (ruvnet) — Clean-Room 재구현

**법적 안전**: US DMCA §1201(f), EU Software Directive Art. 6에 근거

```
v2/src/
├── core/
│   ├── agent-loop.mjs    # async generator 기반 (13 event types)
│   ├── context-manager.mjs
│   ├── providers.mjs     # 5개 AI provider (Anthropic/OpenAI/Google/Bedrock/Vertex)
│   ├── streaming.mjs
│   └── system-prompt.mjs
├── tools/        # 25개 tool
├── permissions/  # 6가지 permission mode
├── hooks/        # hook 시스템
├── mcp/          # 4개 MCP transport (stdio/SSE/StreamableHTTP/WebSocket)
└── auth/         # multi-provider 인증
```

- **Nightly Verified Release**: 매일 03:00 UTC에 npm 레지스트리 감시 → 903+ 테스트 → AI 기반 변경 감지 → 자동 배포
- **ruDevolution 연동**: AI 기반 decompiler로 34,759+ 함수 95.7% 이름 정확도 복원
- `npx @ruvnet/open-claude-code "explain this codebase"` 즉시 실행 가능

#### claw-code (UltraWorkers) — 100K Star 최고속 레포

**철학**: "Stop Staring at the Files — 코드를 보지 말고, 그 코드를 생산한 시스템을 봐라"

```
3-part 시스템:
├── OmX (oh-my-codex)      — 짧은 지시 → 구조화된 실행
├── clawhip                 — git/tmux/GitHub 이벤트 라우터
└── OmO (oh-my-openagent)  — 다중 에이전트 계획/핸드오프/이견 해소
```

Rust workspace 9개 crate: `api`, `commands`, `runtime`, `tools`, `plugins`, `telemetry`, `compat-harness`, `mock-anthropic-service`, `rusty-claude-cli`

> **독특**: "lobsters/claws"(자율 에이전트)가 직접 저장소를 관리하며, 인간은 Discord에서 한 문장만 입력

#### claw-code-agent (HarnessLab) — Pure Python Zero Deps

**로컬 모델 최적화**: Qwen3-Coder-30B-A3B-Instruct 우선 지원

```python
# 59개 Python 모듈, agent_runtime.py (142.4K!)
src/
├── agent_runtime.py      # 핵심 런타임 (142.4K)
├── agent_prompting.py    # 프롬프트 조립
├── agent_tools.py        # 툴 레지스트리
├── agent_manager.py      # 에이전트 계보 추적
├── openai_compat.py      # OpenAI 호환 API 클라이언트
├── plugin_runtime.py     # 매니페스트 기반 플러그인
├── mcp_runtime.py        # stdio MCP transport
├── plan_runtime.py       # 태스크/플랜 런타임
├── hook_policy.py        # 훅 + 정책 런타임
├── tokenizer_runtime.py  # 토크나이저 인식 컨텍스트
└── ... (49개 추가)
```

- **Zero Dependencies**: Python 표준 라이브러리만 사용
- **Nested Agent Delegation**: 의존성 인식 위상 정렬 기반 batch 처리
- **Budget 시스템**: 토큰/비용/tool-call/model-call/세션 턴 한도
- **PARITY_CHECKLIST.md (21K)**: npm 원본 대비 모든 기능의 구현 상태 추적

---

### C. 하네스/플러그인형

#### everything-claude-code (affaan-m) — Anthropic 해커톤 우승작

**Cross-harness 비전**: Claude Code, Codex, Cursor, Gemini, OpenCode, Kiro, Trae 모두 지원

```
everything-claude-code/
├── agents/    # 40개 전문 에이전트 (architect, code-reviewer, planner...)
├── skills/    # 158개 스킬 (agentic-engineering, autonomous-loops, cost-aware-pipeline...)
├── hooks/     # hooks.json (28.7K) -- PreToolUse/PostToolUse 자동화
├── rules/     # 12+ 언어 생태계 규칙
├── SOUL.md    # "Agent-First, Test-Driven, Security-First" 5대 원칙
├── the-longform-guide.md   # Token 최적화, 메모리, eval 상세 가이드
├── the-security-guide.md   # 공격 벡터, 샌드박싱, CVE 가이드
└── .claude-plugin/         # 플러그인 매니페스트
```

주요 Hook:
- `pre:bash:block-no-verify` — git hook bypass 차단
- `pre:bash:auto-tmux-dev` — 디렉토리 기반 tmux 자동 시작
- `pre:bash:commit-quality` — lint, commit message 검증, 시크릿 감지

#### claude-code-harness-chachamaru — Plan→Work→Review 워크플로우

**5-verb**: Setup → Plan → Work → Review → Release

```
core/src/
├── index.ts           # stdin → route → stdout 파이프라인
├── guardrails/
│   ├── rules.ts       # 선언적 규칙 R01-R09
│   ├── pre-tool.ts    # PreToolUse 훅
│   ├── post-tool.ts   # PostToolUse 훅
│   ├── permission.ts  # PermissionRequest 훅
│   └── tampering.ts   # 변조 감지
```

**v3 아키텍처** — 11개 에이전트를 3개로 통합:
- `worker` = task-worker + codex-implementer + error-recovery
- `reviewer` = code-reviewer + plan-critic (Read-only)
- `scaffolder` = project-analyzer + state-updater

**CC 버전 추종**: v2.1.50~v2.1.90의 100+ 기능을 A(구현)/B(미구현)/C(CC 자동 계승)으로 분류

---

### D. 분석/문서형

#### claudecode-leak-autopsy (0PeterAdel) — 기술적 부검

분 단위 사건 타임라인:
```
00:21 UTC  Axios npm 하이재킹 (UNC1069, 북한 연계)
04:00 UTC  Claude Code v2.1.88 배포 (.npmignore 오류)
04:23 UTC  Chaofan Shou 발견, src.zip 링크 공유
06:00 UTC  바이럴 확산 (2시간 내 41,500 fork)
08:00 UTC  Anthropic 패키지 철회, "human error" 발표
```

주요 발견:
- **ANTI_DISTILLATION_CC**: 경쟁사 API 트래픽 스크래핑 방해용 가짜 tool 정의 주입
- **YOLO Permission Classifier**: 저위험 tool call 자동 승인 ML 분류기
- **Skeptical Memory**: grep 기반 사실 검증 규율

#### collection-claude-code (chauncygu) — 원스톱 아카이브

```
collection-claude-code/
├── original-source-code/       # 원본 유출 (1,884 파일)
├── claude-code-source-code/    # 디컴파일 (1,940 파일, 163,318줄)
├── claw-code/                  # Python 재작성 (109 파일)
├── nano-claude-code/           # 미니멀 재구현 (v1.0→v3.0: 1,300→5,000줄)
└── docs/cc_analysis/
    ├── Claude_Code上下文压缩算法深度分析.md   # compress 파이프라인 역공학 (11 파일)
    ├── Claude_Code记忆系统深度分析.md         # 7-layer 메모리 시스템
    ├── Claude_Code隐藏高级功能全景图.md       # 55+ feature flag, 20+ 숨겨진 명령어
    ├── Claude_Code模型切换机制深度分析.md     # 모델 선택 로직
    ├── claude-code-multi-agent-architecture.md # 37+ 파일 multi-agent
    └── Claude_Code_Notable_Algorithms.md      # 주목할 알고리즘/패턴
```

> **중국어 커뮤니티의 분석이 가장 기술적으로 상세**. context 압축 11개 파일 역공학, 7-layer 메모리 분석은 다른 어디에서도 볼 수 없는 수준.

#### claude-code-analysis (ComeOnOliver) — 35K 단일 문서

17개 섹션: Tool System(41개 tool), Command System(101개 command), State Management, Task System, Services, UI Layer(130+ 컴포넌트), Utilities(300+), Special Modes, Plugins & Skills, Architecture Patterns

#### awesome-cli-coding-agents (bradAGI) — 생태계 지형도

80+ CLI coding agent 큐레이션:
- **Open Source**: OpenCode(122K), Claw Code(110K), Gemini CLI(98K), OpenHands(69.3K), Codex CLI(66.1K)
- **OpenClaw 생태계**: OpenClaw(322K), nanobot, ZeroClaw, PicoClaw, NanoClaw, IronClaw
- **Harness/Orchestrator**: claude-flow(21.6K), vibe-kanban(23.4K), cmux(8.1K)
- **인프라**: claude-code-router(29.9K), agent-browser(23.3K)

---

## 블로그 분석글 3편 핵심 요약

### WaveSpeedAI — "Agent Harness Architecture"

**서브에이전트 3가지 실행 모델**:

| 모델 | 동작 | 용도 |
|------|------|------|
| **Fork** | 바이트 동일 컨텍스트 복사 | 프롬프트 캐시 활용 병렬 작업 |
| **Teammate** | tmux/iTerm 별도 패인 + JSONL mailbox | 느슨한 조율 |
| **Worktree** | 격리된 git worktree + 별도 브랜치 | 탐색적/위험한 작업 |

**권한 3단계**:
1. 프로젝트 로드 시 신뢰 수립
2. 도구 실행 전 권한 체크
3. 고위험 작업에 대한 명시적 확인

**Auto 모드**: 별도 LLM이 "이 작업을 사용자가 승인할 것인가?"를 평가 — "무엇을 할 것인가"와 "이것이 허용되는가"를 분리.

### Engineer's Codex — "Diving into Claude Code's Source"

**새로운 발견들**:
- **Anti-Distillation Defense**: `anti_distillation` 플래그로 가짜 tool 정의를 주입하여 경쟁사 학습 데이터 오염
- **CONNECTOR_TEXT**: tool call 사이의 assistant 텍스트를 암호화 서명된 요약으로 변환
- **DRM/API Attestation**: Bun의 Zig 네이티브 HTTP 스택에서 xxHash 서명 추가 (JS 런타임 아래에서 실행되므로 패칭 불가)
- **Magic Documentation**: 제한된 서브에이전트가 `# MAGIC DOC:` 헤더 파일을 자동 업데이트

### DEV.to — "Agent Orchestration Masterclass"

**컨텍스트 엔트로피 관리**:
> "1,200개 이상의 세션에서 50회 이상 연속 컴팩션 실패를 경험한 후, 단 3줄의 수정으로 재시도를 3회로 제한하여 전 세계적으로 하루 ~250,000회의 낭비된 API 호출을 절약"

**4가지 아키텍처 돌파구**:
1. Context Entropy Management (4단계 파이프라인)
2. Tool Orchestration & Security (Bash tool만 2,500줄 보안 검증)
3. Multi-Agent Coordination (포크 서브에이전트 격리)
4. KAIROS Memory System (autoDream 통합)

---

## 하네스 핵심 기능 종합 정리 (기존 4개 + 추가 11개 + 블로그 3편)

### 1. Anti-Distillation Defense (신규 발견!)
경쟁사의 API 트래픽 스크래핑을 방해하기 위한 **가짜 tool 정의 주입** + `CONNECTOR_TEXT` 암호화 서명.

### 2. DRM/API Attestation (신규 발견!)
Bun의 **Zig 네이티브 HTTP 스택**에서 xxHash 기반 요청 서명. JS 런타임 아래에서 실행되므로 런타임 패칭/프록시 불가.

### 3. YOLO Permission Classifier (신규 발견!)
저위험 tool call을 자동 승인하는 **별도 ML 분류기**. "무엇을 할 것인가"와 "이것이 허용되는가"의 완전 분리.

### 4. 서브에이전트 3가지 모델 (상세화)
- **Fork**: 프롬프트 캐시 공유 병렬 작업
- **Teammate**: JSONL mailbox 느슨한 조율
- **Worktree**: git worktree 격리된 탐색

### 5. 컨텍스트 엔트로피 관리 (수치 발견!)
- 1,200+ 세션에서 50+ 연속 컴팩션 실패 → 3줄 수정으로 하루 ~250,000회 API 호출 절약

### 6. Skeptical Memory (상세화)
메모리를 **힌트**로 취급하고 실제 코드베이스에 대해 grep 기반 사실 검증 수행. "trust but verify."

### 7. Magic Documentation
제한된 서브에이전트가 `# MAGIC DOC:` 헤더 파일을 **자동으로 업데이트** — 문서 부패 방지.

### 8. 7-Layer 메모리 (중국어 분석에서 발견!)
```
Layer 1: MEMORY.md (인덱스, 150자/항목)
Layer 2: Topic files (on-demand 로드)
Layer 3: Raw transcripts (grep만 가능)
Layer 4: Session memory (10개 섹션)
Layer 5: Team memory (TEAMMEM flag)
Layer 6: AutoDream (24h+5세션 통합)
Layer 7: Extract memories (포크 에이전트 자동 생성)
```

### 9. 55+ Feature Flags (중국어 분석에서 확장!)
기존 30+에서 55+로 확장 발견. 20+ 숨겨진 명령어 추가 확인.

### 10. nano-claude-code 미니멀 재구현 경로
```
v1.0: 1,300줄 → v2.0: 3,400줄 → v3.0: 5,000줄
```
하네스의 본질을 점진적으로 이해할 수 있는 최소 구현 시리즈.
