---
name: doc-drift
context: fork
description: |
  Use this skill when the user wants to audit the memory and documents Claude
  Code loads into context — CLAUDE.md (user global + project + nested),
  MEMORY.md, @imports, .claude/skills, .claude/agents, .claude/commands,
  installed plugins — and detect three kinds of issues: outdated claims,
  mutually contradictory statements, and risky-or-ambiguous wording. Produces
  a prioritized improvement list at `.drift-reports/`. Zero config.
  Trigger phrases: "doc drift", "memory drift", "memory audit", "context drift",
  "docs audit", "문서 점검", "문서 감사", "메모리 감사", "메모리 점검",
  "outdated 문서", "문서 충돌".
---

# doc-drift — Claude memory audit

한 문장: **Claude가 이 프로젝트에서 읽어들이는 메모리와 문서를 훑어보고, 낡은 것·서로 안 맞는 것·위험하거나 모호한 것을 찾아서 우선순위로 정리해준다.**

## 3단계

### 1. 읽는 것들 모으기

Claude Code가 이 프로젝트 맥락에서 실제로 로드하거나 참조할 수 있는 파일들을 모은다. 스크립트 없이 LLM이 `Read` / `Glob` / `Grep`으로 직접 찾는다.

**시작점**
- `~/.claude/CLAUDE.md` (있으면)
- `<cwd>/CLAUDE.md` + nested `**/CLAUDE.md`
- `~/.claude/projects/<encoded-cwd>/memory/MEMORY.md`
  - 인코딩: cwd 절대경로의 `/` 를 `-` 로 치환. 예: `/foo/bar` → `-foo-bar`
- `<cwd>/.claude/skills/**/SKILL.md`
- `<cwd>/.claude/agents/*.md`
- `<cwd>/.claude/commands/*.md`
- `~/.claude/plugins/**` (설치된 플러그인의 skills/agents/commands)

**확장**
각 파일에서 아래를 추출해 재귀적으로 따라간다:
- `@import` 토큰 (user global은 `~/.claude/` 기준, 그 외는 파일 디렉토리 또는 프로젝트 루트 기준)
- 상대경로 마크다운 링크 `[text](./path.md)`
- 백틱 안의 파일 경로 언급 (실존하는 경우만)

새 노드가 더 안 나올 때까지 수집.

### 2. 세 가지를 감지

감사 대상 각 파일을 LLM이 읽고, **다음 세 가지만** 찾는다:

| 종류 | 기준 |
|------|------|
| **Outdated** | 코드/설정의 현실과 안 맞는 주장 (경로, 커맨드, 숫자, 정책, 버전 등) |
| **Conflict** | 두 문서가 같은 주제를 서로 다르게 서술 |
| **Risky / Ambiguous** | 해석이 갈리거나, 잘못 따라하면 위험한 지시 (예: "적당히 하라", "경우에 따라", 명시 없이 삭제·오버라이드 지시 등) |

각 finding에는 **증거**가 붙어야 한다: 주장한 위치(`file:line`)와 반증(`file:line` 또는 현재 코드 인용).

확신 낮으면 아예 빼라. 오탐이 이 도구의 가장 큰 적이다.

### 3. 우선순위로 개선 제안

다음 기준으로 정렬해 리포트 작성:

1. **영향 범위** — `CLAUDE.md` / `MEMORY.md` 등 매 대화에 로드되는 파일이 최상위
2. **심각도** — 잘못된 지시를 따르면 망가지는 것(HIGH), 헷갈리지만 복구 쉬운 것(MED), 사소한 불일치(LOW)
3. **수정 난이도** — 수정안이 명확한 것부터

각 finding에 **제안하는 수정안**을 포함. 사람이 한눈에 "OK/NO"로 판단 가능한 수준으로.

## 출력

`.drift-reports/` 디렉토리에 저장 (없으면 생성, `.gitignore` 추가 금지 — 히스토리가 PR에서 보여야 함):

- `.drift-reports/<YYYY-MM-DD-HHMM>.md` — 타임스탬프 리포트
- `.drift-reports/latest.md` — 최신본 복사

**리포트 템플릿:**

```markdown
# Memory Audit — {timestamp}

**Scanned:** {n} files reachable from CLAUDE.md / MEMORY.md / skills / agents
**Findings:** HIGH {h} / MED {m} / LOW {l}

## Top priority
1. **[HIGH] `path:line`** — {한 줄 요약}
   - 주장: "..."
   - 현실: `other/path:line` — ...
   - 제안: ...
2. ...

## Medium
...

## Low
...

## 판단 대기 (사람 결정 필요)
- Conflict 중 어느 쪽이 맞는지 불명확한 항목
- Ambiguous 중 의도가 불확실한 항목

## 다음 단계
자동 수정 PR 만들까요? (Outdated 중 수정안 명확한 것만)
```

## 자동 수정 (선택)

리포트 요약 출력 뒤에만 물어본다:

> "HIGH {h}건 발견. 어떻게 할까요?
> 1) 수정안 명확한 것만 PR 생성
> 2) 리포트만"

선택 시 `docs/drift-fix-<timestamp>` 브랜치에 finding당 atomic 커밋, `gh pr create`.

**항상 제외**: Conflict(어느 쪽이 맞는지 사람 판단), Risky/Ambiguous(의도 확인 필요).

## 원칙

- **오탐 최소화** — 확신 없으면 빼라. 사람이 리포트 신뢰해야 도구가 산다.
- **증거 필수** — 모든 finding은 `file:line` 양쪽 인용.
- **요약 + 링크 패턴 존중** — `CLAUDE.md`가 다른 문서를 요약·링크하는 건 정상. 의미가 실제로 drift한 경우에만 신고.
- **매 대화 로드되는 파일 우선** — `CLAUDE.md`, `MEMORY.md` 의 drift가 다른 어떤 파일보다 위험.

## 인자

| 입력 | 동작 |
|------|------|
| `/doc-drift` | 전체 감사 (기본) |
| `/doc-drift recent` / `recent 50` | 최근 N 커밋에 바뀐 영역만 |
| `/doc-drift path <glob>` | 특정 경로만 |
