---
name: check-harness
description: |
  Harness 성숙도 진단 — **6축 24항목 체크리스트**와 **2×3 분석 매트릭스**(Static/Behavioral/Growth × User/Project)로 하네스의 사이클(구조→맥락→계획→실행→검증→개선)을 평가한다.
  판단은 항상 "갖춘 것(Static) ↔ 실제로 하는 것(Behavioral)의 gap" 또는 "하네스가 자라고 있는지(Growth)"에서 나온다.
  4개 서브에이전트(skill-portfolio-analyzer, session-pattern-analyzer, context-quality-reviewer, project-automation-auditor)를 병렬 실행.
  session-pattern-analyzer는 User 전역과 현재 프로젝트 두 번 돌려 User/Project 스코프를 분리한다.

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

# /check-harness — Harness 성숙도 진단 (v3)

**6축 사이클** 로 평가한다: **구조 → 맥락 → 계획 → 실행 → 검증 → 개선**.
체크리스트 원본: `references/checklist.md` (반드시 먼저 읽어 항목 정의 확인).

**분석의 정의**
> 갖춘 것(Static)과 실제로 하는 것(Behavioral)의 **gap** 을, User/Project 두 스코프에서 측정한다.
> 축 6(개선)만 시간축 — 학습이 artifact로 돌아와 축적되는지(Growth)를 본다.

**산출물 2가지**
1. **스코어카드** — 6축별 PASS/WEAK_PASS/FAIL/N/A + User/Project 성숙도 레벨
2. **액션 리포트** — TL;DR + 사이클 서술 + Quick Wins

**출력 방식** — 대화에 풍부한 축별 렌더 + HTML/MD 파일 저장 + 자동 오픈
- **대화창**: 축별 섹션 × 6 + 인라인 ASCII 스코어 바 + 체크리스트 테이블 (Phase 3 A 블록)
- **파일**: `.harness/check-reports/check-harness-{YYYY-MM-DD}-{scope}/` 디렉토리에
  - `report.html` — 시각 리포트 (CSS 스코어 게이지, 축별 확장 패널, 자체 완결)
  - `report.md` — 마크다운 미러 (diff·git·아카이브용)
- **자동 오픈**: 마지막에 `Bash: open {dir}/report.html` 실행
- 리포트 끝 한 줄로 경로 안내: `📁 Saved: {dir}/ · 🌐 Opened: report.html`

---

## Phase 0 — Scope 결정

사용자 입력에 스코프가 명시됐으면 그대로:
- `overall` / `전체` / `유저` → User만
- `project` / `프로젝트` → Project만
- `all` / `둘 다` / 지정 없음 → **Both**

스코프가 불명확하면 `AskUserQuestion`:

```
question: "어디까지 진단할까요?"
options:
  - Both (User + Project) — 전체 진단 (Recommended)
  - User만 (전역 · 스킬 정리 + 전체 세션 습관)
  - Project만 (현재 프로젝트 맥락·자동화·프로젝트 세션)
```

### 프로젝트 루트 탐색 (Project 포함 시)

cwd에서 상위로 올라가며 `.claude` 또는 `CLAUDE.md`가 있는 디렉토리를 `PROJECT_ROOT`로. 못 찾으면 Project 스코프 비활성화.

### 캐시 디렉토리

```bash
mkdir -p /tmp/cc-cache/check-harness/
```

산출물 경로:
- `/tmp/cc-cache/check-harness/PORTFOLIO.json`
- `/tmp/cc-cache/check-harness/SESSION_USER.json`
- `/tmp/cc-cache/check-harness/SESSION_PROJECT.json`
- `/tmp/cc-cache/check-harness/CONTEXT.json`
- `/tmp/cc-cache/check-harness/AUTOMATION.json`

---

## Phase 1 — 병렬 데이터 수집

**선택된 스코프에 해당하는 서브에이전트를 모두 같은 메시지에서 병렬 호출.**

### User 스코프 에이전트 (2개)

```
Agent(
  subagent_type: "skill-portfolio-analyzer",
  description: "Skill portfolio health scan",
  prompt: """
    설치된 모든 스킬/플러그인/MCP를 ~/.claude.json과 교차 분석.
    enabledPlugins (user/project), mcpServers (user + project .mcp.json),
    installed plugins 인벤토리, MCP 호출 사용도(최근 30일) 포함.
    current_state/unused_mcp/plugin_findings 필드 포함.
    /tmp/cc-cache/check-harness/PORTFOLIO.json 에 저장.
    project_root={PROJECT_ROOT 있으면 전달}
    dead_days=90, low_value_days=60, low_value_count=3
  """
)

Agent(
  subagent_type: "session-pattern-analyzer",
  description: "User-wide session pattern scan",
  prompt: """
    scope=overall, days=7, long_session_min=20.
    ~/.claude/projects/ 전체 세션의 tool_use 메타데이터만 분석. 프롬프트 텍스트 절대 읽지 않음.
    /tmp/cc-cache/check-harness/SESSION_USER.json 에 저장.
  """
)
```

### Project 스코프 에이전트 (3개)

```
Agent(
  subagent_type: "session-pattern-analyzer",
  description: "Project-scope session pattern scan",
  prompt: """
    scope=current_project, project_dir={PROJECT_ROOT}, days=30, long_session_min=20.
    해당 프로젝트 세션만 대상. tool_use 메타데이터만.
    /tmp/cc-cache/check-harness/SESSION_PROJECT.json 에 저장.
  """
)

Agent(
  subagent_type: "context-quality-reviewer",
  description: "Project context quality review",
  prompt: """
    project_root={PROJECT_ROOT}
    CLAUDE.md + .claude/rules/* + .gitignore + settings.json + MCP 설정 평가.
    /tmp/cc-cache/check-harness/CONTEXT.json 에 저장.
  """
)

Agent(
  subagent_type: "project-automation-auditor",
  description: "Project automation, verification & compounding audit",
  prompt: """
    project_root={PROJECT_ROOT}
    session_report=/tmp/cc-cache/check-harness/SESSION_PROJECT.json  (있으면 참조)
    test/hooks/verifier/isolation 감사 + **compounding 신호**(git log 기반 CLAUDE.md·rules·docs·skills·hooks 최근 변경) 수집.
    /tmp/cc-cache/check-harness/AUTOMATION.json 에 저장.
  """
)
```

**Both 스코프면 5개를 한 번에 띄운다** (Project-session과 automation-auditor는 의존성이 있지만 auditor가 session 없이도 돌도록 설계 — 병렬 OK).

완료 후 JSON들을 Read로 읽어 `PORTFOLIO`, `SESSION_USER`, `SESSION_PROJECT`, `CONTEXT`, `AUTOMATION` 으로 메모리에 보관.

---

## Phase 2 — 체크리스트 판정 (6축 매핑)

24항목 각각에 대해 PASS/WEAK_PASS/FAIL/N/A를 판정하고 증거 문자열을 함께 기록.

### 축별 데이터 소스 매핑

| 축 | 소스 | 스코프 | 판정 근거 필드 |
|---|---|---|---|
| **1. 구조** | PORTFOLIO | User | summary (A1–A5) |
| **2. 맥락** | CONTEXT | Project | claude_md·rules·sensitive (C1–C6) |
| **3. 계획** | SESSION_USER | User | plan_first_ratio (B2) |
| **4. 실행** | SESSION_USER + SESSION_PROJECT | User+Project | delegation/parallel/top_ngram (B3·B5·B6) — **두 스코프 각각 계산** |
| **5. 검증** | SESSION_PROJECT + AUTOMATION | Project | completion_check + D1–D5 |
| **6. 개선** | AUTOMATION.compounding + SESSION (wrap/memory) | Both | E1·B4·E2·E3 |

### 4축(실행) 특이사항

B3/B5/B6는 **User 값과 Project 값 둘 다 기록**. 축 레벨 판정은 다음 규칙:
- Both 스코프: User·Project 중 **낮은 쪽**을 축 판정에 사용 (약한 고리 원칙)
- Single 스코프: 해당 스코프 값만

### 상태 규칙

- **PASS** — 증거 명확
- **WEAK_PASS** — 조건 충족하나 리포트의 `weak_pass_flags`에 해당 필드 있음
- **FAIL** — 증거 없음 또는 명시적 실패
- **N/A** — 데이터 없음(예: 세션 0개) 또는 프로젝트 성격상 무관

### 성숙도 계산

**축별 레벨**
- L1 항목 전부 PASS/WEAK_PASS → L1 달성
- L2도 모두 → L2, L3도 모두 → L3

**User Maturity** = min(축1, 축3, 축4-User부분, 축6-User부분)
**Project Maturity** = min(축2, 축4-Project부분, 축5, 축6-Project부분)

**점수 (축별, 0–100)**
- 항목 점수: PASS=1, WEAK_PASS=0.5, FAIL=0, N/A=제외
- 가중치: L1×3, L2×2, L3×1
- 축 점수 = Σ(점수 × 가중치) / Σ(가중치) × 100

**Harness Score**
- User Score = mean(축1, 축3, 축4-User)
- Project Score = mean(축2, 축4-Project, 축5)
- Compounding(축6)은 User·Project 각각에 가산되기보다 **독립 축 점수**로 리포트에 표기 (스코어 합산 안 함 — 축적은 시간축이라 단일 시점 합산이 왜곡됨)
- **Harness Score** = (User + Project + Compounding) / 3 (Both일 때)
- 등급: 90+ Excellent / 75+ Good / 60+ Fair / <60 Needs Work

**다음 레벨 진행률**: `현재 레벨의 통과 항목 수 / 다음 레벨 요구 항목 수`

---

## Phase 2.5 — TL;DR & 액션 합성

Phase 2 판정을 입력으로 리포트 상단 핵심 요약을 먼저 만든다.

### 생성할 5가지 변수

1. **`headline`** (1문장) — User/Project 성숙도 + "가장 큰 문제" + "가장 쉬운 시작점"
   예: *"User L2 / Project L1 — CLAUDE.md에 실행 명령이 빠져있고 개선 축이 얇아요. dev 명령어 3줄 추가부터 해보세요."*

2. **`cycle_line`** (6개, 각 1줄) — 사이클 순서로 축별 한 줄 상태:
   - `1. 구조 — {한 줄}`
   - `2. 맥락 — {한 줄}`
   - `3. 계획 — {한 줄}`
   - `4. 실행 — {한 줄, User/Project 대비 포함 가능}`
   - `5. 검증 — {한 줄}`
   - `6. 개선 — {한 줄}`

3. **`strength`** (1문장) — FAIL 적고 PASS 많은 축에서 뽑은 강점.

4. **`weakness`** (1문장) — FAIL 집중된 축의 약점.

5. **`actions`** (3–7개) — 3계층 분류:
   - 🟢 **바로 해볼 만한 것**: 단일 커맨드/한 줄 수정
   - 🟡 **한 번 정리하면 좋을 것**: 파일 하나 편집
   - 🔴 **시간 내서 다뤄볼 것**: 구조 변경

### 선정 규칙

- High Priority Finding → 반드시 포함
- 커맨드/파일경로 명확한 것 우선
- 중복 제거
- 최대 7개
- **Runtime 인벤토리(PORTFOLIO) 기반 제안 최소 1개 포함**
- **Compounding(축6) FAIL 시 제안 최소 1개 포함** — "이번 세션에서 배운 것을 rules/에 추가하면" 류

### 제안톤 강제

- ❌ "~가 없습니다" / "~가 부족합니다" / "~를 해야 합니다"
- ✅ "~를 추가하면 {효과}" / "~하면 더 편해져요"
- 각 액션에 `예상 효과:` 필드 포함

---

## Phase 3 — 리포트 작성 (3개 산출물)

**두괄식 + 사이클 서술 + 축별 풍부화**: 상단은 빠르게, 아래로 내려갈수록 축별 상세.

**3개 산출물을 동시에 만든다**:
- **A. 대화 렌더** (아래 마크다운 템플릿) — 바로 출력
- **B. report.md** (A와 동일 내용) — Write
- **C. report.html** (references/html-template.html 기반 자체완결) — Write, 그리고 `open` 으로 실행

### A/B 마크다운 템플릿 (대화 + report.md 공용)

```markdown
# 🧭 Harness Maturity Report

**{YYYY-MM-DD}** · Scope: `{user|project|all}` · Project: `{name or "-"}`

---

## 🧭 Harness Score: **{NN} / 100**  ({Excellent|Good|Fair|Needs Work})

```
👤 User Scope    L{n}  ▓▓▓▓▓▓▓░░░  {XX}% → L{n+1}   (Score: {NN})
📁 Project Scope L{n}  ▓▓▓▓▓▓▓▓░░  {XX}% → L{n+1}   (Score: {NN})
🔁 Compounding   L{n}  ▓▓▓▓░░░░░░  {XX}% → L{n+1}   (Score: {NN})
```

> User = `~/.claude/` 전역 (구조·계획·실행 User면)
> Project = 이 프로젝트 (맥락·검증·실행 Project면)
> Compounding = 하네스가 자라고 있는지 (축 6, 공통)

## 🎯 한 줄 요약

> {headline}

**잘 되고 있는 것**: {strength}
**개선하면 좋을 것**: {weakness}

---

## 🔄 사이클 한눈에 보기

> 구조 → 맥락 → 계획 → 실행 → 검증 → 개선 순서로 한 줄씩.

1. **구조** · 👤 User — {cycle_line[0]}
2. **맥락** · 📁 Project — {cycle_line[1]}
3. **계획** · 👤 User — {cycle_line[2]}
4. **실행** · 👤+📁 — {cycle_line[3]}
5. **검증** · 📁 Project — {cycle_line[4]}
6. **개선** · 👤+📁 — {cycle_line[5]}

---

## ✅ 지금 하면 좋은 것 (제안)

> 부담 없이 골라서 하세요. 각 액션에 **적용 범위(👤/📁)** 표시.

### 🟢 바로 해볼 만한 것
- [ ] {👤|📁} **{action}** — 예상 효과: {benefit}
  - 제안 명령: `{copy-paste 커맨드}`
  - 근거: {evidence}

### 🟡 한 번 정리하면 좋을 것
- [ ] {👤|📁} **{action}** — 예상 효과: {benefit}
  - 참고: {파일 경로}
  - 근거: {evidence}

### 🔴 시간 내서 다뤄볼 것
- [ ] {👤|📁} **{action}** — 예상 효과: {benefit}
  - 근거: {evidence}

---

## 📊 축별 스코어카드 (6축)

| Axis | Scope | Score | Level | 다음 레벨까지 |
|------|-------|------:|:-----:|:-------------|
| 1. 구조 (Scaffolding)      | 👤 User     | {NN} | L{n} | {XX}% → L{n+1} |
| 2. 맥락 (Context)          | 📁 Project  | {NN} | L{n} | {XX}% → L{n+1} |
| 3. 계획 (Planning)         | 👤 User     | {NN} | L{n} | {XX}% → L{n+1} |
| 4. 실행 (Execution)        | 👤+📁        | {NN} | L{n} | {XX}% → L{n+1} |
| 5. 검증 (Verification)     | 📁 Project  | {NN} | L{n} | {XX}% → L{n+1} |
| 6. 개선 (Compounding)      | 👤+📁        | {NN} | L{n} | {XX}% → L{n+1} |

Sessions analyzed: User {Nu} / Project {Np} ({period}) · Scanned: {YYYY-MM-DD HH:MM}

---

## 🔌 현재 활성 상태 (Runtime Inventory)

> 지금 이 Claude Code 세션에서 **실제로 접근 가능한** 것들.

**📦 플러그인** ({enabled_project}/{installed} enabled in this project)
| Plugin | Version | Scope | Skills | 최근 30일 활용 |
|--------|:-------:|:-----:|------:|:--------------|
| ... | ... | 👤📁 | ... | ... |

**🔗 MCP 서버** ({total}개 · {unused}개 미사용)
| Server | Scope | Type | 최근 호출 |
|--------|:-----:|:----:|:--------:|
| ... | ... | ... | ... |

**🧩 스킬 출처**
- 유저 standalone: {N}개
- 플러그인 경유: {N}개
- 프로젝트 로컬: {N}개

---

<details>
<summary>🧭 2×3 분석 매트릭스 (Static/Behavioral/Growth × User/Project)</summary>

|  | Static (갖춘 것) | Behavioral (하는 것) | Growth (축적) |
|---|---|---|---|
| 👤 User    | 축 1 (Score {NN}) | 축 3 · 축 4-User (Score {NN}/{NN}) | 축 6-User (Score {NN}) |
| 📁 Project | 축 2 (Score {NN}) | 축 4-Project · 축 5 (Score {NN}/{NN}) | 축 6-Project (Score {NN}) |

**교차에서 나온 gap 요약**
- Static vs Behavioral (User): {설치됐는데 안 쓰는 스킬 N개 → B2 plan-first 낮음}
- Static vs Behavioral (Project): {hook 있는데 호출 0회 / 없는데 필요한 징후}
- Growth: {최근 30일 artifact 갱신 {N}건 → 축적 여부}

</details>

<details>
<summary>📋 상세 체크리스트 (6축 · 24항목)</summary>

### 축 1 — 구조 (👤 User × Static)
| ID | L | Item | Status | Evidence |
|----|---|------|--------|----------|
| A1 | L1 | 70%+ skills used last 30d | PASS | ... |
| ... |

### 축 2 — 맥락 (📁 Project × Static)
...

### 축 3 — 계획 (👤 User × Behavioral)
...

### 축 4 — 실행 (👤+📁 × Behavioral)
> User / Project 두 값 표기

| ID | L | Item | User | Project | Status(min) | Evidence |
|----|---|------|------|---------|-------------|----------|
| B3 | L2 | delegation_ratio ≥ 0.4 | 0.55 | 0.20 | FAIL(project) | ... |
| ... |

### 축 5 — 검증 (📁 Project)
...

### 축 6 — 개선 (👤+📁 × Growth)
| ID | L | Item | Status | Evidence |
|----|---|------|--------|----------|
| E1 | L1 | CLAUDE.md/rules/docs 30d 갱신 | PASS | CLAUDE.md updated 2026-04-02 |
| B4 | L2 | session-wrap/handoff ratio | WEAK_PASS | 0.42 |
| E2 | L2 | wrap/compound/memory 호출 ≥1 | PASS | session-wrap: 3회 |
| E3 | L3 | 최근 90일 신규 skill/hook/rule | FAIL | 0건 |

</details>

<details>
<summary>🔍 Findings 전체 목록</summary>

**High Priority**
- 💡 {...}

**Medium Priority**
- 💡 {...}

**Low Priority**
- 💡 {...}

</details>

<details>
<summary>🧩 Skill Portfolio 상세 (👤 User × Static)</summary>

전체 설치 스킬: {N}개 · 최근 30일 사용: {X}개 ({%})

| 분류 | 개수 | 설명 | 예시 |
|------|----:|------|------|
| 😴 오래 안 쓴 스킬 (90일+) | N | ... | ... |
| 👻 기록만 남은 스킬 | N | ... | ... |
| 🔁 비슷한 기능 중복 | N | ... | ... |
| 🏷️ 네임스페이스 중복 | N | ... | ... |
| ⚠️ 트리거 겹침 | N | ... | ... |

</details>

<details>
<summary>⚡ Execution 상세 (최근 세션 습관 — User vs Project)</summary>

|                          | 👤 User | 📁 Project |
|--------------------------|-------:|-----------:|
| plan-first ratio         | {X}%   | {X}%       |
| delegation ratio         | {X}%   | {X}%       |
| parallel calls           | {N}    | {N}        |
| handoff ratio            | {X}%   | {X}%       |
| completion check ratio   | {X}%   | {X}%       |
| top 3-gram share         | {X}%   | {X}%       |

**자동화 후보 (반복 패턴)**
- User: `Read → Edit → Bash(npm test)` — {N}회
- Project: `...` — {N}회

</details>

<details>
<summary>🔁 Compounding 상세 (축 6 — 하네스의 축적)</summary>

- CLAUDE.md 최근 30일 갱신: {Yes/No} ({commit evidence})
- `.claude/rules/` 최근 90일 추가: {N}개
- `skills/` 최근 90일 추가: {N}개
- `hooks/` 최근 90일 변경: {N}건
- `docs/learnings/` 존재: {Yes/No}
- 세션에서 session-wrap/compound 호출: User {N}회 / Project {N}회

**관찰**
- {예: 최근 이 프로젝트에서 새 rule이 추가된 적이 없음 → 학습이 사람 머리에만 있음}

</details>

📁 Saved: {dir}/ · 🌐 Opened: report.html
```

### 대화 렌더 확장 규칙 (A 블록)

대화에 내보낼 때는 **축별 섹션**을 더 풍부하게 출력한다. 각 축마다:

```
### {N}. {축 이름} · {스코프 이모지} · {상태 요약}
Score: {NN}/100  L{n}  ▓▓▓▓▓▓▓░░░ {XX}% → L{n+1}

핵심:
- ✅ {잘 된 것 1–2개, 근거 숫자 포함}
- ⚠️ {개선 포인트 1–2개, 근거 숫자 포함}

체크리스트:
| ID | L | Item | Status | Evidence |
|----|---|------|--------|----------|
| {row} |

가장 값싼 다음 한 수: {quick win + 커맨드 또는 경로}
```

6축 전부 동일 구조로 출력. 각 축 섹션은 5–12줄이면 충분.

### C 블록 — HTML 리포트 생성

1. `references/html-template.html` 읽기 (자체완결 HTML — 인라인 CSS, 스코어 게이지, 축별 카드, 접힘 패널)
2. 아래 placeholder 치환:
   - `{{GENERATED_AT}}`, `{{SCOPE}}`, `{{PROJECT_NAME}}`
   - `{{HARNESS_SCORE}}`, `{{HARNESS_GRADE}}`
   - `{{USER_SCORE}}`, `{{USER_LEVEL}}`, `{{PROJECT_SCORE}}`, `{{PROJECT_LEVEL}}`, `{{COMPOUNDING_SCORE}}`, `{{COMPOUNDING_LEVEL}}`
   - `{{HEADLINE}}`, `{{STRENGTH}}`, `{{WEAKNESS}}`
   - `{{CYCLE_ROWS}}` — 6축 한 줄씩 `<li>{cycle_line}</li>`
   - `{{ACTIONS_GREEN}}`, `{{ACTIONS_YELLOW}}`, `{{ACTIONS_RED}}` — 액션 카드 `<li>`
   - `{{AXIS_CARDS}}` — 6축 카드 (아래 축별 구조)
   - `{{INVENTORY_TABLE}}` — 런타임 인벤토리 테이블
   - `{{MATRIX_TABLE}}` — 2×3 매트릭스
   - `{{FINDINGS_LIST}}`
3. Write 후 `Bash: open {dir}/report.html` 실행 — macOS 기본 브라우저로 열림.

축별 카드 블록 예시(HTML 안에서 반복):
```html
<article class="axis" data-status="{pass|weak|fail|na}">
  <header>
    <h3>{icon} 축 {N} — {이름}</h3>
    <div class="score">
      <span class="score-num">{NN}</span>
      <div class="bar"><span style="width:{XX}%"></span></div>
      <span class="level">L{n}</span>
    </div>
  </header>
  <p class="headline">{요약 한 줄}</p>
  <details><summary>체크리스트 · {PASS_N}/{TOTAL_N} 통과</summary>
    <table>...</table>
  </details>
  <p class="next-move">💡 {quick win}</p>
</article>
```

### 리포트 작성 시 톤 가이드

- **판정문 금지** — "~가 없습니다" ❌
- **제안문 사용** — "~를 추가하면 {효과}" ✅
- Findings 라벨은 **💡 이모지 + 제안형 문장** 하나로 통일
- "지금 하면 좋은 것"의 각 액션은 **카피-페이스트 가능한 커맨드**나 **파일 경로** 포함
- 사이클 한눈에 보기의 각 라인은 **상태(✅/⚠️/❌) + 근거 숫자 한 개** 포함 원칙

---

## Hard Rules

1. **프롬프트 내용 읽지 않는다** — session-pattern-analyzer는 tool_use 메타데이터만
2. **프로젝트 파일 수정 금지** — 리포트만 `.harness/check-reports/`에 작성
3. **증거 기반 평가** — 모든 상태에 근거 문자열 필수
4. **맥락 인식** — 프로젝트 무관 항목은 N/A
5. **병렬 실행** — Phase 1 서브에이전트는 반드시 같은 메시지에서 동시 호출 (Both 스코프면 5개)
6. **User/Project 세션 분리** — session-pattern-analyzer는 scope별로 따로 호출 (SESSION_USER, SESSION_PROJECT)
7. **축 6(개선)은 Growth 축** — 시간 미분이므로 단일 시점 합산이 아니라 독립 점수로 표기
8. **캐시 재사용** — `/tmp/cc-cache/check-harness/` JSON 5개는 후속 분석에서 재사용 가능
