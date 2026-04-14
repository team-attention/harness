---
name: check-harness
description: |
  Harness 성숙도 진단 — 4축 22항목 체크리스트로 **Overall(유저 전역)** 과
  **Per-Project(프로젝트)** 스코프를 분리 평가한다.
  Overall: 설치된 스킬·플러그인 정리 상태 + 전체 세션의 실행 패턴.
  Per-Project: CLAUDE.md 품질·경계 + 프로젝트 자동화·검증.
  4개의 서브에이전트(skill-portfolio-analyzer, session-pattern-analyzer,
  context-quality-reviewer, project-automation-auditor)를 병렬 실행하여
  스코어카드 + Missing/Broken/Opportunity Findings + Quick Wins 리포트를 만든다.

  Use whenever the user asks to audit their Claude Code harness, review skill
  portfolio health, evaluate execution patterns across sessions, check project
  context/rules quality, or wants to know what's missing in their AI setup —
  even if they don't say "check-harness" explicitly.

  Trigger: "/check-harness", "check harness", "하네스 체크", "하네스 점검",
  "harness audit", "설정 점검", "뭐가 부족한지 봐줘", "하네스 진단",
  "성숙도 점검", "maturity check", "내 클로드 설정 봐줘", "스킬 정리".
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
  - Agent
  - AskUserQuestion
---

# /check-harness — Harness 성숙도 진단 (v2)

4축 22항목 체크리스트를 **Overall + Per-Project** 두 스코프로 나눠 평가한다.
체크리스트 원본: `references/checklist.md` (반드시 먼저 읽어서 항목 정의 확인).

**산출물 2가지:**
1. **스코어카드** — 4축별 PASS/WEAK_PASS/FAIL/N/A + Overall/Project 성숙도 레벨
2. **액션 리포트** — Findings(Missing/Broken/Opportunity) + Quick Wins Top 3

리포트 저장 위치: `.harness/check-reports/check-harness-{YYYY-MM-DD}-{scope}.md`

---

## Phase 0 — Scope 결정

사용자 입력에 스코프가 명시됐으면 그대로 사용:
- `overall` / `전체` / `유저` → Overall만
- `project` / `프로젝트` → Per-Project만
- `all` / `둘 다` / 지정 없음 → **Both**

스코프가 불명확하면 `AskUserQuestion`:

```
question: "어디까지 진단할까요?"
options:
  - Both (Overall + Per-Project) — 전체 진단 (Recommended)
  - Overall만 (유저 전역 · 스킬 정리 + 세션 패턴)
  - Per-Project만 (현재 프로젝트 맥락·자동화)
```

선택된 스코프에 따라 Phase 1에서 띄울 서브에이전트가 결정된다.

### 프로젝트 루트 탐색 (Per-Project 스코프일 때)

cwd에서 상위로 올라가며 `.claude` 또는 `CLAUDE.md`가 있는 디렉토리를 찾아 `PROJECT_ROOT`로 삼는다. 못 찾으면 Per-Project 스코프는 비활성화하고 사용자에게 경고.

### 캐시 디렉토리

```bash
mkdir -p /tmp/cc-cache/check-harness/
```

서브에이전트 산출물은 `/tmp/cc-cache/check-harness/{PORTFOLIO,SESSION,CONTEXT,AUTOMATION}.json` 에 떨어진다.

---

## Phase 1 — 병렬 데이터 수집

**선택된 스코프에 해당하는 서브에이전트를 모두 같은 메시지에서 병렬로 호출**한다.
이게 중요 — 순차로 띄우면 의미가 없다.

### Overall 스코프 (2개)

```
Agent(
  subagent_type: "skill-portfolio-analyzer",
  description: "Skill portfolio health scan",
  prompt: """
    설치된 모든 스킬/플러그인과 ~/.claude.json의 skillUsage를 교차 분석.
    SKILL.md 정의된 JSON 스키마로만 응답.
    결과를 /tmp/cc-cache/check-harness/PORTFOLIO.json 에 저장하고
    그 경로를 마지막 줄에 출력해라.
    dead_days=90, low_value_days=60, low_value_count=3
  """
)

Agent(
  subagent_type: "session-pattern-analyzer",
  description: "Session execution pattern scan",
  prompt: """
    scope=overall, days=7, long_session_min=20.
    ~/.claude/projects/ 전체 세션의 tool_use 메타데이터만 분석.
    프롬프트 텍스트는 절대 읽지 마라.
    결과 JSON을 /tmp/cc-cache/check-harness/SESSION.json 에 저장하고
    경로를 마지막 줄에 출력.
  """
)
```

### Per-Project 스코프 (2개)

```
Agent(
  subagent_type: "context-quality-reviewer",
  description: "Project context quality review",
  prompt: """
    project_root={PROJECT_ROOT}
    CLAUDE.md + .claude/rules/* + .gitignore + settings.json + MCP 설정을
    읽고 CONTEXT_REPORT 스키마로 평가.
    /tmp/cc-cache/check-harness/CONTEXT.json 에 저장.
  """
)

Agent(
  subagent_type: "project-automation-auditor",
  description: "Project automation & verification audit",
  prompt: """
    project_root={PROJECT_ROOT}
    session_report=/tmp/cc-cache/check-harness/SESSION.json  (있으면 참조)
    test/hooks/verifier/isolation 감사 후 AUTOMATION_REPORT 스키마로 저장.
    /tmp/cc-cache/check-harness/AUTOMATION.json
  """
)
```

**Both 스코프면 4개를 한 번에 띄운다.** session-pattern-analyzer 결과를 project-automation-auditor가 참조하지만, 전자가 아직 없어도 후자는 session_report 없이 동작하도록 설계됨 — 병렬 실행 가능.

모든 서브에이전트 완료 후 JSON 4개를 Read로 읽어 메모리에 `PORTFOLIO`, `SESSION`, `CONTEXT`, `AUTOMATION` 으로 보관.

---

## Phase 2 — 체크리스트 판정

`references/checklist.md`의 22항목 각각에 대해:

```
for item in checklist:
  status = evaluate(item, PORTFOLIO | SESSION | CONTEXT | AUTOMATION)
  # status ∈ {PASS, WEAK_PASS, FAIL, N/A}
  record(item.id, status, evidence)
```

### 평가 로직 요약

| 축 | 참조 리포트 | 판정 방식 |
|---|---|---|
| **A. Portfolio** (5) | PORTFOLIO | summary 필드 직접 매핑 |
| **B. Execution** (6) | SESSION | metrics 필드 임계치 비교 |
| **C. Context** (6) | CONTEXT | claude_md.quality + rules + sensitive_protection |
| **D. Automation** (5) | AUTOMATION | 각 boolean + project_skills.usage_rate |

### 상태 규칙

- **PASS** — 증거 명확
- **WEAK_PASS** — 조건은 충족하나 리포트의 `weak_pass_flags`에 해당 필드가 있음
- **FAIL** — 증거 없음 또는 명시적 실패
- **N/A** — 리포트에 해당 데이터 없음(예: 세션 0개) 또는 프로젝트 성격상 무관

모든 판정에 **증거 문자열**을 함께 기록 (리포트에 인용).

### 성숙도 계산

- 축 레벨: L1 항목 전체 PASS/WEAK_PASS → L1 달성, L2도 모두 → L2 달성, L3도 모두 → L3
- Overall Maturity: A, B 중 낮은 레벨
- Project Maturity: C, D 중 낮은 레벨
- 스코어: PASS=1, WEAK_PASS=0.5, FAIL=0, N/A=제외. 축별 통과율 = 합 / (전체 - N/A)

---

## Phase 3 — 리포트 작성

`.harness/check-reports/check-harness-{YYYY-MM-DD}-{scope}.md` 경로에 아래 포맷으로 Write.

```markdown
═══════════════════════════════════════════
         HARNESS MATURITY REPORT (v2)
═══════════════════════════════════════════

Scope:        {overall|project|all}
Project:      {name or "-"}
Path:         {PROJECT_ROOT or "-"}
Scanned:      {YYYY-MM-DD HH:MM}
Sessions:     {N} sessions ({period})

───────────────────────────────────────────
  SCORECARD
───────────────────────────────────────────

Overall Maturity:  L{n}   (axes A, B)
Project Maturity:  L{n}   (axes C, D)

| Axis                          | Pass | Total | Score | Level |
|-------------------------------|-----:|------:|-------|-------|
| A. Portfolio Hygiene          |  x   |   5   | xx%   | L{n}  |
| B. Execution Patterns         |  x   |   6   | xx%   | L{n}  |
| C. Context & Boundaries       |  x   |   6   | xx%   | L{n}  |
| D. Automation & Verification  |  x   |   5   | xx%   | L{n}  |

───────────────────────────────────────────
  DETAILED CHECKLIST
───────────────────────────────────────────

### A. Portfolio Hygiene  (Overall)

| ID | L  | Item                         | Status       | Evidence |
|----|----|------------------------------|--------------|----------|
| A1 | L1 | 70%+ skills used last 30d    | PASS         | used 92/131 (70.2%) |
| A2 | L2 | No dead skills               | FAIL         | 18 dead (e.g. `xxx`, ...) |
| ... |   |                              |              |          |

### B. Execution Patterns  (Overall)
...

### C. Context & Boundaries  (Per-Project)
...

### D. Automation & Verification  (Per-Project)
...

───────────────────────────────────────────
  FINDINGS ({total}건)
───────────────────────────────────────────

### High Priority
[Missing]  CLAUDE.md에 dev 명령어 누락
  → Action: "Development" 섹션에 install/test/run 추가

### Medium Priority
[Broken]   등록된 `scaffold` 스킬이 지난 30일간 호출 0회
  → Action: 트리거 키워드 재점검 또는 스킬 제거

### Low Priority
[Opportunity]  PostToolUse 포맷터 훅 미설정
  → Action: `.claude/settings.json`에 prettier/ruff 훅 추가

───────────────────────────────────────────
  QUICK WINS (Top 3)
───────────────────────────────────────────

1. 🟢 [Low]     {action} — 예상 효과: {benefit}
2. 🟡 [Medium]  {action} — ...
3. 🟡 [Medium]  {action} — ...

───────────────────────────────────────────
  SKILL PORTFOLIO HEALTH
───────────────────────────────────────────

Total installed: {N}   |   Used last 30d: {X} ({%})

| Category            | Count | Top examples |
|---------------------|------:|--------------|
| Dead (>90d unused)  |   N   | ...          |
| Ghost entries       |   N   | ...          |
| Duplicate clusters  |   N   | ...          |
| Prefix duplicates   |   N   | ...          |
| Trigger collisions  |   N   | ...          |

### Recommended Cleanups
- [Remove] {skill} — 0회, 설치 45일
- [Consolidate] "code review" 클러스터 → {survivor} 유지
- [Rename/Dedupe] `dev-scan` vs `dev:dev-scan`

───────────────────────────────────────────
  EXECUTION PATTERN HIGHLIGHTS
───────────────────────────────────────────

- Plan-first ratio:      {X}%
- Delegation ratio:      {X}%
- Parallel invocations:  {N}
- Handoff ratio:         {X}% (장시간 세션 중)
- Completion check:      {X}%
- Top 3-gram share:      {X}%  {⚠️ if > 30%}

### Automation Opportunities (반복 패턴)
- `Read → Edit → Bash(npm test)` — 18회 반복 → 스킬 후보
- ...
```

---

## Hard Rules

1. **프롬프트 내용 읽지 않는다** — session-pattern-analyzer는 tool_use 메타데이터만
2. **프로젝트 파일 수정 금지** — 리포트만 `.harness/check-reports/`에 작성
3. **증거 기반 평가** — 모든 상태에 근거 문자열 필수
4. **맥락 인식** — 프로젝트 무관 항목은 N/A (예: 프론트엔드 없는 프로젝트의 UI 훅)
5. **병렬 실행** — Phase 1 서브에이전트는 반드시 같은 메시지에서 동시 호출
6. **캐시 재사용** — `/tmp/cc-cache/check-harness/` JSON 4개는 후속 분석에서 재사용 가능
