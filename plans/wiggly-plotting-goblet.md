# SAP Assistant Desktop Platform — v6.1.0 고도화 계획

## Context

SAP Assistant Desktop Platform v6.0.0은 탄탄한 3계층 Electron 아키텍처를 갖추고 있지만, 코드 탐색 결과 **3가지 치명적 품질 이슈**가 발견됨:

1. **에이전트 파이프라인**: 순차 실행만 지원, 스텝 간 컨텍스트 약함, Provider 하드코딩, 에러 시 전체 중단
2. **ProviderResilience 미연결**: retry/circuit breaker/fallback 코드가 존재하지만 **아무 곳에서도 호출되지 않음**
3. **IPC 에러 핸들링 미적용**: `wrapHandler()` 정의만 있고 143개 채널 어디에도 적용 안 됨

**목표**: 기능 추가 없이 안정성·실행 흐름·코드 품질을 높여 v6.1.0으로 릴리즈

---

## Phase 1: 에이전트 파이프라인 고도화 (핵심)

### 문제 (executor.ts 152줄)

| 줄 | 문제 | 영향 |
|---|------|------|
| 64-128 | `for...of` 순차 루프 | 독립 스텝도 병렬 불가 |
| 86-93 | 문자열 연결로 컨텍스트 전달 | 구조화 데이터 손실 |
| 107-108 | `provider: "copilot"`, `model: "gpt-4o"` 하드코딩 | 사용자 설정 무시 |
| 131-146 | catch 하나로 전체 중단 | 부분 재시도/스킵 불가 |
| — | 타임아웃 없음 | LLM hang 시 무한 대기 |

### 변경 파일

**`src/main/types/agent.ts`** — 타입 확장
- `StepContextData`: 구조화된 스텝 결과 (output을 JSON 파싱 시도, metadata 포함)
- `PipelineContext`: `Map<stepId, StepContextData>` + globalContext
- `StepCondition`: 조건분기용 (reference, operator, value)
- `AgentStep` 확장: `executeIf?`, `retry?`, `timeoutMs?`, `providerOverride?`, `modelOverride?`, `fallbackSkillId?`
- `AgentExecutionProgress`: UI 진행률 이벤트 타입

**`src/main/agents/executor.ts`** — 핵심 리팩토링 (~50% 재작성)
- **DAG 기반 스케줄링**: dependsOn 그래프에서 독립 스텝 그룹 추출 → `Promise.all()` 병렬 실행
- **구조화 컨텍스트**: 스텝 결과를 `StepContextData`로 저장, JSON 파싱 시도 + string fallback
- **Provider 유연성**: `step.providerOverride` → 세션 기본값 → fallback 체인
- **조건분기**: `executeIf` 평가 → false면 "skipped" 처리 (에러 아님)
- **스텝별 재시도**: `retry.maxAttempts` + `fallbackSkillId` → 실패 시에도 나머지 스텝 계속
- **타임아웃**: `AbortSignal.timeout(step.timeoutMs ?? 300_000)`
- **진행률**: `AgentExecutionProgress` 이벤트를 IPC로 전송

**`src/main/ipc/agentHandlers.ts`** — 진행률 IPC 채널 추가
- `AGENTS_EXECUTION_PROGRESS` 채널 등록

**`src/main/ipc/channels.ts`** — 채널 상수 추가

**`src/main/agents/__tests__/executor.test.ts`** — 신규 (350줄)
- 순차/병렬 실행, 컨텍스트 전파, 조건분기, Provider override, fallback, 타임아웃, 취소

### 하위 호환성
모든 신규 필드는 optional → 기존 프리셋 에이전트 2개 변경 없이 동작

---

## Phase 2: ProviderResilience 연결

### 문제
`providerResilience.ts`(103줄)에 `withRetry()`, `withCircuitBreaker()`, `withFallback()` 구현되어 있지만 ChatRuntime·CboAnalyzer 어디에서도 호출하지 않음. 모든 LLM 호출이 resilience 없이 직접 실행 중.

### 변경 파일

**`src/main/providers/providerResilience.ts`** — 메서드 추가
- `withProviderCall<T>(provider, fn, options?)`: retry + circuit breaker + timeout 통합 래퍼

**`src/main/chatRuntime.ts`** — DI 주입
- 생성자에 `providerResilience: ProviderResilience` 파라미터 추가
- `sendMessage()` / `sendMessageWithStream()` 내 provider 직접 호출을 `withProviderCall()`로 래핑

**`src/main/cbo/analyzer.ts`** — DI 주입
- 생성자에 `providerResilience` 추가
- `analyzeContent()` 내 LLM enrichment 호출 래핑

**`src/main/bootstrap/createServices.ts`** — 와이어링
```typescript
const providerResilience = new ProviderResilience();
// → chatRuntime, cboAnalyzer 생성자에 주입
```

**`src/main/providers/__tests__/providerResilience.test.ts`** — 신규 (200줄)
- retry 지수 백오프, circuit breaker 상태 전이, fallback, 타임아웃

---

## Phase 3: IPC 에러 핸들링 표준화

### 문제
`wrapHandler()` 정의만 있고 10개 핸들러 모듈 어디에도 적용 안 됨. 에러가 비일관적으로 전파.

### 변경 파일

**`src/main/ipc/helpers/wrapHandler.ts`** — 강화
- 실행 시간 측정 + 구조화 로깅 (channel, elapsed, error)
- 에러 메시지 정규화: `[channel] message` 형식

**9개 핸들러 모듈에 wrapHandler 적용:**

| 파일 | 수정량 |
|------|--------|
| `ipc/agentHandlers.ts` | ~10줄 |
| `ipc/chatHandlers.ts` | ~10줄 |
| `ipc/cboHandlers.ts` | ~10줄 |
| `ipc/authHandlers.ts` | ~10줄 |
| `ipc/sourceHandlers.ts` | ~10줄 |
| `ipc/closingHandlers.ts` | ~10줄 |
| `ipc/routineHandlers.ts` | ~10줄 |
| `ipc/scheduleHandlers.ts` | ~10줄 |
| `ipc/archiveHandlers.ts` | ~10줄 |

패턴: `ipcMain.handle(CH, wrapHandler(CH, async (_e, args) => { ... }))`

**`src/main/ipc/__tests__/wrapHandler.test.ts`** — 신규 (100줄)

---

## Phase 4: 테스트 커버리지 확충

### 현재: 15개 파일, 20% threshold
### 목표: 25+ 파일, 35% threshold

### 신규 테스트 파일

| 파일 | 줄 | 대상 |
|------|---|------|
| `agents/__tests__/executor.test.ts` | 350 | Phase 1에서 작성 |
| `providers/__tests__/providerResilience.test.ts` | 200 | Phase 2에서 작성 |
| `ipc/__tests__/wrapHandler.test.ts` | 100 | Phase 3에서 작성 |
| `chatRuntime.test.ts` | 250 | 세션 관리, Provider 선택, 스트리밍 |
| `cbo/__tests__/analyzer.test.ts` | 150 | 규칙 분석 + LLM enrichment |
| `cbo/__tests__/batchRuntime.test.ts` | 150 | 배치 흐름, progress, 취소 |
| `agents/__tests__/registry.test.ts` | 80 | 에이전트 CRUD, 중복 제거 |
| `ipc/__tests__/agentHandlers.test.ts` | 80 | IPC 통합 |

**`vitest.config.ts`** — threshold 상향: lines 35%, functions 35%, branches 25%

---

## 파일 변경 요약

| # | 파일 | Phase | 작업 |
|---|------|-------|------|
| 1 | `src/main/types/agent.ts` | 1 | 타입 확장 (+150줄) |
| 2 | `src/main/agents/executor.ts` | 1 | 핵심 리팩토링 (~50%) |
| 3 | `src/main/ipc/channels.ts` | 1 | 채널 상수 추가 |
| 4 | `src/main/ipc/agentHandlers.ts` | 1,3 | 진행률 + wrapHandler |
| 5 | `src/main/providers/providerResilience.ts` | 2 | 통합 래퍼 메서드 |
| 6 | `src/main/chatRuntime.ts` | 2 | ProviderResilience DI |
| 7 | `src/main/cbo/analyzer.ts` | 2 | ProviderResilience DI |
| 8 | `src/main/bootstrap/createServices.ts` | 2 | 와이어링 |
| 9 | `src/main/ipc/helpers/wrapHandler.ts` | 3 | 강화 |
| 10-17 | `src/main/ipc/*Handlers.ts` (8개) | 3 | wrapHandler 적용 |
| 18 | `vitest.config.ts` | 4 | threshold 상향 |
| 19-26 | `__tests__/*.test.ts` (8개) | 1-4 | 신규 테스트 |

**총 변경**: 수정 17개 + 신규 8개 = **25개 파일**

---

## 실행 순서

```
Phase 1 ──┐
           ├─→ Phase 3 (독립) ──→ Phase 4 (검증)
Phase 2 ──┘
```

Phase 1·2는 병렬 진행 가능, Phase 3은 독립, Phase 4는 마지막

---

## 검증

```bash
cd /c/Users/chois/sap-ops-bot
npm run typecheck       # 3개 tsconfig 모두 에러 0
npm run lint            # ESLint 통과
npm run test:run        # 기존 + 신규 테스트 통과
npm run test:coverage   # 35% threshold 충족
npm run verify          # build + lint + test 통합
```

기존 프리셋 에이전트 2개 (transport-review-workflow, incident-resolution-workflow) 정상 동작 확인
