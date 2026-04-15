# Harness Engineering 체크리스트 (v3)

> AI가 잘 일하는 환경을 설계하는 기술 — **6축 · 24개 항목**
>
> 축은 "사이클"로 정렬되어 있다: **구조 → 맥락 → 계획 → 실행 → 검증 → 개선** → (다시 구조). 한 바퀴 돌 때마다 하네스가 단단해진다.

## 3단계 성숙도

| 레벨 | 이름 | 비유 |
|------|------|------|
| **L1** | 시작하기 | 교과서로 문법 배우기 |
| **L2** | 내 것으로 만들기 | 교과서 문법으로 내 글 쓰기 |
| **L3** | 자율 운영 | 글쓰기 스타일이 자리잡음 |

## 분석 프레임 — 2×3 매트릭스

|  | **Static** (갖춘 것) | **Behavioral** (하는 것) | **Δ/Growth** (축적) |
|---|---|---|---|
| **👤 User 스코프** | 1. 구조 | 3. 계획 · 4. 실행 (공유) | 6. 개선 (공유) |
| **📁 Project 스코프** | 2. 맥락 | 4. 실행 (공유) · 5. 검증 | 6. 개선 (공유) |

모든 판정은 **갖춘 것과 실제로 하는 것의 gap** 또는 **하네스가 자라고 있는지** 에서 나온다.

---

## 축 1 — 구조 (Scaffolding) · 👤 User × Static [5개]

> "뭘 깔아두었는가?" — 설치된 스킬/플러그인/MCP가 정리되어 있고 실제로 쓰이는가.

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (PORTFOLIO) |
|---|----------|:-:|------------|------------------------|
| A1 | 설치된 스킬의 70% 이상이 최근 30일 내 사용됨 | L1 | 안 쓰는 스킬이 많을수록 AI가 엉뚱한 스킬을 트리거할 확률↑ | `used_last_30d / total_installed ≥ 0.7` |
| A2 | Dead 스킬(90일+ 미사용 또는 usageCount=0)이 없다 | L2 | Dead 스킬이 트리거 매칭 단계에서 noise로 작용, 컨텍스트 낭비 | `dead_count == 0` |
| A3 | Ghost 엔트리(usage 기록만 남고 설치 없음)가 없다 | L2 | 히스토리와 실제 상태 불일치 → 분석/복구 시 혼란 | `ghost_count == 0` |
| A4 | 중복 스킬 클러스터(같은 의도 여러 개)가 없다 | L2 | AI가 매번 다른 스킬 선택 → 결과 재현성↓, 개선 누적 안됨 | `duplicate_clusters == 0` |
| A5 | Prefix 중복/Trigger 충돌이 없다 | L3 | 같은 키워드로 2개 이상 매칭 → 의도와 다른 스킬 실행 | `prefix_duplicates == 0 && trigger_collisions == 0` |

---

## 축 2 — 맥락 (Context) · 📁 Project × Static [6개]

> "AI가 무엇을 아는가?" — CLAUDE.md·rules·MCP 등 프로젝트 지식이 점진적으로 노출되는가.

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (CONTEXT) |
|---|----------|:-:|------------|----------------------|
| C1 | CLAUDE.md 존재 & 프로젝트 목적/구조 설명 포함 | L1 | 없으면 매 세션마다 같은 설명 반복, 토큰·시간 낭비 | `has_claude_md && has_project_purpose` |
| C2 | CLAUDE.md 품질 — 모순/ambiguity/placeholder 없음 | L1 | 모순된 지시는 AI를 헷갈리게 해서 임의 해석 유발 | `quality.contradictions == 0 && quality.ambiguities == 0` |
| C3 | 민감 파일 보호 — `.gitignore` + PreToolUse 훅 | L1 | .env 등 비밀 파일 노출 시 사고 → 사전 차단 필수 | `sensitive_protection.gitignore && sensitive_protection.hook_exists` |
| C4 | Rules 분리 — `.claude/rules/` 파일 ≥ 2개 & 역할 구분 | L2 | CLAUDE.md 비대화 방지, 역할별 로딩으로 컨텍스트 효율↑ | `rules_count ≥ 2 && rules_have_distinct_scope` |
| C5 | 외부 시스템 연결 — MCP 서버 설정 | L2 | DB·API 등 외부 데이터 직접 접근으로 코드 대신 팩트로 판단 | `mcp_configured` |
| C6 | Progressive Disclosure — 조건부 로드 (glob/skill) | L3 | 모든 규칙을 항상 로드하면 컨텍스트 폭발, 필요할 때만 로드 | `conditional_load_evidence ≥ 1` |

---

## 축 3 — 계획 (Planning) · 👤 User × Behavioral [1개]

> "뭘 할지 정하는가?" — "해줘" 대신 spec/plan·AskUserQuestion으로 모호성을 먼저 없애는가.

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (SESSION) |
|---|----------|:-:|------------|----------------------|
| B2 | 계획 후 실행 — specify/scaffold/plan 스킬 활용 | L2 | 바로 코딩 시작 vs 계획 후 실행의 품질 격차가 큼 | `plan_first_ratio ≥ 0.3` |

---

## 축 4 — 실행 (Execution) · 👤+📁 × Behavioral [3개]

> "어떻게 시키는가?" — 혼자 vs 부하 파견, 팀·subagent·오케스트레이션으로 작업을 어떻게 배치하는가.

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 (SESSION) |
|---|----------|:-:|------------|----------------------|
| B3 | 위임 활용 — Agent 호출이 있는 세션 비율 | L2 | 메인 컨텍스트 보호 + 병렬화로 속도·품질 동시 개선 | `delegation_ratio ≥ 0.4` |
| B5 | 병렬 실행 — `run_in_background=true` 사용 | L3 | 독립 작업을 순차 실행하면 시간 낭비 | `parallel_count ≥ 1` |
| B6 | 반복 요청 자동화율 — top-5 tool 3-gram이 전체의 30% 미만 | L3 | 같은 tool 패턴 반복 = 스킬/훅으로 뽑아낼 자동화 기회 | `top_ngram_share < 0.3` |

> 이 축은 User 전역과 Project 전용 세션 양쪽에서 계산한다. 리포트에는 **User / Project 두 값을 나란히** 표기한다.

---

## 축 5 — 검증 (Verification) · 📁 × Static+Behavioral [6개]

> "어떻게 믿는가?" — 기준·분리된 관점·독립 검증자로 컨텍스트와 모델을 나눠서 스스로를 속이지 않게 한다.

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 |
|---|----------|:-:|------------|-----------|
| B1 | 완료 기준 충족 패턴 — 세션 종료 전 test/ralph 실행 | L1 | 검증 없이 종료하면 "다 됐다"는 착각만 남고 재작업 비용↑ | SESSION: `completion_check_ratio ≥ 0.5` |
| D1 | 테스트 환경 — 스크립트 또는 테스트 프레임워크 | L1 | 검증할 수단이 없으면 AI 결과물의 품질을 확인 불가 | AUTOMATION: `test_runner_configured` |
| D2 | 포맷터/린터 자동 적용 — PostToolUse 훅 | L1 | 매번 수동 포맷 지시 = 반복 비용, 훅 한 번으로 해결 | AUTOMATION: `posttool_format_hook` |
| D3 | 위험 작업 차단 — PreToolUse 훅 존재 | L1 | rm -rf, force push 등 실수 방지의 안전망 | AUTOMATION: `pretool_block_hook` |
| D4 | 프로젝트 스킬/훅 실제 사용됨 — 세션에서 호출 확인 | L2 | 만들어두고 안 쓰면 존재 가치 없음, 실제 사용 여부 확인 | AUTOMATION: `project_skills_used ≥ 1` |
| D5 | 만드는/검증 AI 분리 — verifier 류 에이전트 존재 | L3 | 같은 AI가 구현·검증하면 자기확증 편향, 분리로 품질 안정화 | AUTOMATION: `verifier_agent_exists` |

---

## 축 6 — 개선 (Compounding) · 👤+📁 × Growth [3개 + 1]

> "어떻게 나아지는가?" — 세션에서 배운 것이 SKILL·Docs·CLAUDE.md 등 하네스 artifact로 돌아와 축적되는가.

| # | 체크 항목 | L | 왜 중요한가 | 판정 근거 |
|---|----------|:-:|------------|-----------|
| E1 | 최근 30일 내 CLAUDE.md/rules/docs 중 하나라도 갱신됨 | L1 | 설정이 한 번도 업데이트되지 않으면 학습이 축적 안 됨 | AUTOMATION: `compounding.claude_md_updated_30d OR rules_added_90d>0 OR docs_learnings_exist` |
| B4 | 세션 인수인계 — session-wrap/memory write 흔적 | L2 | 인수인계 없으면 다음 세션에서 같은 설명 반복 | SESSION: `handoff_ratio ≥ 0.5` (장시간 세션 중) |
| E2 | session-wrap / compound / memory-write 류 사용 ≥1회 | L2 | 세션 학습이 외부 기록으로 환류되어야 다음 세션에서 활용됨 | SESSION: `skills_invoked` 에 wrap/compound/memory 계열 포함 |
| E3 | 최근 90일 내 프로젝트에 새 skill·hook·rule 추가 | L3 | 개선안이 artifact로 굳어야 L3(자율 운영) 달성 | AUTOMATION: `compounding.skills_added_90d ≥ 1 OR hooks_added_90d ≥ 1` |

---

## 판정 상태

| 상태 | 의미 |
|------|------|
| **PASS** | 증거 기반 충족 |
| **WEAK_PASS** | 조건은 만족하나 품질 낮음 (Quick Win 후보) |
| **FAIL** | 증거 없음 또는 명시적 실패 |
| **N/A** | 프로젝트 성격상 적용 불가 또는 증거 수집 불가 |

스코어 계산: PASS=1, WEAK_PASS=0.5, FAIL=0, N/A=제외 · 가중치 L1×3, L2×2, L3×1

---

## 성숙도 달성 규칙

- **축별 레벨**: 해당 축의 모든 L1 항목 PASS → L1 달성, L2도 모두 PASS → L2 달성, …
- **User Maturity**: 축 1 · 3 · 4(User부분) · 6(User부분) 중 낮은 레벨
- **Project Maturity**: 축 2 · 4(Project부분) · 5 · 6(Project부분) 중 낮은 레벨

---

## "잘 가고 있다"는 신호

- 같은 말을 두 번 하지 않는다 (맥락 축적)
- 실수가 규칙이 된다 (개선 루프 — 축 6)
- 안 쓰는 스킬이 계속 줄어든다 (구조 정리 — 축 1)
- 새 세션에서 설명 시간이 줄어든다 (맥락 품질 — 축 2)

## "실패하고 있다"는 징후

- 설정 파일이 길어지기만 한다 (맥락 비대화)
- AI 결과의 절반을 다시 고친다 (검증 부재)
- 만들어둔 자동화를 안 쓴다 (구조에 dead skill)
- 세션마다 처음부터 설명한다 (인수인계 · 개선 실패)

---

## 용어집

| 용어 | 뜻 |
|------|----|
| **하네스(Harness)** | AI가 잘 일하도록 만든 환경 — 스킬·훅·규칙·컨텍스트의 총합. |
| **스킬(Skill)** | 특정 상황에 트리거되는 재사용 가능한 프롬프트+로직 모음. `SKILL.md`로 정의. |
| **플러그인(Plugin)** | 스킬·훅·에이전트·커맨드를 묶어 배포하는 단위. |
| **훅(Hook)** | 특정 이벤트(PreToolUse, PostToolUse, Stop 등)에 자동 실행되는 스크립트. |
| **에이전트(Agent)** | 독립 컨텍스트에서 돌아가는 하위 AI. |
| **MCP** | Model Context Protocol. 외부 시스템을 AI가 직접 접근하는 표준. |
| **CLAUDE.md** | 프로젝트 루트에 두는 AI용 지침서. |
| **Dead 스킬** | 90일+ 호출 0회인 스킬. |
| **Ghost 엔트리** | 사용 기록은 있지만 실제 설치는 없는 상태. |
| **Trigger 충돌** | 같은 키워드로 2개 이상 스킬이 매칭되는 상황. |
| **Progressive Disclosure** | "필요할 때만 로드" 원칙. |
| **Plan-first ratio** | 세션 시작 전 specify/scaffold/plan 스킬로 계획부터 세운 세션 비율. |
| **Handoff** | 세션 종료 시 다음 세션이 이어받도록 memory/CLAUDE.md/session-wrap에 상태 기록. |
| **Compounding** | 세션에서 얻은 학습이 SKILL·rules·docs로 환류되어 하네스가 자라는 것. |
| **Verifier 에이전트** | 독립적으로 검증하는 에이전트. 자기확증 편향 완화. |

---

*총 24개 항목 (6축) | L1: 9개, L2: 9개, L3: 6개*
