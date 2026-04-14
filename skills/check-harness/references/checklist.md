# Harness Engineering 체크리스트 (v2)

> AI가 잘 일하는 환경을 설계하는 기술 — 4축 22개 항목

## 3단계 성숙도

| 레벨 | 이름 | 비유 |
|------|------|------|
| **L1** | 시작하기 | 교과서로 문법 배우기 |
| **L2** | 내 것으로 만들기 | 교과서 문법으로 내 글 쓰기 |
| **L3** | 자율 운영 | 글쓰기 스타일이 자리잡음 |

## 2가지 스코프

| 스코프 | 대상 | 축 |
|--------|------|----|
| **Overall** | 유저 전역 (`~/.claude` + 모든 세션) | A. Portfolio, B. Execution |
| **Per-Project** | 현재 프로젝트 루트 | C. Context, D. Verification |

---

## 축 A — Portfolio Hygiene (Overall) [5개]

> "설치된 스킬/플러그인이 정리되어 있고 실제로 쓰이고 있는가?"

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (PORTFOLIO_REPORT) |
|---|----------|:-:|------------|------------------------------|
| A1 | 설치된 스킬의 70% 이상이 최근 30일 내 사용됨 | L1 | 안 쓰는 스킬이 많을수록 AI가 엉뚱한 스킬을 트리거할 확률↑ | `used_last_30d / total_installed ≥ 0.7` |
| A2 | Dead 스킬(90일+ 미사용 또는 usageCount=0)이 없다 | L2 | Dead 스킬이 트리거 매칭 단계에서 noise로 작용, 컨텍스트 낭비 | `dead_count == 0` |
| A3 | Ghost 엔트리(usage 기록만 남고 설치 없음)가 없다 | L2 | 히스토리와 실제 상태 불일치 → 분석/복구 시 혼란 | `ghost_count == 0` |
| A4 | 중복 스킬 클러스터(같은 의도 여러 개)가 없다 | L2 | AI가 매번 다른 스킬 선택 → 결과 재현성↓, 개선 누적 안됨 | `duplicate_clusters == 0` |
| A5 | Prefix 중복/Trigger 충돌이 없다 | L3 | 같은 키워드로 2개 이상 매칭 → 의도와 다른 스킬 실행 | `prefix_duplicates == 0 && trigger_collisions == 0` |

---

## 축 B — Execution Patterns (Overall) [6개]

> "AI에게 일 시키는 방식이 설계되어 있는가?"

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (SESSION_REPORT) |
|---|----------|:-:|------------|----------------------------|
| B1 | 완료 기준 충족 패턴 — 세션 종료 전 test/ralph 실행 | L1 | 검증 없이 종료하면 "다 됐다"는 착각만 남고 재작업 비용↑ | `completion_check_ratio ≥ 0.5` |
| B2 | 계획 후 실행 — specify/scaffold/plan 스킬 활용 | L2 | 바로 코딩 시작 vs 계획 후 실행의 품질 격차가 큼 | `plan_first_ratio ≥ 0.3` |
| B3 | 위임 활용 — Agent 호출이 있는 세션 비율 | L2 | 메인 컨텍스트 보호 + 병렬화로 속도·품질 동시 개선 | `delegation_ratio ≥ 0.4` |
| B4 | 세션 인수인계 — session-wrap/memory write 흔적 | L2 | 인수인계 없으면 다음 세션에서 같은 설명 반복 | `handoff_ratio ≥ 0.5` (장시간 세션 중) |
| B5 | 병렬 실행 — `run_in_background=true` 사용 | L3 | 독립 작업을 순차 실행하면 시간 낭비 | `parallel_count ≥ 1` |
| B6 | 반복 요청 자동화율 — top-5 tool 3-gram이 전체의 30% 미만 | L3 | 같은 tool 패턴 반복 = 스킬/훅으로 뽑아낼 자동화 기회 | `top_ngram_share < 0.3` |

---

## 축 C — Context & Boundaries (Per-Project) [6개]

> "AI가 이 프로젝트의 맥락과 경계를 알고 있는가?"

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (CONTEXT_REPORT) |
|---|----------|:-:|------------|----------------------------|
| C1 | CLAUDE.md 존재 & 프로젝트 목적/구조 설명 포함 | L1 | 없으면 매 세션마다 같은 설명 반복, 토큰·시간 낭비 | `has_claude_md && has_project_purpose` |
| C2 | CLAUDE.md 품질 — 모순/ambiguity/placeholder 없음 | L1 | 모순된 지시는 AI를 헷갈리게 해서 임의 해석 유발 | `quality.contradictions == 0 && quality.ambiguities == 0` |
| C3 | 민감 파일 보호 — `.gitignore` + PreToolUse 훅 | L1 | .env 등 비밀 파일 노출 시 사고 → 사전 차단 필수 | `sensitive_protection.gitignore && sensitive_protection.hook_exists` |
| C4 | Rules 분리 — `.claude/rules/` 파일 ≥ 2개 & 역할 구분 | L2 | CLAUDE.md 비대화 방지, 역할별 로딩으로 컨텍스트 효율↑ | `rules_count ≥ 2 && rules_have_distinct_scope` |
| C5 | 외부 시스템 연결 — MCP 서버 설정 | L2 | DB·API 등 외부 데이터 직접 접근으로 코드 대신 팩트로 판단 | `mcp_configured` |
| C6 | Progressive Disclosure — 조건부 로드 (glob/skill) | L3 | 모든 규칙을 항상 로드하면 컨텍스트 폭발, 필요할 때만 로드 | `conditional_load_evidence ≥ 1` |

---

## 축 D — Project Automation & Verification (Per-Project) [5개]

> "이 프로젝트에 검증과 자동화가 구조적으로 들어 있는가?"

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (AUTOMATION_REPORT) |
|---|----------|:-:|------------|-------------------------------|
| D1 | 테스트 환경 — 스크립트 또는 테스트 프레임워크 | L1 | 검증할 수단이 없으면 AI 결과물의 품질을 확인 불가 | `test_runner_configured` |
| D2 | 포맷터/린터 자동 적용 — PostToolUse 훅 | L1 | 매번 수동 포맷 지시 = 반복 비용, 훅 한 번으로 해결 | `posttool_format_hook` |
| D3 | 위험 작업 차단 — PreToolUse 훅 존재 | L1 | rm -rf, force push 등 실수 방지의 안전망 | `pretool_block_hook` |
| D4 | 프로젝트 스킬/훅 실제 사용됨 — 세션에서 호출 확인 | L2 | 만들어두고 안 쓰면 존재 가치 없음, 실제 사용 여부 확인 | `project_skills_used ≥ 1` |
| D5 | 만드는/검증 AI 분리 — verifier 류 에이전트 존재 | L3 | 같은 AI가 구현·검증하면 자기확증 편향, 분리로 품질 안정화 | `verifier_agent_exists` |

---

## 판정 상태

| 상태 | 의미 |
|------|------|
| **PASS** | 증거 기반 충족 |
| **WEAK_PASS** | 조건은 만족하나 품질 낮음 (Quick Win 후보) |
| **FAIL** | 증거 없음 또는 명시적 실패 |
| **N/A** | 프로젝트 성격상 적용 불가 또는 증거 수집 불가 |

스코어 계산: PASS=1, WEAK_PASS=0.5, FAIL=0, N/A=제외

---

## 성숙도 달성 규칙

- **Overall Maturity**: Overall 2축(A, B) 중 낮은 레벨
- **Project Maturity**: Per-Project 2축(C, D) 중 낮은 레벨
- **Axis Level**: 해당 축의 모든 L1 항목 PASS → L1 달성, L2도 모두 PASS → L2 달성, …

---

## "잘 가고 있다"는 신호

- 같은 말을 두 번 하지 않는다 (맥락 축적)
- 실수가 규칙이 된다 (개선 루프)
- 안 쓰는 스킬이 계속 줄어든다 (Portfolio 정리)
- 새 세션에서 설명 시간이 줄어든다 (Context 품질)

## "실패하고 있다"는 징후

- 설정 파일이 길어지기만 한다 (Context 비대화)
- AI 결과의 절반을 다시 고친다 (Verification 부재)
- 만들어둔 자동화를 안 쓴다 (Portfolio dead)
- 세션마다 처음부터 설명한다 (인수인계 실패)

---

## 용어집

체크리스트에 나오는 용어 정리 (처음 보는 수강생용).

| 용어 | 뜻 |
|------|----|
| **하네스(Harness)** | AI가 잘 일하도록 만든 환경 — 스킬·훅·규칙·컨텍스트의 총합. 소프트웨어 엔지니어링의 "개발 환경"과 비슷한 개념. |
| **스킬(Skill)** | 특정 상황에 트리거되는 재사용 가능한 프롬프트+로직 모음. `SKILL.md`로 정의. |
| **플러그인(Plugin)** | 스킬·훅·에이전트·커맨드를 묶어 배포하는 단위. `.claude-plugin/plugin.json`. |
| **훅(Hook)** | 특정 이벤트(PreToolUse, PostToolUse, Stop 등)에 자동 실행되는 스크립트. 자동화의 핵심. |
| **에이전트(Agent)** | 독립 컨텍스트에서 돌아가는 하위 AI. 메인 세션의 컨텍스트 보호 + 병렬 실행에 사용. |
| **MCP** | Model Context Protocol. 외부 시스템(DB, API, 파일)을 AI가 직접 접근하는 표준. |
| **CLAUDE.md** | 프로젝트 루트에 두는 AI용 지침서. 목적·구조·규칙을 적어두면 매 세션 자동 로드됨. |
| **Dead 스킬** | 90일+ 호출 0회인 스킬. 트리거 매칭 노이즈가 되어 정리 대상. |
| **Ghost 엔트리** | `~/.claude.json`의 skillUsage에는 기록이 있지만 실제 설치는 없는 상태. |
| **트리거 충돌** | 같은 키워드로 2개 이상 스킬이 매칭되는 상황. 의도와 다른 스킬 실행 유발. |
| **Progressive Disclosure** | "필요할 때만 로드" 원칙. CLAUDE.md 비대화를 막고 컨텍스트 효율↑. |
| **PreToolUse / PostToolUse** | 각각 "도구 실행 전/후" 훅 이벤트. 검증·차단은 Pre, 포맷·로깅은 Post에 붙음. |
| **3-gram** | 연속된 3개 tool 호출 패턴(예: `Read → Edit → Bash`). 반복되면 스킬/훅으로 자동화 후보. |
| **Plan-first ratio** | 세션 시작 전에 specify/scaffold/plan 스킬로 계획부터 세운 세션 비율. |
| **Handoff** | 세션 종료 시 다음 세션이 이어받도록 memory/CLAUDE.md/session-wrap에 상태 기록. |
| **Verifier 에이전트** | 만든 결과물을 독립적으로 검증하는 에이전트. 구현-검증 분리로 자기확증 편향 완화. |

---

*총 22개 항목 (4축) | L1: 8개, L2: 9개, L3: 5개*
