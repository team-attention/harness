---
name: check-harness
description: |
  Harness 성숙도 진단 — 5축 35개 체크리스트 기반 스코어카드 + 액션 리포트.
  현재 프로젝트의 .claude 구조, 최근 1주일 세션, scaffold 구성을 분석하여
  L1/L2/L3 성숙도 레벨과 구체적 개선 액션을 제시한다.
  "/check-harness", "check harness", "하네스 체크", "하네스 점검",
  "harness check", "설정 점검", "뭐가 부족한지 봐줘", "harness audit",
  "harness health", "하네스 진단", "성숙도 점검", "maturity check"
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
  - Agent
  - AskUserQuestion
---

# /check-harness — Harness 성숙도 진단

5축 35개 체크리스트 기반으로 프로젝트의 harness 성숙도를 진단한다.
체크리스트 원본은 `references/checklist.md`를 참조한다.

**산출물 2가지:**
1. **스코어카드** — 5축별 통과/미통과 + L1→L2→L3 성숙도
2. **액션 리포트** — Missing/Broken/Opportunity 분류 + Quick Wins

---

## Phase 0: 프로젝트 인식

### 프로젝트 루트 탐색

cwd에서 시작하여 `.claude` 디렉토리를 찾을 때까지 상위로 올라간다.

```bash
PROJECT_ROOT=""
DIR="$(pwd)"
while [ "$DIR" != "/" ]; do
  if [ -d "$DIR/.claude" ] || [ -f "$DIR/CLAUDE.md" ]; then
    PROJECT_ROOT="$DIR"
    break
  fi
  DIR="$(dirname "$DIR")"
done

if [ -z "$PROJECT_ROOT" ]; then
  echo "ERROR: .claude 또는 CLAUDE.md를 찾을 수 없습니다."
  echo "Claude Code 프로젝트 디렉토리에서 실행해주세요."
  exit 1
fi

echo "Project root: $PROJECT_ROOT"
```

프로젝트를 찾지 못하면 에러 메시지를 출력하고 종료한다.

찾았으면 `PROJECT_ROOT`를 기준으로 모든 분석을 수행한다.

---

## Phase 1: 데이터 수집

모든 데이터를 한 번에 수집하여 CONTEXT_BLOCK으로 조립한다.

### 1.1 프로젝트 구조

```bash
cd "$PROJECT_ROOT"

# 디렉토리 트리 (depth 3, .git/node_modules 제외)
find . -maxdepth 3 -type f \
  -not -path './.git/*' \
  -not -path './node_modules/*' \
  -not -path './.next/*' \
  -not -path './dist/*' \
  -not -path './build/*' \
  | head -200

# 확장자별 파일 수
find . -type f -not -path './.git/*' -not -path './node_modules/*' \
  | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20
```

### 1.2 Context 파일

```
Read CLAUDE.md (없으면 부재 기록)
Glob .claude/rules/*.md → 각각 Read
```

### 1.3 Harness 인벤토리

```bash
cd "$PROJECT_ROOT"

# Skills
find .claude/skills -name "SKILL.md" 2>/dev/null
# 플러그인 스킬도 확인
find .claude-plugin/skills -name "SKILL.md" 2>/dev/null

# Agents
find .claude/agents -name "*.md" 2>/dev/null

# Settings (hooks)
cat .claude/settings.json 2>/dev/null

# MCP
cat .mcp.json 2>/dev/null
cat .claude/.mcp.json 2>/dev/null

# Memory
find . -path "*/memory/*.md" -maxdepth 4 2>/dev/null
cat MEMORY.md 2>/dev/null
```

각 스킬의 frontmatter (첫 30줄)를 읽어 name, description, trigger keywords를 추출한다.

### 1.4 빌드 인프라

```bash
cd "$PROJECT_ROOT"

# Package manager scripts
cat package.json 2>/dev/null | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d.get('scripts',{}), indent=2))" 2>/dev/null

# 기타 빌드 시스템
ls Makefile pyproject.toml Cargo.toml go.mod 2>/dev/null

# CI
ls .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null

# Git
git status --porcelain 2>/dev/null | head -20
git log --oneline -5 2>/dev/null

# Safety
cat .gitignore 2>/dev/null
```

### 1.5 세션 분석 (최근 1주일)

최근 7일간의 세션 JSONL을 분석한다. **프롬프트 내용은 절대 읽지 않는다** — tool_use 메타데이터만 추출한다.

```bash
cd "$PROJECT_ROOT"

python3 << 'PYEOF'
import sys, json, os, glob
from collections import Counter
from datetime import datetime, timedelta, timezone

# 프로젝트 경로를 인코딩하여 세션 디렉토리 찾기
project_dir = os.getcwd()

# ~/.claude/projects/ 아래에서 매칭되는 디렉토리 찾기
projects_base = os.path.expanduser('~/.claude/projects')
session_dir = None

if os.path.isdir(projects_base):
    # 경로를 '-'로 인코딩한 디렉토리명 매칭
    encoded = project_dir.replace('/', '-')
    candidate = os.path.join(projects_base, encoded)
    if os.path.isdir(candidate):
        session_dir = candidate
    else:
        # 부분 매칭 시도
        for d in os.listdir(projects_base):
            full = os.path.join(projects_base, d)
            if os.path.isdir(full) and project_dir.replace('/', '-').lstrip('-') in d:
                session_dir = full
                break

if not session_dir:
    print(json.dumps({"sessions_analyzed": 0, "error": "세션 디렉토리를 찾을 수 없습니다"}))
    sys.exit(0)

# 최근 1주일 필터
one_week_ago = datetime.now(timezone.utc) - timedelta(days=7)
all_sessions = sorted(glob.glob(f'{session_dir}/*.jsonl'), key=os.path.getmtime, reverse=True)
sessions = [s for s in all_sessions if datetime.fromtimestamp(os.path.getmtime(s), tz=timezone.utc) > one_week_ago]

if not sessions:
    print(json.dumps({"sessions_analyzed": 0, "error": "최근 1주일 세션이 없습니다"}))
    sys.exit(0)

tool_counts = Counter()
skills_invoked = set()
agents_invoked = set()
hook_events = Counter()
edit_targets = Counter()
bash_commands = Counter()
session_durations = []

for sf in sessions:
    timestamps = []
    with open(sf) as f:
        for line in f:
            try:
                d = json.loads(line)
                ts = d.get('timestamp')
                if ts:
                    timestamps.append(ts)

                if d.get('type') == 'assistant':
                    content = d.get('message', {}).get('content', []) or []
                    tool_blocks = [b for b in content if isinstance(b, dict) and b.get('type') == 'tool_use']
                    for block in tool_blocks:
                        name = block.get('name', '')
                        tool_counts[name] += 1
                        inp = block.get('input', {})
                        if name == 'Skill':
                            skills_invoked.add(inp.get('skill', '?'))
                        if name == 'Agent':
                            st = inp.get('subagent_type', inp.get('description', '?'))
                            agents_invoked.add(st)
                        if name == 'Edit':
                            fp = inp.get('file_path', '?')
                            edit_targets[fp] += 1
                        if name == 'Bash':
                            cmd = inp.get('command', '')
                            short = cmd.split('&&')[0].split('|')[0].split(';')[0].strip()
                            if short and len(short) < 80:
                                bash_commands[short] += 1

                if d.get('type') == 'system' and d.get('subtype') == 'stop_hook_summary':
                    for hi in d.get('hookInfos', []):
                        hook_events[hi.get('command', '?')] += 1

            except Exception:
                pass

    if len(timestamps) >= 2:
        try:
            t0 = datetime.fromisoformat(timestamps[0].replace('Z', '+00:00'))
            t1 = datetime.fromisoformat(timestamps[-1].replace('Z', '+00:00'))
            dur = (t1 - t0).total_seconds() / 60
            if dur > 0:
                session_durations.append(dur)
        except Exception:
            pass

repeated_edits = {k: v for k, v in edit_targets.items() if v > 3}
avg_dur = round(sum(session_durations) / len(session_durations), 1) if session_durations else 0

result = {
    'sessions_analyzed': len(sessions),
    'period': f'{one_week_ago.strftime("%Y-%m-%d")} ~ {datetime.now().strftime("%Y-%m-%d")}',
    'tool_counts': dict(tool_counts.most_common(20)),
    'skills_invoked': sorted(skills_invoked - {'?'}),
    'agents_invoked': sorted(agents_invoked - {'?'}),
    'hook_events': dict(hook_events),
    'avg_session_duration_min': avg_dur,
    'repeated_edits': dict(sorted(repeated_edits.items(), key=lambda x: -x[1])[:10]),
    'frequent_bash_commands': dict(bash_commands.most_common(15))
}
print(json.dumps(result, indent=2, ensure_ascii=False))
PYEOF
```

### 1.6 CONTEXT_BLOCK 조립

수집한 데이터를 하나의 마크다운 블록으로 조립한다:

```markdown
## Project Structure
{1.1 output}

## CLAUDE.md
{1.2 content or "NOT FOUND"}

## Rules
{1.2 rules or "NONE"}

## Harness Inventory
### Skills
{list with frontmatter summaries}
### Agents
{list}
### Hooks (settings.json)
{content or "NOT CONFIGURED"}
### MCP
{content or "NOT CONFIGURED"}
### Memory
{content or "NOT CONFIGURED"}

## Build Infrastructure
{1.4 output}

## Session Analysis (1 week)
{1.5 JSON output}
```

---

## Phase 2: 체크리스트 평가

`references/checklist.md`를 읽고, CONTEXT_BLOCK의 데이터를 기반으로 35개 항목을 각각 평가한다.

### 평가 기준

각 항목에 대해:
- **PASS**: 증거가 명확히 확인됨
- **FAIL**: 증거가 없거나 부족함
- **PARTIAL**: 일부만 충족 (PASS로 카운트하지 않음)

### 축별 평가 로직

**축 1: 준비 (Scaffolding)** — 프로젝트 구조 + harness 인벤토리 기반

| # | 항목 | 확인 방법 |
|---|------|----------|
| 1 | 디렉토리 구조 파악 가능 | 프로젝트 구조에서 명확한 역할 구분 (src/, tests/ 등) |
| 2 | AI 설명 파일 존재 | CLAUDE.md 존재 여부 |
| 3 | 긴 맥락 분리+참조 | CLAUDE.md에서 @참조 또는 별도 문서 링크 |
| 4 | 민감 파일 보호 | .gitignore에 .env 포함, 또는 hooks에서 .env 보호 |
| 5 | 도메인별 규칙 분리 | .claude/rules/ 에 파일이 2개 이상 |
| 6 | 프로젝트 자동화 존재 | skills 또는 hooks가 1개 이상 |
| 7 | AI 행동 범위 제한 | settings.json에 permission 설정, 또는 PreToolUse 훅 |
| 8 | 신규 참여자 즉시 시작 | CLAUDE.md + rules + skills 조합이 충분 (주관적, 엄격 평가) |

**축 2: 맥락 (Context)** — CLAUDE.md 내용 + 세션 분석 기반

| # | 항목 | 확인 방법 |
|---|------|----------|
| 1 | 반복 지시 설정화 | CLAUDE.md가 비어있지 않고 프로젝트별 내용 포함 |
| 2 | 설정 간결성 | CLAUDE.md 줄 수 < 100, 또는 분리 참조 사용 |
| 3 | 암묵지 명시화 | rules/ 파일에 구체적 행동 지침이 있음 |
| 4 | 정보 범위 구분 | global CLAUDE.md + project CLAUDE.md + rules 3계층 존재 |
| 5 | 도메인 문서화 | CLAUDE.md 또는 docs/에 도메인 용어, 정책 기록 |
| 6 | 외부 시스템 연결 | MCP 설정이 존재 |
| 7 | Progressive Disclosure | rules/ 파일이 글로브 패턴으로 조건부 로드 |

**축 3: 실행 (Orchestration)** — 세션 분석 기반

| # | 항목 | 확인 방법 |
|---|------|----------|
| 1 | 배경+목적+제약 전달 | (세션에서 직접 확인 불가 — 스킬/도구 사용 패턴으로 추론) |
| 2 | 완료 기준 명시 | ralph/DoD 류 스킬 사용 흔적, 또는 테스트 실행 패턴 |
| 3 | 계획 후 실행 | plan 모드 또는 specify/scaffold 스킬 사용 흔적 |
| 4 | 위임 규모 구분 | Agent 호출이 세션에 존재 |
| 5 | 세션 인수인계 | session-wrap 또는 memory 사용 흔적 |
| 6 | 자동 반복 시도 | ralph 류 loop 스킬 사용 흔적 |
| 7 | 커스텀 계획 프로세스 | specify/scaffold/deep-interview 스킬 존재 |
| 8 | 병렬 실행 | Agent 호출 시 run_in_background=true 패턴 |

**축 4: 검증 (Verification)** — 빌드 인프라 + hooks 기반

| # | 항목 | 확인 방법 |
|---|------|----------|
| 1 | 테스트 환경 | package.json scripts에 test, 또는 테스트 프레임워크 설정 |
| 2 | 포맷터/린터 자동 적용 | PostToolUse hooks에 포맷터/린터 |
| 3 | 위험 작업 자동 차단 | PreToolUse hooks 존재 |
| 4 | 되돌릴 수 있는 구조 | git 사용 + 브랜치 전략 흔적 |
| 5 | Dry-run 흐름 | plan 모드 사용 또는 --dry-run 패턴 |
| 6 | 에러 자동 복구 | ralph/loop 류 재시도 구조 |
| 7 | E2E 격리 검증 | docker 또는 테스트 격리 환경 |
| 8 | 만드는 AI/검증 AI 분리 | verifier 에이전트 존재 또는 사용 흔적 |

**축 5: 개선 (Compounding)** — 세션 분석 + harness 인벤토리 기반

| # | 항목 | 확인 방법 |
|---|------|----------|
| 1 | 실수 → 규칙화 | rules/ 파일 수가 2개 이상 |
| 2 | 반복 작업 인지 | (세션에서 반복 bash 명령 3회+ 감지) |
| 3 | 반복 → 자동화 | skills가 2개 이상 |
| 4 | 관찰/추적 가능 | usage-analyze, harness-check 류 스킬 존재 |
| 5 | 안 쓰는 것 정리 | (skills_invoked와 등록된 skills 비교 — 미사용 스킬 있으면 FAIL) |

---

## Phase 3: 리포트 생성

### 3.1 스코어카드 계산

축별로:
- **통과율**: PASS 수 / 전체 항목 수
- **달성 레벨**: L1 항목 전체 PASS → L1 달성, L2 항목 전체 PASS → L2 달성, L3도 동일
- **전체 성숙도**: 5축 중 가장 낮은 달성 레벨

### 3.2 액션 아이템 도출

FAIL/PARTIAL 항목에서 구체적 개선 액션을 도출한다.

분류:
- **Missing** — 아예 없는 것 (CLAUDE.md 없음, hooks 없음)
- **Broken** — 있지만 동작하지 않는 것 (등록된 스킬인데 안 쓰임)
- **Opportunity** — 있으면 좋겠지만 현재는 없는 것 (MCP 연동, 병렬 실행)

### 3.3 Quick Wins

FAIL 항목 중 노력 대비 효과가 가장 큰 3개를 선별한다.

노력 기준:
- **Low**: CLAUDE.md에 한 줄 추가, .gitignore 수정
- **Medium**: 스킬 1개 생성, hooks 설정
- **High**: CI 파이프라인 구축, MCP 서버 연동

### 3.4 리포트 작성

`.harness/check-reports/check-harness-{YYYY-MM-DD}.md`에 저장한다.

---

## Output Format

```markdown
═══════════════════════════════════════════
         HARNESS MATURITY REPORT
═══════════════════════════════════════════

Project: {name}
Path: {PROJECT_ROOT}
Scanned: {date}
Sessions analyzed: {N} ({period})

═══════════════════════════════════════════
  SCORECARD
═══════════════════════════════════════════

Overall Maturity: L{level}

| Axis | Score | Level | Details |
|------|-------|-------|---------|
| 1. Scaffolding (준비) | {pass}/{total} | L{n} | {summary} |
| 2. Context (맥락)     | {pass}/{total} | L{n} | {summary} |
| 3. Orchestration (실행)| {pass}/{total} | L{n} | {summary} |
| 4. Verification (검증) | {pass}/{total} | L{n} | {summary} |
| 5. Compounding (개선)  | {pass}/{total} | L{n} | {summary} |

───────────────────────────────────────────
  DETAILED CHECKLIST
───────────────────────────────────────────

### Axis 1: Scaffolding (준비)

| # | L | Check | Status | Evidence |
|---|---|-------|--------|----------|
| 1 | L1 | 디렉토리 구조 파악 가능 | PASS/FAIL | {근거} |
| 2 | L1 | AI 설명 파일 존재 | PASS/FAIL | {근거} |
| ... | | | | |

### Axis 2: Context (맥락)
...

### Axis 3: Orchestration (실행)
...

### Axis 4: Verification (검증)
...

### Axis 5: Compounding (개선)
...

───────────────────────────────────────────
  FINDINGS ({total}건)
───────────────────────────────────────────

### High Priority ({N}건)

[Missing] **Title**
  Detail.
  -> Action

### Medium Priority ({N}건)
...

### Low Priority ({N}건)
...

───────────────────────────────────────────
  QUICK WINS (Top 3)
───────────────────────────────────────────

1. {action} -- {expected benefit}
2. ...
3. ...

───────────────────────────────────────────
  HARNESS SNAPSHOT
───────────────────────────────────────────

| Component      | Status          |
|----------------|-----------------|
| CLAUDE.md      | ...             |
| Skills         | {N} registered  |
| Agents         | ...             |
| Hooks          | ...             |
| MCP            | ...             |
| Memory         | ...             |
| Tests          | ...             |
| CI             | ...             |
```

---

## Hard Rules

1. **프롬프트 내용 읽지 않는다** — 세션 분석은 tool_use 메타데이터만
2. **프로젝트 파일을 수정하지 않는다** — 리포트만 `.harness/check-reports/`에 작성
3. **증거 기반 평가** — 모든 PASS/FAIL에 구체적 근거를 기록
4. **맥락 인식** — 프로젝트에 불필요한 항목은 "N/A"로 표시 (예: 프론트엔드 없는 프로젝트에 컴포넌트 스킬 제안하지 않음)
5. **Quick Wins는 노력순** — 가장 쉬운 것부터
