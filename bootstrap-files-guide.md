# OpenClaw Bootstrap Files Guide

OpenClaw 에이전트가 세션 시작 시 로드하는 지침 파일 목록, 로드 시점, 조건, 설정에 대한 정리.

---

## 부트스트랩 파일 목록 & 역할

| 파일 | 역할 | 시스템 프롬프트 우선순위 |
|------|------|--------------------------|
| `AGENTS.md` | 워크스페이스 규칙, 세션 절차, 안전 규칙 | 10 (최우선) |
| `SOUL.md` | 에이전트 성격/영혼 | 20 |
| `IDENTITY.md` | 에이전트 신원 (이름, 역할 등) | 30 |
| `USER.md` | 사용자 프로필 | 40 |
| `TOOLS.md` | 도구/스킬 사용 메모 | 50 |
| `BOOTSTRAP.md` | 최초 실행 안내 (1회용) | 60 |
| `MEMORY.md` / `memory.md` | 장기 기억 저장소 | 70 |
| `HEARTBEAT.md` | 하트비트 체크리스트 | 동적 (캐시 경계 아래) |

- 우선순위는 `src/agents/system-prompt.ts`의 `CONTEXT_FILE_ORDER` 맵에서 정의
- 우선순위가 낮을수록 시스템 프롬프트에서 앞쪽에 배치

---

## 템플릿 파일 위치

- 일반 템플릿: `docs/reference/templates/*.md`
- Dev 모드 템플릿 (C-3PO): `docs/reference/templates/*.dev.md`
- 템플릿 resolve: `src/agents/workspace-templates.ts`의 `resolveWorkspaceTemplateDir()`
- 워크스페이스 생성 시 `writeFileIfMissing()`으로 파일이 없을 때만 작성 (flag `wx`)

---

## 로드 흐름

```
1. loadWorkspaceBootstrapFiles()          -- 워크스페이스에서 파일 읽기
       |
2. filterBootstrapFilesForSession()       -- 세션 타입별 필터링
       |
3. applyContextModeFilter()               -- lightweight/full 모드 필터
       |
4. applyBootstrapHookOverrides()          -- 훅으로 런타임 수정 (extra files 포함)
       |
5. filterHeartbeatBootstrapFile()         -- HEARTBEAT.md 조건부 제외
       |
6. buildBootstrapContextFiles()           -- 예산 적용 & 잘라내기
       |
7. sortContextFilesForPrompt()            -- 우선순위 정렬
       |
8. 시스템 프롬프트에 주입                   -- 안정 파일은 캐시 경계 위, HEARTBEAT는 아래
```

### 관련 소스 파일

| 단계 | 파일 |
|------|------|
| 파일 읽기 | `src/agents/workspace.ts` (loadWorkspaceBootstrapFiles) |
| 세션 필터 | `src/agents/workspace.ts` (filterBootstrapFilesForSession) |
| 컨텍스트 모드 | `src/agents/bootstrap-files.ts` (applyContextModeFilter) |
| 훅 적용 | `src/agents/bootstrap-hooks.ts` (applyBootstrapHookOverrides) |
| 예산/잘림 | `src/agents/bootstrap-budget.ts` (analyzeBootstrapBudget) |
| 정렬 | `src/agents/system-prompt.ts` (sortContextFilesForPrompt) |
| 전체 오케스트레이션 | `src/agents/bootstrap-files.ts` (resolveBootstrapContextForRun) |

---

## 세션 타입별 로딩 범위

| 세션 타입 | 로드되는 파일 |
|-----------|-------------|
| **메인 세션** (일반 대화) | 전체 (AGENTS, SOUL, IDENTITY, USER, TOOLS, BOOTSTRAP, MEMORY, HEARTBEAT) |
| **서브에이전트** | AGENTS, SOUL, IDENTITY, USER, TOOLS 만 |
| **Cron** | AGENTS, SOUL, IDENTITY, USER, TOOLS 만 |
| **하트비트 (lightweight)** | HEARTBEAT.md 만 |

- 필터링 allowlist: `src/agents/workspace.ts`의 `MINIMAL_BOOTSTRAP_ALLOWLIST` (line 547-553)
- 세션 키 판별: `src/routing/session-key.ts`의 `isSubagentSessionKey()`, `isCronSessionKey()`

---

## 컨텍스트 압축 시 보존 메커니즘

컨텍스트 윈도우가 가득 차서 압축될 때, AGENTS.md의 핵심 섹션이 두 가지 경로로 보존된다.

### 1. Compaction Safeguard

- 파일: `src/agents/pi-hooks/compaction-safeguard.ts` (line 673-722)
- 추출 섹션: `## Session Startup`, `## Red Lines`
- 레거시 호환: `## Every Session`, `## Safety`
- `<workspace-critical-rules>` 태그로 감싸서 압축 요약 꼬리에 부착
- 최대 2,000자
- 꼬리 우선 보존: 요약 본문보다 워크스페이스 규칙이 잘림에서 살아남음

### 2. Post-Compaction Injection

- 파일: `src/auto-reply/reply/post-compaction-context.ts`
- 압축 완료 후 AGENTS.md에서 지정 섹션을 다시 추출하여 주입
- 기본 대상: `["Session Startup", "Red Lines"]`
- 최대 3,000자
- 설정: `agents.defaults.compaction.postCompactionSections`
- 빈 배열 `[]`이면 비활성화
- `YYYY-MM-DD` 플레이스홀더를 실제 날짜로 치환

---

## BOOTSTRAP.md 특수 로직

`src/agents/workspace.ts` (line 398-446)에서 관리되는 1회용 파일.

### 생성 조건

- 워크스페이스가 완전히 새로운 경우에만 생성
- IDENTITY.md, USER.md가 수정되지 않았고
- `memory/`, `.git`, `MEMORY.md` 등 사용자 콘텐츠 흔적이 없을 때

### 수명 주기

1. 워크스페이스 최초 생성 시 BOOTSTRAP.md 작성
2. `.openclaw/workspace-state.json`에 `bootstrapSeededAt` 타임스탬프 기록
3. 에이전트가 BOOTSTRAP.md를 읽고 설정 완료 후 삭제
4. 다음 세션에서 BOOTSTRAP.md 부재 감지 -> `setupCompletedAt` 기록
5. 이후 BOOTSTRAP.md는 다시 생성되지 않음

### 레거시 마이그레이션

- IDENTITY.md나 USER.md가 템플릿과 다르거나, 사용자 콘텐츠 인디케이터 존재 시
- `setupCompletedAt`를 바로 기록하고 BOOTSTRAP.md 생성 건너뜀

---

## 예산 및 잘림 (Truncation)

파일: `src/agents/bootstrap-budget.ts`

### 기본값

| 파라미터 | 기본값 | 설명 |
|----------|--------|------|
| `DEFAULT_BOOTSTRAP_MAX_CHARS` | 20,000 | 파일당 최대 문자 수 |
| `DEFAULT_BOOTSTRAP_TOTAL_MAX_CHARS` | 150,000 | 전체 부트스트랩 총 문자 수 |
| `DEFAULT_BOOTSTRAP_NEAR_LIMIT_RATIO` | 0.85 | 근접 경고 비율 |
| `MIN_BOOTSTRAP_FILE_BUDGET_CHARS` | 64 | 이하면 파일 건너뜀 |

### 잘림 전략

- Head-to-tail 방식: 앞 70% + 뒤 20% 유지
- 중간에 `[...truncated, read <파일명> for full content...]` 마커 삽입
- 파일별 -> 전체 순서로 예산 적용

### 잘림 경고

- 모드: `"off"` / `"once"` (기본) / `"always"`
- 잘림 시그니처로 중복 경고 방지
- 설정: `agents.defaults.bootstrapPromptTruncationWarning`

---

## 주요 설정 값

| 설정 경로 | 기본값 | 설명 |
|-----------|--------|------|
| `agents.defaults.contextInjection` | `"always"` | `"continuation-skip"`이면 연속 턴에서 재주입 생략 |
| `agents.defaults.skipBootstrap` | `false` | `true`면 부트스트랩 파일 생성 건너뜀 |
| `agents.defaults.bootstrapMaxChars` | 20,000 | 파일당 최대 문자 수 |
| `agents.defaults.bootstrapTotalMaxChars` | 150,000 | 전체 총 문자 수 |
| `agents.defaults.bootstrapPromptTruncationWarning` | `"once"` | 잘림 경고 모드 |
| `agents.defaults.compaction.postCompactionSections` | `["Session Startup", "Red Lines"]` | 압축 후 재주입 섹션 |
| `agents.defaults.heartbeat.includeSystemPromptSection` | `true` | HEARTBEAT 가이드 포함 여부 |
| `agents.defaults.llm.idleTimeoutSeconds` | 60 | LLM 스트림 idle 타임아웃 (초) |

설정 변경:

```bash
openclaw config set agents.defaults.bootstrapMaxChars 30000
openclaw config set agents.defaults.contextInjection continuation-skip
openclaw config set agents.defaults.compaction.postCompactionSections '[]'
```

---

## Extra Bootstrap Files (훅)

파일: `src/hooks/bundled/bootstrap-extra-files/handler.ts`

- 훅 이벤트: `agent:bootstrap`
- 설정: `hooks["bootstrap-extra-files"]`의 `paths`, `patterns`, `files` 배열
- glob 패턴 지원 (`*`, `?`, `{...}`)
- 허용 파일명: 인식된 부트스트랩 basename만 (AGENTS.md, SOUL.md 등)
- 보안: `openBoundaryFile()`로 워크스페이스 루트 밖 접근 차단
- 세션 필터링, 예산 적용 모두 동일하게 적용

---

## Continuation Skip 최적화

파일: `src/agents/pi-embedded-runner/run/attempt.ts` (line 453-475)

- `contextInjection: "continuation-skip"` 설정 시
- 이전 턴에서 부트스트랩 완료 마커 (`FULL_BOOTSTRAP_COMPLETED_CUSTOM_TYPE`) 기록
- 다음 턴에서 해당 마커 확인 후 부트스트랩 재주입 생략
- 토큰 절약 효과

---

## MEMORY.md 케이스 처리

파일: `src/agents/workspace.ts` (line 465-483)

- `MEMORY.md` 우선, 없을 때만 `memory.md` 폴백
- macOS Docker 등 대소문자 무시 파일시스템에서 두 파일 모두 존재하는 것으로 보일 수 있음
- 이중 주입 방지를 위해 하나만 로드
