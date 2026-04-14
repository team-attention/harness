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
2. **액션 리포트** — Findings + 제안 액션 (TL;DR 상단)

**출력 방식: 대화에 직접 출력 + 파일 저장 (둘 다)**
- 같은 내용을 대화창에 그대로 출력 (사용자가 즉시 확인)
- 동시에 `.harness/check-reports/check-harness-{YYYY-MM-DD}-{scope}.md` 에 Write
  (다음 진단과 비교용 기록)
- 저장 경로는 리포트 끝 한 줄로만 안내: `📁 Saved: {path}`

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

**레벨 (축별)**
- L1 항목이 전부 PASS/WEAK_PASS → L1 달성
- L2도 모두 충족 → L2, L3도 모두 → L3
- Overall Maturity: A, B 중 낮은 레벨 / Project Maturity: C, D 중 낮은 레벨

**점수 (축별, 0–100)**
- 항목 점수: PASS=1, WEAK_PASS=0.5, FAIL=0, N/A=제외
- **가중치**: L1 항목 × 3, L2 × 2, L3 × 1 (기초가 더 중요)
- 축 점수 = Σ(항목점수 × 가중치) / Σ(가중치, N/A 제외) × 100

**Harness Score (종합)**
- Overall Score = (A + B) / 2
- Project Score = (C + D) / 2
- **Harness Score = (Overall + Project) / 2** (스코프가 Both일 때)
- 스코프가 한쪽만이면 해당 스코프 점수를 그대로 사용
- 등급: 90+ Excellent / 75+ Good / 60+ Fair / <60 Needs Work

**다음 레벨 진행률 (축별)**
- 현재 레벨이 L{n}일 때 → `progress = 현재 레벨(n)의 통과 항목 수 / L{n+1} 요구 항목 수`
- 예: L2 달성했고 L3 6항목 중 4개 PASS → "80% → L3"
- L3 달성 시 "L3 완료 🎉" 표시

---

## Phase 2.5 — TL;DR & 액션 합성

Phase 2 판정 결과를 입력으로, 리포트 상단에 박제할 **핵심 요약과 제안 액션**을 먼저 만든다.
이 단계 없이 바로 Phase 3로 가면 사용자가 표만 보고 스스로 우선순위를 짜야 한다.

### 생성할 4가지 변수

1. **`headline`** (1문장) — 두 성숙도 레벨 + "가장 큰 문제" + "가장 쉬운 시작점".
   예: *"Overall L2 / Project L1 — 스킬 포트폴리오에 미사용 18개가 쌓여있고, CLAUDE.md에 실행 명령이 빠져있어요. `xxx` 3개 제거부터 시작해보세요."*

2. **`strength`** (1문장) — FAIL이 적거나 PASS가 많은 축에서 뽑은 **잘 되고 있는 것** 한 줄.

3. **`weakness`** (1문장) — FAIL이 집중된 축에서 뽑은 **가장 개선이 필요한 것** 한 줄.

4. **`actions`** (3–7개) — Findings 전체에서 아래 기준으로 3계층 분류:
   - 🟢 **바로 해볼 만한 것**: 단일 커맨드/한 줄 수정으로 끝나는 것 (스킬 제거, 명령어 추가 등)
   - 🟡 **한 번 정리하면 좋을 것**: 파일 하나 편집 수준 (CLAUDE.md 섹션 추가, 훅 한 개 등록)
   - 🔴 **시간 내서 다뤄볼 것**: 구조 변경 필요한 것 (스킬 통합, 자동화 파이프라인 구축)

### 선정 규칙

- High Priority Finding → 반드시 `actions`에 포함
- 커맨드/파일경로가 명확한 것 우선 (애매한 조언 배제)
- 중복 제거: 같은 파일을 건드리는 액션은 하나로 묶기
- 최대 7개. 다 넣지 말고 **임팩트 있는 것만**.

### 제안톤 강제

actions 문장은 아래 규칙을 따른다:

- ❌ "~가 없습니다" / "~가 부족합니다" / "~를 해야 합니다"
- ✅ "~를 추가하면 {효과}" / "~하면 더 편해져요" / "~ 해볼 만해요"
- 명령조 금지, **제안+근거** 형태 유지
- 각 액션은 반드시 `예상 효과:` 필드 포함 → 사용자가 왜 할지 납득하도록

---

## Phase 3 — 리포트 작성

**두괄식 원칙**: 사용자가 **상단 30줄만 봐도 "뭘 할지" 알 수 있어야** 한다.
세부 스코어·표는 접힘(`<details>`)으로 숨긴다.

`.harness/check-reports/check-harness-{YYYY-MM-DD}-{scope}.md` 경로에 아래 포맷으로 Write.

```markdown
# 🧭 Harness Maturity Report

**{YYYY-MM-DD}** · Scope: `{overall|project|all}` · Project: `{name or "-"}`

---

## 🧭 Harness Score: **{NN} / 100**  ({Excellent|Good|Fair|Needs Work})

```
Overall  L{n}  ▓▓▓▓▓▓▓░░░  {XX}% → L{n+1}    (Score: {NN})
Project  L{n}  ▓▓▓▓▓▓▓▓░░  {XX}% → L{n+1}    (Score: {NN})
```

> 진행바는 현재 레벨에서 다음 레벨까지의 진척도입니다. 점수는 다음 진단 때와 비교해보세요.

## 🎯 한 줄 요약

> 가장 큰 문제는 **{핵심 문제}**, 가장 쉽게 시작할 수 있는 건 **{Top 1 액션}** 입니다.

**잘 되고 있는 것**: {강점 1줄}
**개선하면 좋을 것**: {약점 1줄}

---

## ✅ 지금 하면 좋은 것 (제안)

> 부담 없이 골라서 하세요. 난이도/임팩트 순입니다.

### 🟢 바로 해볼 만한 것
- [ ] **{action}** — 예상 효과: {benefit}
  - 제안 명령: `{copy-paste 가능한 커맨드}`
  - 근거: {evidence 1줄}

### 🟡 한 번 정리하면 좋을 것
- [ ] **{action}** — 예상 효과: {benefit}
  - 참고: {파일 경로 or 스니펫 링크}
  - 근거: {evidence 1줄}

### 🔴 시간 내서 다뤄볼 것
- [ ] **{action}** — 예상 효과: {benefit}
  - 근거: {evidence 1줄}

---

## 📊 축별 스코어카드

| Axis                          | Score | Level | 다음 레벨까지 |
|-------------------------------|------:|:-----:|:-------------|
| A. Portfolio Hygiene          |  {NN} | L{n}  | {XX}% → L{n+1} |
| B. Execution Patterns         |  {NN} | L{n}  | {XX}% → L{n+1} |
| C. Context & Boundaries       |  {NN} | L{n}  | {XX}% → L{n+1} |
| D. Automation & Verification  |  {NN} | L{n}  | {XX}% → L{n+1} |

Sessions analyzed: {N} ({period}) · Scanned: {YYYY-MM-DD HH:MM}

---

<details>
<summary>📋 상세 체크리스트 (클릭해서 펼치기)</summary>

### A. Portfolio Hygiene  (Overall)
| ID | L  | Item                         | Status | Evidence |
|----|----|------------------------------|--------|----------|
| A1 | L1 | 70%+ skills used last 30d    | PASS   | used 92/131 (70.2%) |
| A2 | L2 | No dead skills               | FAIL   | 18 dead (e.g. `xxx`) |
| ... |   |                              |        |          |

### B. Execution Patterns  (Overall)
...

### C. Context & Boundaries  (Per-Project)
...

### D. Automation & Verification  (Per-Project)
...

</details>

<details>
<summary>🔍 Findings 전체 목록</summary>

**High Priority**
- 💡 CLAUDE.md에 dev 명령어 추가하면 좋겠어요 — "Development" 섹션에 install/test/run 넣기

**Medium Priority**
- 💡 `scaffold` 스킬이 30일간 호출 0회 — 트리거 키워드 재점검 또는 제거 고려

**Low Priority**
- 💡 PostToolUse 포맷터 훅 추가하면 편해요 — `.claude/settings.json`에 prettier/ruff 훅

</details>

<details>
<summary>🧩 Skill Portfolio 상세</summary>

Total installed: {N} · Used last 30d: {X} ({%})

| Category            | Count | Top examples |
|---------------------|------:|--------------|
| Dead (>90d unused)  |   N   | ...          |
| Ghost entries       |   N   | ...          |
| Duplicate clusters  |   N   | ...          |
| Prefix duplicates   |   N   | ...          |
| Trigger collisions  |   N   | ...          |

**정리 제안**
- 제거 후보: {skill} (0회, 설치 45일)
- 통합 제안: "code review" 클러스터 → {survivor} 유지
- 이름 충돌: `dev-scan` vs `dev:dev-scan`

</details>

<details>
<summary>⚡ Execution Pattern 상세</summary>

- Plan-first ratio:      {X}%
- Delegation ratio:      {X}%
- Parallel invocations:  {N}
- Handoff ratio:         {X}%
- Completion check:      {X}%
- Top 3-gram share:      {X}%  {⚠️ if > 30%}

**자동화 후보 (반복 패턴)**
- `Read → Edit → Bash(npm test)` — 18회 반복 → 스킬 후보

</details>
```

### 리포트 작성 시 톤 가이드

- **판정문 금지** — "~가 없습니다", "~가 부족합니다" ❌
- **제안문 사용** — "~를 추가하면 {효과}", "~하면 더 편해져요" ✅
- Findings 라벨은 `[Missing]/[Broken]/[Opportunity]` 대신 **💡 이모지 + 제안형 문장** 하나로 통일
- 상단 "지금 하면 좋은 것"의 각 액션은 **카피-페이스트 가능한 커맨드**나 **파일 경로**를 반드시 포함

---

## Hard Rules

1. **프롬프트 내용 읽지 않는다** — session-pattern-analyzer는 tool_use 메타데이터만
2. **프로젝트 파일 수정 금지** — 리포트만 `.harness/check-reports/`에 작성
3. **증거 기반 평가** — 모든 상태에 근거 문자열 필수
4. **맥락 인식** — 프로젝트 무관 항목은 N/A (예: 프론트엔드 없는 프로젝트의 UI 훅)
5. **병렬 실행** — Phase 1 서브에이전트는 반드시 같은 메시지에서 동시 호출
6. **캐시 재사용** — `/tmp/cc-cache/check-harness/` JSON 4개는 후속 분석에서 재사용 가능
