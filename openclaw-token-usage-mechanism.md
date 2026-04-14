# OpenClaw 토큰 사용량 기록 메커니즘

## 개요

OpenClaw은 모든 AI 모델 호출의 토큰 사용량과 비용을 세션 파일에 기록하고, 다차원으로 집계하여 조회할 수 있다.

---

## 1. 기록 흐름 (4단계)

```
[1] 프로바이더 응답 → Transport Stream에서 usage 추출
    ↓
[2] pi-agent-core의 SessionManager가 JSONL 파일에 assistant 메시지 기록 (message.usage)
    ↓
[3] Subscribe handler에서 run 전체 usage 누적 (recordAssistantUsage)
    ↓
[4] Run 완료 후 sessions.json에 누적 토큰/비용 업데이트 (updateSessionStoreAfterAgentRun)
```

### 세션 파일 위치

```
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
```

### 세션 파일 usage 필드 구조

```jsonl
{
  "message": {
    "role": "assistant",
    "content": "...",
    "usage": {
      "input": 1234,
      "output": 567,
      "cacheRead": 890,
      "cacheWrite": 0,
      "totalTokens": 2691,
      "cost": {
        "total": 0.0042,
        "input": 0.0030,
        "output": 0.0012,
        "cacheRead": 0.0000,
        "cacheWrite": 0.0000
      }
    },
    "provider": "anthropic",
    "model": "sonnet-4.6",
    "durationMs": 1234
  },
  "timestamp": "2026-04-14T10:30:00.000Z"
}
```

---

## 2. Embedded Provider vs CLI Backend

| 항목 | Embedded (omni-route 등 API 직접 호출) | CLI Backend (claude-cli) |
|---|---|---|
| usage 소스 | API 응답의 `usage` 필드 | CLI stdout의 JSON/JSONL 파싱 |
| cost 계산 시점 | Transport에서 즉시 (`calculateCost`) | session-store 업데이트 시 후계산 |
| cacheRead | API 응답에서 추출 | CLI 출력의 다양한 필드명 지원 |
| cacheWrite | Anthropic만 제공, OpenAI-compatible은 항상 0 | CLI가 보고하면 기록 |

### Embedded Provider 경로

각 transport stream이 프로바이더 응답에서 usage를 추출:
- **Anthropic**: `message_start` 이벤트의 `cache_read_input_tokens`, `cache_creation_input_tokens`
- **OpenAI/omni-route**: `input_tokens_details.cached_tokens` → cacheRead, cacheWrite는 항상 0
- **Google Gemini**: `usageMetadata.cachedContentTokenCount` → cacheRead

OpenAI-compatible API에서 `prompt_tokens`는 cached tokens를 포함하므로, `input = prompt_tokens - cached_tokens`로 보정하여 이중 카운팅을 방지.

### CLI Backend 경로

`src/agents/cli-output.ts`의 `toCliUsage` 함수가 다양한 형식을 정규화:
- `input_tokens`, `inputTokens`, `prompt_tokens` 등 모든 필드명 지원
- `cache_read_input_tokens`, `cached_input_tokens`, `cacheRead`, `cached` 등 모든 cache 필드명 지원
- CLI 경로에서는 `calculateCost`가 호출되지 않음 — session-store 업데이트 시 후계산

---

## 3. Cost 계산 (3중 레이어)

### 레이어 1: Transport 레벨 (즉시)

각 transport stream에서 `calculateCost(model, usage)` 호출하여 `usage.cost` 필드를 채움.
JSONL 세션 파일의 `message.usage.cost`에 기록됨.

### 레이어 2: Session-store 레벨 (run 완료 시)

`src/utils/usage-format.ts`의 `estimateUsageCost` 함수:
```
total = (input * cost.input + output * cost.output + cacheRead * cost.cacheRead + cacheWrite * cost.cacheWrite) / 1,000,000
```

모델 가격 조회 우선순위 (`resolveModelCostConfig`):
1. `models.json` (로컬 모델 정의)
2. `config` (openclaw.json의 모델 설정)
3. Gateway pricing cache

### 레이어 3: 리딩 시 후계산

`src/infra/session-cost-usage.ts`에서 세션 파일 읽을 때, cost가 없는 항목에 대해 후계산.

---

## 4. Usage 정규화

`src/agents/usage.ts`의 `normalizeUsage` 함수가 모든 프로바이더 형식을 통합:

| 프로바이더 필드명 | 정규화 대상 |
|---|---|
| `input_tokens` / `inputTokens` / `prompt_tokens` / `promptTokens` | `input` |
| `output_tokens` / `outputTokens` / `completion_tokens` / `completionTokens` | `output` |
| `cache_read_input_tokens` / `cached_tokens` / `input_tokens_details.cached_tokens` | `cacheRead` |
| `cache_creation_input_tokens` / `cache_write` | `cacheWrite` |

---

## 5. 조회 방법

### CLI

```bash
# 최근 30일 비용 요약
openclaw gateway usage-cost --days 30

# JSON 출력
openclaw gateway usage-cost --days 30 --json
```

### Gateway RPC

```bash
# 세션별 상세 사용량 (다차원 집계 포함)
openclaw gateway call sessions.usage --params '{"startDate":"2026-04-01","endDate":"2026-04-14"}'

# 기간별 비용 요약
openclaw gateway call usage.cost --params '{"days":30}'

# 특정 세션의 시계열 데이터
openclaw gateway call sessions.usage.timeseries --params '{"key":"agent:main:direct:U0718RUK69G"}'

# 특정 세션의 로그 엔트리
openclaw gateway call sessions.usage.logs --params '{"key":"...", "limit":200}'
```

### 집계 가능한 차원

| 차원 | RPC 응답 필드 | 설명 |
|---|---|---|
| 모델별 | `aggregates.byModel` | provider+model 조합별 토큰/비용 |
| 프로바이더별 | `aggregates.byProvider` | 프로바이더별 합계 |
| 에이전트별 | `aggregates.byAgent` | agentId별 합계 |
| 채널별 | `aggregates.byChannel` | slack, telegram 등 채널별 |
| 일별 | `aggregates.daily` | 날짜별 tokens, cost, messages |
| 도구별 | `aggregates.tools` | 도구 이름별 호출 횟수 |
| 레이턴시 | `aggregates.latency` | 응답 지연 통계 (avg, p95, min, max) |
| 모델+일별 | `aggregates.modelDaily` | 날짜+모델별 세부 사용량 |
| 메시지 유형별 | `aggregates.messages` | user/assistant/toolCalls/errors 카운트 |

---

## 6. 주요 소스코드 파일

| 역할 | 파일 |
|---|---|
| Usage 정규화 | `src/agents/usage.ts` |
| Transport: Anthropic | `src/agents/anthropic-transport-stream.ts` |
| Transport: OpenAI | `src/agents/openai-transport-stream.ts` |
| Transport: Google | `src/agents/google-transport-stream.ts` |
| CLI 출력 파싱 | `src/agents/cli-output.ts` |
| CLI runner | `src/agents/cli-runner.ts` |
| 세션 비용 파싱/집계 | `src/infra/session-cost-usage.ts` |
| 세션 스토어 업데이트 | `src/agents/command/session-store.ts` |
| 비용 추정/포매팅 | `src/utils/usage-format.ts` |
| RPC 핸들러 | `src/gateway/server-methods/usage.ts` |
| CLI 등록 | `src/cli/gateway-cli/register.ts` |
| 타입 정의 | `src/infra/session-cost-usage.types.ts`, `src/shared/usage-types.ts` |
| 프로토콜 스키마 | `src/gateway/protocol/schema/sessions.ts` |

---

*문서 작성일: 2026-04-14 / OpenClaw v2026.4.12 기준*
