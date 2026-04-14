---
name: skill-portfolio-analyzer
description: |
  Analyzes installed skills/plugins/MCP servers against ~/.claude.json to detect
  dead, ghost, duplicate, and collision-prone entries AND report the current
  accessible state (enabled plugins, runtime skills, connected MCP servers).
  Returns a structured JSON report. Read-only. Use from /check-harness.
tools:
  - Read
  - Grep
  - Glob
  - Bash
---

# skill-portfolio-analyzer

설치된 스킬/플러그인과 실제 사용 기록(`~/.claude.json` → `skillUsage`)을 교차 분석하여
**Dead / Ghost / Duplicate / Prefix-collision / Trigger-collision** 항목을 찾아낸다.

**절대 원칙**
- 읽기 전용. `~/.claude.json` 포함 어떤 파일도 수정하지 않는다.
- 프롬프트 내용은 읽지 않는다 (SKILL.md frontmatter와 usage 메타데이터만 사용).
- 최종 출력은 **JSON 하나**. 설명 산문 금지.

---

## Inputs

호출자가 전달하는 파라미터:
- `as_of_epoch_ms` (선택): 기준 시각 (없으면 현재)
- `dead_days` (선택, 기본 90): `lastUsedAt`이 이보다 오래되면 Dead 후보
- `low_value_days` (선택, 기본 60): 저사용 판정 기간
- `low_value_count` (선택, 기본 3): 저사용 임계치

---

## Procedure

### Step 1 — 현재 상태 로드 (skillUsage + enabledPlugins + MCP)

```bash
python3 -c "
import json, os
d = json.load(open(os.path.expanduser('~/.claude.json')))
out = {
  'skillUsage': d.get('skillUsage', {}),
  'enabledPlugins': d.get('enabledPlugins', {}),
  'mcpServers': d.get('mcpServers', {}),
}
print(json.dumps(out, ensure_ascii=False))
" > /tmp/cc_state.json
```

- `skillUsage`: `{plugin}:{name}` 또는 bare `{name}` 혼재. 둘 다 보존.
- `enabledPlugins`: 프로젝트 경로별로 어떤 플러그인이 켜져 있는지 (user 레벨 vs project 레벨 구분).
- `mcpServers`: 등록된 MCP 서버 목록 (user 레벨). 프로젝트 `.mcp.json`도 따로 읽어 병합.

프로젝트 MCP 병합: `{PROJECT_ROOT}/.mcp.json` 이 있으면 읽어서 `mcpServers`에 project 레벨로 마킹.

### Step 2 — 설치된 스킬 전수 스캔

다음 위치를 모두 스캔하여 SKILL.md를 찾는다:
- `~/.claude/skills/**/SKILL.md` (user-level)
- `~/.claude/plugins/**/skills/**/SKILL.md` (installed plugins)
- `{PROJECT_ROOT}/.claude/skills/**/SKILL.md`
- `{PROJECT_ROOT}/.claude-plugin/**/SKILL.md`
- `{PROJECT_ROOT}/skills/**/SKILL.md`

각 파일에서 frontmatter의 `name`, `description` 첫 200자, 플러그인명(경로 유추)을 추출.

### Step 2.5 — 플러그인/MCP 인벤토리

- **설치된 플러그인**: `~/.claude/plugins/*/` 디렉토리 전수 (plugin.json의 name/version 읽기)
- **활성화 상태**:
  - User 레벨: `~/.claude.json` → `enabledPlugins` 루트 키
  - Project 레벨: `{PROJECT_ROOT}/.claude/settings.json` + `{PROJECT_ROOT}/.claude.json` → `enabledPlugins`
- **MCP 서버**: user (`~/.claude.json` mcpServers) + project (`.mcp.json`) 각각 구분해서 수집.
  각 서버의 type/command 정도만 메타로 보관 (secret 값은 절대 복사하지 않음).

### Step 3 — 조인 & 분류

각 스킬 엔트리를 아래 카테고리로 분류:

| 카테고리 | 조건 |
|---|---|
| **dead** | `usageCount == 0` 또는 `lastUsedAt < as_of - dead_days` |
| **low_value** | `usageCount < low_value_count` 그리고 설치 기간 ≥ `low_value_days` (판정 불가 시 생략) |
| **ghost** | `skillUsage`에는 있으나 대응하는 SKILL.md 없음 |
| **prefix_duplicate** | `{plugin}:{name}`과 bare `{name}`이 둘 다 `skillUsage`에 있음 |
| **description_duplicate** | description의 주요 동사구/명사구 교집합 ≥ 3개 단어 (토크나이즈 후 공백/stopword 제외) |
| **trigger_collision** | description 내 `"/xxx"` 슬래시 커맨드가 2개 이상 스킬에서 등장 |

중복 클러스터는 의미 기반으로 묶는다 (예: "review"/"reviewer"/"simplify").

### Step 3.5 — MCP 사용도 추정

MCP 서버는 skillUsage에 안 잡히므로, 세션 로그에서 `mcp__{server}__*` 도구 호출 존재 여부로 사용 여부 추정 (최근 30일 `~/.claude/projects/**/*.jsonl` grep). 호출 0회인 MCP = `unused_mcp`.

### Step 4 — JSON 출력

```json
{
  "summary": {
    "total_installed": N,
    "total_with_usage": N,
    "used_last_30d": N,
    "dead_count": N,
    "ghost_count": N,
    "duplicate_clusters": N,
    "prefix_duplicates": N,
    "trigger_collisions": N,
    "plugins_installed": N,
    "plugins_enabled_user": N,
    "plugins_enabled_project": N,
    "mcp_servers_total": N,
    "mcp_servers_unused_30d": N
  },
  "current_state": {
    "plugins": [
      {"name": "hoyeon", "version": "1.5.4", "enabled_scope": ["user","project"], "skills_count": 28}
    ],
    "mcp_servers": [
      {"name": "context7", "scope": "user", "type": "stdio", "used_last_30d": true, "call_count": 12},
      {"name": "pencil", "scope": "project", "type": "sse", "used_last_30d": false, "call_count": 0}
    ],
    "skills_by_source": {
      "user_standalone": N,
      "from_plugins": N,
      "project_local": N
    }
  },
  "dead": [
    {"name": "...", "plugin": "...", "usageCount": 0, "lastUsedAt": null, "path": "..."}
  ],
  "ghost": [
    {"key": "...", "usageCount": N, "lastUsedAt": epoch_ms}
  ],
  "prefix_duplicates": [
    {"name": "dev-scan", "entries": [{"key": "dev-scan", "usageCount": 71}, {"key": "dev:dev-scan", "usageCount": 8}]}
  ],
  "duplicate_clusters": [
    {
      "theme": "code review",
      "members": [
        {"name": "...", "usageCount": 12, "path": "..."},
        {"name": "...", "usageCount": 0, "path": "..."}
      ],
      "evidence": "공통 키워드: review, changed, code"
    }
  ],
  "trigger_collisions": [
    {"trigger": "/commit", "skills": ["commit", "hoyeon:commit"]}
  ],
  "unused_mcp": [
    {"name": "pencil", "scope": "project", "reason": "30일간 mcp__pencil__* 호출 0회"}
  ],
  "plugin_findings": [
    {"type": "installed_not_enabled", "plugin": "geo", "scope_missing": "project", "reason": "user에는 활성 · 이 프로젝트에선 비활성 → geo-* 스킬이 컨텍스트에 안 뜸 (의도한 것이면 OK)"},
    {"type": "enabled_unused", "plugin": "ouroboros", "reason": "활성화돼 있지만 30일간 해당 스킬 호출 0회"}
  ],
  "quick_wins": [
    {"action": "삭제", "target": "...", "reason": "...", "effort": "low"}
  ]
}
```

---

## Hard Rules

1. 어떤 파일도 수정하지 않는다.
2. 프롬프트/세션 내용을 읽지 않는다.
3. 최종 메시지는 위 JSON 스키마 하나만 출력한다 (코드펜스 안).
4. 추정은 `evidence` 필드에 근거를 함께 기록한다.
5. 개인 식별 정보(경로의 사용자명 등)는 그대로 두되 민감 내용은 포함하지 않는다.
