# Haseeb Qureshi의 Claude Code 소스 코드 유출 분석
## 레포: Haseeb-Qureshi/d0dc36844c19d26303ce09b42e7188c1 (Gist)

---

## 분석 파일 정보

- **파일**: `claude-code-harness-deep-dive.md`
- **저자**: Haseeb Qureshi
- **분석 대상**: Anthropic의 Claude Code CLI 소스 코드 (~1,900개 파일, TypeScript)
- **비교 대상**: OpenAI의 Codex

---

## 1. 핵심 논점: "Harness가 전부다"

> **"아무도 harness에 대해 이야기하지 않고 모두 어떤 모델이 더 똑똑한지에 대해 이야기하지만, 두 코드베이스를 비교해 보면 모델이야말로 가장 교체 가능한 부분이고, harness에 수년간의 프로덕션 경험이 축적되어 있다."**

Claude Code는 약 **500,000줄의 TypeScript**로 구성되어 있으며, 실제 API 호출은 약 200줄에 불과하다. 나머지 전부가 "harness"이다:
- 4계층 compaction
- 프롬프트 캐시 경계
- 거부 피드백이 있는 권한 파이프라인
- 비대칭 세션 영속화
- Build-time feature flag
- 추측적 사전 계산
- 스트리밍 중 메모리 프리페치

---

## 2. 요청 생명주기 (Request Lifecycle)

전체 처리 흐름:

1. **입력 처리** - slash command, 첨부 파일, queued command 파싱
2. **시스템 프롬프트 조립** - 15개의 composable 함수로 구성, `DYNAMIC_BOUNDARY`로 캐시 분할
3. **파일 스냅샷** - LRU 캐시 (100개 파일, 25MB) 기반의 undo 지원
4. **query() 루프** - `while(true)` 구조의 async generator로 LLM 응답 스트리밍, tool 실행, 결과 피드백을 반복
5. **세션 영속화** - JSON transcript로 저장

핵심: 전체 시스템은 **async generator** 기반. 텍스트 청크, tool call, 진행 상태, 오류, compaction 경계 등 모든 이벤트가 하나의 `yield` 기반 스트림을 통해 흐른다.

**Codex와의 차이점**: Codex는 Rust `Session` struct에 `submit()`과 `next_event()` 메서드를 사용하는 async channel 패턴. Claude Code의 generator 패턴은 `yield*`로 sub-generator에 위임할 수 있어 더 composable.

---

## 3. 터미널 UI는 React 앱이다

진입점이 `src/entrypoints/cli.tsx`이며, 전체 터미널 인터페이스가 **Ink** (터미널용 React) 기반의 React 컴포넌트 트리로 구성. 메시지 버블, tool call 디스플레이, 권한 프롬프트, markdown 렌더러 모두 React 컴포넌트.

**Codex**: Rust의 `ratatui`와 `crossterm`으로 TUI 구성. 메모리 오버헤드/렌더링 속도에서 유리하지만, Claude Code 방식은 웹 경험을 구축하는 동일 팀이 CLI에도 기여할 수 있는 이점.

---

## 4. Compaction 4단계 전략 (가장 중요한 발견)

긴 세션에서 context window가 넘칠 때의 **4단계 계층적 compaction 전략**:

### 4.1 Proactive Compaction
매 턴마다 토큰 수를 모니터링하여 한계에 가까워지면 이전 메시지를 "compact boundary" 마커로 요약. API 전송 전에 수행되므로 **사용자는 실패를 경험하지 않음**.

### 4.2 Reactive Compaction
폴백 전략. proactive 체크가 놓친 경우(race condition, 부정확한 토큰 추정) API가 `prompt_too_long`을 반환하면 소급 compaction 후 재시도. 사용자는 약간의 지연만 경험.

### 4.3 Snip Compaction
SDK/headless 모드 전용. 요약 대신 정의된 경계에서 잘라내어 장기 자동화 세션에서 메모리를 제한. REPL은 스크롤백을 위한 전체 히스토리를 유지하지만 SDK는 불필요.

### 4.4 Context Collapse (`marble_origami` feature flag)
전체 compaction을 트리거하지 않고 대화 중간의 장황한 tool 결과를 압축. 예: 3턴 전 500줄 tool 출력을 짧은 표현으로 축소.

핵심 특이점: collapse commit이 `ContextCollapseCommitEntry` 레코드로 transcript에 영속화되어 **선택적으로 복원 가능**. Codex에는 이에 상응하는 기능이 없음.

**Codex 비교**: Codex는 `reference_context_item`으로 턴 간 설정 스냅샷을 추적하고 diff를 통해 변경된 항목만 전송. Claude Code는 매 턴 전체 시스템 프롬프트를 재전송하되 prompt caching에 의존. Claude Code의 4계층 fallback이 더 방어적이고, Codex의 diff 기반이 턴당 토큰 효율이 더 높음.

---

## 5. 시스템 프롬프트 세부 사항

### 5.1 프롬프트 캐시 경계 기법
약 15개의 composable 함수로 조립, `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 마커로 분할:

- **상단 (정적)**: 행동 지침, 코딩 스타일, 안전 가이드 → `scope: 'global'`로 API에 캐싱
- **하단 (동적)**: CLAUDE.md, MCP 지시, 환경 정보 → 세션별 주입

소스에 "Blake2b prefix hash variants" 관련 주석 → 캐시 가능한 접두사를 해싱하여 **사용자 간 캐시 히트를 극대화**. 약 3,000 토큰의 지침이 전역적으로 캐싱 및 재사용.

### 5.2 Anthropic 내부 프롬프트 차이

`process.env.USER_TYPE === 'ant'` 체크로 내부/외부 다른 지침:

- **내부**: "사용자의 요청이 잘못된 이해에 기반한 경우 그렇게 말하라"
- **내부**: "출력에 실패가 표시될 때 절대로 '모든 테스트 통과'라고 주장하지 마라" (anti-hallucination guardrail)
- **내부**: "tool call 사이의 텍스트를 25단어 이하로 유지하라" - ~1.2% output token 감소
- **내부**: `VERIFICATION_AGENT` flag - 비사소한 변경 후 적대적 sub-agent 검증

코드에 `@[MODEL LAUNCH]` 마커(예: `capy v8 thoroughness counterweight (PR #24302)`)가 있어 A/B 테스트 추적.

### 5.3 Undercover 모드
`isUndercover()` 모드: 시스템 프롬프트에서 모든 모델 이름/ID를 제거. 내부 모델 식별자가 공개 commit/PR에 유출되지 않도록 함.

Dead code elimination 주석:
```typescript
// DCE: `process.env.USER_TYPE === 'ant'` is build-time
// --define. It MUST be inlined at each callsite (not
// hoisted to a const) so the bundler can constant-fold
// it to `false` in external builds and eliminate the
// branch.
```

---

## 6. 숨겨진 기능/미출시 기능

Build-time DCE를 통해 feature flag로 관리되는 미출시 기능:

| Feature Flag | 설명 |
|-------------|------|
| `VOICE_MODE` | 음성 모드 |
| `COORDINATOR_MODE` | 코디네이터 모드 |
| `KAIROS` | 자율적 proactive agent |
| `ABLATION_BASELINE` | A/B 테스트 인프라 |
| `PROACTIVE` | proactive 기능 |
| `marble_origami` | Context Collapse |
| `VERIFICATION_AGENT` | 적대적 검증 sub-agent |

Bun의 `feature()` 함수를 사용한 compile-time feature flag:
```typescript
import { feature } from 'bun:bundle'
if (feature('VOICE_MODE')) {
  // stripped at build time if flag is off
}
```

동일 코드베이스에서 공개 CLI, 내부 빌드, SDK, IDE bridge 등 서로 다른 빌드를 feature set만 달리하여 생성.

---

## 7. 도구 정의 및 권한 시스템

### 7.1 권한 파이프라인
모든 tool call은 다음 파이프라인 통과:
1. **Mode check** - 현재 모드 확인
2. **Hook evaluation** - 훅 평가
3. **Rule matching** - 규칙 매칭
4. **User prompt** - 사용자 프롬프트

### 7.2 거부(Denial) 피드백 메커니즘
가장 주목할 점: **거부가 모델에게 피드백**된다. 권한 거부가 tool result로 래핑되어 마치 tool이 오류를 반환한 것처럼 모델에게 전달. Claude는 "permission denied"를 tool 출력으로 인식하고 접근 방식을 조정.

세션 전체의 모든 거부를 추적하여 SDK 호출자에게 보고 → IDE 통합에서 패턴 표시 가능 (예: "사용자가 /etc에서 bash 명령을 계속 거부함").

**Codex**: OS 수준 샌드박싱. 커널 수준에서 `rm -rf /` 같은 명령 시도 자체를 차단. Linux에서 `prctl(2)`로 parent death signal 설정.

---

## 8. 메모리/컨텍스트 관리

### 8.1 파일 스냅샷
LRU 캐시 (100개 파일, 25MB)로 undo 지원.

### 8.2 메모리 프리페치
스트리밍 중 모델 응답 생성과 병렬로 CLAUDE.md에서 관련 메모리를 미리 가져옴:

```typescript
using pendingMemoryPrefetch =
  startRelevantMemoryPrefetch(
    state.messages,
    state.toolUseContext,
  )
```

`using` 키워드 = TC39 explicit resource management. generator의 모든 종료 경로에서 프리페치가 정리되도록 보장. 프리페치는 스트림과 병렬 실행 → tool 실행 시점에 결과 이미 준비 → I/O 지연 은닉.

### 8.3 Speculation (추측적 사전 계산)
사용자가 타이핑하는 동안 예측 가능한 다음 응답을 미리 계산. 사용자가 Enter 누르기 전에 응답이 준비될 수 있음.

---

## 9. Thinking 규칙

`query.ts`의 유명한 주석:
```
The rules of thinking are lengthy and fortuitous...
1. thinking block을 포함하는 메시지는 max_thinking_length > 0인 query의 일부여야 함
2. thinking block이 block의 마지막 메시지가 될 수 없음
3. thinking block은 assistant trajectory 기간 동안 보존되어야 함
```

이 주석은 **하루 종일 디버깅한 후 남긴 경고**. 대충 넘기지 못하도록 의도적으로 mock-medieval English로 작성.

---

## 10. 비대칭 세션 영속화

의도적으로 비대칭적 설계:
- **사용자 메시지**: `await`하여 동기적으로 저장
- **어시스턴트 메시지**: fire-and-forget으로 비동기 저장

이유: 실제 프로덕션 사고에서 비롯. 사용자가 Enter 누른 후 API 응답 전에 프로세스 종료 → transcript에 queue-operation 항목만 남음 → `--resume` 시 "No conversation found" 오류. 해결: 사용자 메시지는 항상 동기 영속화, 어시스턴트 메시지는 API 응답에 이미 내구성이 있으므로 비동기 처리.

**Codex**: JSONL + SQLite 기반. 더 무거운 인프라이지만 더 쿼리 가능.

---

## 11. Claude Code vs Codex 비교표

| 영역 | Claude Code | Codex |
|------|------------|-------|
| 언어 | TypeScript (~500K LOC) | Rust |
| UI 프레임워크 | React + Ink (터미널) | ratatui + crossterm |
| 동시성 모델 | Async generator (yield) | Async channel (submit/next_event) |
| Compaction | 4단계 (proactive, reactive, snip, context collapse) | 2단계 (pre-turn, mid-turn) + remote |
| 프롬프트 캐싱 | 전역 캐시 경계 분할 (Blake2b hash) | 없음 (작은 프롬프트 유지) |
| 세션 저장 | JSON transcript (비대칭) | JSONL + SQLite |
| 권한 모델 | 애플리케이션 수준 파이프라인 + 거부 피드백 | OS 수준 샌드박싱 + escalation |
| Feature flag | Bun build-time DCE | 정보 없음 |

---

## 12. 프로덕션 현장의 흔적 (Production Scars)

소스 코드 전반에 프로덕션에서 겪은 고통의 흔적:

1. **비대칭 영속화**: 세션 손실 사고에서 비롯
2. **Thinking 규칙 주석**: 하루 종일 디버깅한 결과물 (medieval English)
3. **Anti-hallucination guardrail**: "모든 테스트 통과"라고 거짓말하는 문제가 심각해서 명시적 추가
4. **25단어 간결성 규칙**: A/B 테스트로 ~1.2% output token 감소 정량적 검증
5. **DCE 보안 주석**: 각 callsite에서 인라인 필수, const 호이스팅 금지 (실제 누출 방지)
