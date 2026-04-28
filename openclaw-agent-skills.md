# OpenClaw Agent 스킬 관리 정리

## 1. 스킬 로딩 — 6개 소스 머지

`src/agents/skills/workspace.ts:502-546`의 `loadSkillEntries(workspaceDir, opts)`가 6개 디렉토리에서 스킬을 로드해 머지합니다.

| 우선순위 | 소스 | 경로 |
|---|---|---|
| 1 (최저) | extra | `config.skills.load.extraDirs` + 플러그인 디렉토리 |
| 2 | openclaw-bundled | 호스트 패키지 번들 (`resolveBundledSkillsDir()`) |
| 3 | openclaw-managed | `~/.openclaw/skills` (`CONFIG_DIR/skills`) |
| 4 | agents-skills-personal | `~/.agents/skills` |
| 5 | agents-skills-project | `<workspaceDir>/.agents/skills` |
| 6 (최고) | openclaw-workspace | `<workspaceDir>/skills` |

같은 이름 충돌 시 **높은 우선순위가 덮어씀** (`workspace.ts:548-567`). 머지 후 `filterSkillEntries`가 두 단계로 거름:

- `shouldIncludeSkill` — frontmatter `os`/`always`/`requires` 등 런타임 eligibility
- `resolveEffectiveAgentSkillFilter` — agent 단위 allow-list

> 참고: `~/.openclaw/workspace/skills/...` 같은 경로는 코드에 **존재하지 않음**.

---

## 2. allow-list 방식 (`agents.list[].skills`)

### 동작 (`src/agents/skills/agent-filter.ts:9-24`)

`resolveEffectiveAgentSkillFilter(cfg, agentId)`:

1. `agentId`를 정규화 → `cfg.agents.list`에서 매칭 검색
2. 엔트리에 `skills` 키가 **명시적으로 존재**하면 그 배열로 **완전 대체** (defaults와 머지 안 함)
3. 없으면 `cfg.agents.defaults.skills` 폴백
4. 둘 다 undefined → allow-list 비활성 (모두 통과)

### 설정 예

```yaml
agents:
  defaults:
    skills: []
  list:
    - id: advertiser-60
      skills: [skill-a, skill-b]   # 이 agent 전체 셋
```

### 단점

- **머지 불가, 완전 대체** — defaults에 50개 있어도 advertiser-60에 1개 추가하려면 51개 다 다시 적어야 함
- "defaults + 한두 개 추가/제외" 같은 패치형 설정이 안 됨
- agent 수가 늘수록 config가 비대해지고 동기화 부담

---

## 3. workspace-{agent_id} 디렉토리 격리 방식 (추천)

### 원리

- `loadSkillEntries`의 `workspaceDir` 인자는 호출자가 결정
- agent마다 다른 workspaceDir이 넘어가면 자연스럽게 격리됨
- `<workspaceDir>/skills`는 6개 소스 중 **최고 우선순위**

### 실제 패턴 (`extensions/feishu/src/dynamic-agent.ts:80-103`)

```
workspaceTemplate ?? "~/.openclaw/workspace-{agentId}"
```

agent마다 `~/.openclaw/workspace-<agentId>/` 자동 생성, config의 `agents.list[].workspace`에 등록. mattermost / telegram / whatsapp 모두 동일 규약.

### 설정 — allow-list 안 쓰고 격리만 사용

```yaml
agents:
  defaults: {}                                      # skills 키 자체를 안 둠
  list:
    - id: advertiser-60
      workspace: ~/.openclaw/workspace-advertiser-60
      # skills 키 없음 → allow-list 비활성
```

스킬 파일은 `~/.openclaw/workspace-advertiser-60/skills/my-skill/SKILL.md` 에 두면 그 agent에만 노출.

### 장점

- **풀 리스트 다시 적을 필요 없음** — allow-list 단점 회피
- 디렉토리 = 권한 경계라 직관적
- agent별 추가/제거가 파일 이동만으로 가능

### 주의

- `agents.list[].skills` 또는 `defaults.skills`가 **정의되어 있으면** allow-list가 활성화되어, 격리한 스킬도 frontmatter `name`이 allow-list에 없으면 거름
- 즉 격리만으로 운영하려면 두 곳 모두 `skills` 키를 두지 말 것
