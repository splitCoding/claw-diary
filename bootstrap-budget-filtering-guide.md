# OpenClaw 부트스트랩 파일 — 예산, 잘림, 필터링 쉬운 설명

---

## 한 줄 요약

에이전트가 시작할 때 워크스페이스의 지침 파일들을 시스템 프롬프트에 넣어주는데,
**너무 크면 잘라내고**, **세션 종류에 따라 안 넣는 파일도 있다.**

---

## 1. 토큰 예산 관리

에이전트의 시스템 프롬프트 공간은 한정되어 있다.
부트스트랩 파일들이 이 공간을 독점하면 안 되므로, 두 가지 한도가 있다.

### 한도

| 항목 | 기본값 | 의미 |
|------|--------|------|
| 파일당 한도 | 20,000자 | 파일 하나가 차지할 수 있는 최대 크기 |
| 전체 한도 | 150,000자 | 모든 부트스트랩 파일의 합계 최대 크기 |

### 예산이 적용되는 방식

파일들이 우선순위 순서대로 예산을 나눠 가진다.

```
예산 통: 150,000자

1번째 파일 (AGENTS.md)  → 최대 20,000자 사용 → 남은 통: 130,000자
2번째 파일 (SOUL.md)    → 최대 20,000자 사용 → 남은 통: 110,000자
3번째 파일 (IDENTITY.md)→ 최대 20,000자 사용 → 남은 통: 90,000자
...
통이 64자 미만이 되면   → 나머지 파일은 건너뜀
```

우선순위가 높은 파일이 먼저 예산을 확보하므로,
**AGENTS.md(우선순위 10)가 가장 안전하고, MEMORY.md(우선순위 70)가 가장 잘릴 위험이 크다.**

### 파일 우선순위

| 순위 | 파일 | 설명 |
|------|------|------|
| 10 | AGENTS.md | 워크스페이스 규칙 (가장 먼저 예산 확보) |
| 20 | SOUL.md | 에이전트 성격 |
| 30 | IDENTITY.md | 에이전트 신원 |
| 40 | USER.md | 사용자 프로필 |
| 50 | TOOLS.md | 도구 메모 |
| 60 | BOOTSTRAP.md | 최초 실행 안내 |
| 70 | MEMORY.md | 장기 기억 (가장 나중에 예산 확보) |
| 동적 | HEARTBEAT.md | 하트비트 체크리스트 (캐시 경계 아래 별도 처리) |

### 설정 변경

```bash
# 파일당 한도를 30,000자로 늘리기
openclaw config set agents.defaults.bootstrapMaxChars 30000

# 전체 한도를 200,000자로 늘리기
openclaw config set agents.defaults.bootstrapTotalMaxChars 200000
```

---

## 2. 잘림 처리

파일이 한도를 넘으면 통째로 버리는 게 아니라, **앞부분과 뒷부분을 살리고 중간을 잘라낸다.**

### 잘리는 방식

```
원본 파일 (30,000자):
[앞쪽 내용 ...................... 중간 내용 ...................... 뒷쪽 내용]

한도 20,000자 적용 후:
[앞쪽 70% = 14,000자] [...잘림 표시...] [뒷쪽 20% = 4,000자]
```

- 앞쪽 70%를 보존 — 보통 파일의 핵심 규칙이 앞에 있으므로
- 뒷쪽 20%를 보존 — 마지막 섹션도 중요할 수 있으므로
- 나머지 10%는 잘림 표시(마커) 공간

### 잘림 표시 예시

잘린 부분에는 이런 마커가 들어간다:

```
[...truncated, read AGENTS.md for full content...]
...(truncated AGENTS.md: kept 14000+4000 chars of 30000)...
```

에이전트는 이 마커를 보고 "이 파일이 잘렸구나, 필요하면 직접 읽어야겠다"고 판단할 수 있다.

### 잘림 경고

잘림이 발생하면 시스템 프롬프트에 경고 메시지가 추가된다:

```
[Bootstrap truncation warning]
Some workspace bootstrap files were truncated before injection.
Treat Project Context as partial and read the relevant files directly if details seem missing.
- AGENTS.md: 30000 raw -> 20000 injected (~33% removed; max/file).
```

경고 모드 3가지:

| 모드 | 동작 |
|------|------|
| `"off"` | 경고 안 함 |
| `"once"` (기본) | 같은 패턴이면 한 번만 경고 |
| `"always"` | 매 턴마다 경고 |

---

## 3. 세션 필터링

모든 세션이 모든 파일을 받는 건 아니다.
세션 종류에 따라 불필요한 파일을 빼준다.

### 왜 필터링하나?

- **서브에이전트**는 독립적인 하위 작업만 수행 → 하트비트나 메모리가 필요 없음
- **Cron 작업**은 예약된 단발 실행 → 마찬가지로 최소한의 지침만 필요
- **하트비트 lightweight**는 주기적 체크 → HEARTBEAT.md만 있으면 됨
- 불필요한 파일을 빼면 **토큰 절약 + 에이전트 집중도 향상**

### 세션별 파일 로딩 표

```
                    메인 세션    서브에이전트    Cron    하트비트(lightweight)
AGENTS.md              O             O          O              X
SOUL.md                O             O          O              X
IDENTITY.md            O             O          O              X
USER.md                O             O          O              X
TOOLS.md               O             O          O              X
BOOTSTRAP.md           O             X          X              X
MEMORY.md              O             X          X              X
HEARTBEAT.md          조건부          X          X              O
```

- **O** = 로드됨
- **X** = 제외됨
- **조건부** = 하트비트 설정이 켜져 있을 때만 포함

### 필터링 3단계

```
[모든 파일]
    |
    v
1단계: 세션 타입 필터
    서브에이전트/Cron이면 → BOOTSTRAP, MEMORY, HEARTBEAT 제거
    |
    v
2단계: 컨텍스트 모드 필터
    lightweight 모드이면 → 하트비트는 HEARTBEAT만, 나머지는 전부 제거
    |
    v
3단계: HEARTBEAT 조건부 필터
    일반 세션인데 하트비트 비활성이면 → HEARTBEAT 제거
    |
    v
[최종 파일 목록] → 예산 적용 → 시스템 프롬프트에 주입
```

---

## 4. 에이전트가 직접 읽는 파일과의 차이

| | 부트스트랩 자동 주입 | 에이전트가 도구로 직접 읽기 |
|---|---|---|
| 시점 | 세션 시작 시 자동 | 대화 중 에이전트 판단으로 |
| 예산 관리 | O (한도 내에서 잘림 처리) | X (컨텍스트 윈도우 직접 소비) |
| 세션 필터링 | O (세션 종류별 자동 제외) | X (에이전트가 알아서 판단) |
| 대화 턴 소비 | X (시스템 프롬프트에 포함) | O (도구 호출 1턴 소비) |
| 지원 파일명 | 9개 고정 | 아무 파일이나 가능 |

부트스트랩에 포함되지 않는 커스텀 파일(예: RULES.md, CONVENTIONS.md)을 쓰고 싶다면,
AGENTS.md의 `## Session Startup` 섹션에 "Read RULES.md" 같은 지시를 추가하면
에이전트가 세션 시작 시 직접 읽게 된다.

---

## 5. 실용적 팁

### 파일이 잘리고 있다면

```bash
# 파일당 한도 늘리기
openclaw config set agents.defaults.bootstrapMaxChars 30000

# 또는 파일 자체를 줄이기 (핵심만 남기고 나머지는 별도 파일로)
```

### 서브에이전트에서도 특정 파일이 필요하다면

서브에이전트는 allowlist(AGENTS, SOUL, IDENTITY, USER, TOOLS)만 받는다.
추가 지침이 필요하면 AGENTS.md에 통합하거나, 훅(`bootstrap-extra-files`)을 사용한다.

### AGENTS.md가 가장 중요한 이유

1. 우선순위 10번 — 예산을 가장 먼저 확보
2. 모든 세션 타입에서 로드됨 (lightweight 제외)
3. 컨텍스트 압축 시에도 `Session Startup`과 `Red Lines` 섹션이 보존됨
4. 사실상 유일하게 "절대 빠지지 않는" 지침 파일

---

## 관련 소스 코드

| 역할 | 파일 경로 |
|------|-----------|
| 예산 적용 & 잘림 | `src/agents/pi-embedded-helpers/bootstrap.ts` |
| 예산 분석 & 경고 | `src/agents/bootstrap-budget.ts` |
| 세션 필터링 | `src/agents/bootstrap-files.ts` |
| 파일 로딩 & allowlist | `src/agents/workspace.ts` |
| 시스템 프롬프트 주입 | `src/agents/system-prompt.ts` |
| 압축 시 보존 | `src/agents/pi-hooks/compaction-safeguard.ts` |
| 압축 후 재주입 | `src/auto-reply/reply/post-compaction-context.ts` |
