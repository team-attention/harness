## L0: Goal

**Output**: spec.md 파일 생성 — `Goal`, `Non-goals`, `Confirmed Goal` 섹션

### Mirror Protocol

Before asking questions, mirror the user's goal:

```
"I understand you want [goal]. Scope: [included / excluded].
 Done when: [success criteria].
 Does this match?"
```

Then `AskUserQuestion`: "Does this match your intent?"

**Rules:**
- Mirror confirms goal, scope, done criteria ONLY. No tech choices (those are L2).
- Include at least one inference beyond the literal request.
- If ambiguous, surface the ambiguity explicitly.
- Max 3 mirror attempts. If still unclear, ask directly.

### Write to spec.md

After user confirms mirror, create the spec file:

```markdown
Write tool → {specDir}/spec.md

# Spec: {name}

## Meta
- **Created**: {date}
- **Type**: dev
- **Status**: drafting

## Goal
{user's original goal}

## Non-goals
- {non_goal_1}
- {non_goal_2}
(or "(none)" if no non-goals)

## Confirmed Goal
{refined goal after mirror protocol}
```

### Gate

User confirms mirror → advance to L1. No reviewer.

---

## L1: Context Research

**Output**: `## Research` 섹션 추가 to spec.md

### Execution

Orchestrator scans the codebase with Glob/Grep/Read to find:
- Existing patterns relevant to the goal
- Project structure, build/test/lint commands
- Internal docs, ADRs, READMEs

For larger codebases, optionally dispatch agents:

```
Agent(subagent_type="code-explorer",
     prompt="Find: existing patterns for [feature type]. Report findings as file:line format.")

Agent(subagent_type="code-explorer",
     prompt="Find: project structure, package.json scripts for lint/test/build commands. Report as file:line format.")

Agent(subagent_type="docs-researcher",
     prompt="Find internal documentation relevant to [feature/task]. Search docs/, ADRs, READMEs, config files for conventions, architecture decisions, and constraints. Report as file:line format.")
```

### Append to spec.md

```markdown
## Research
- {finding_1} (`file:line`)
- {finding_2} (`file:line`)
- ...
```

### Gate

Auto-advance after writing. No reviewer, no user approval at L1.
