---
name: specify
description: |
  Turn a goal into an implementation plan (spec.md).
  Simplified layer chain: L0:Goal → L1:Context → L2:Decisions → L3:Requirements → L4:Tasks.
  Evidence-based clarity scoring at L2. User approves at L2, L3, L4.
  Output is a single spec.md file written with the Write tool.
  Use when: "/specify", "specify", "plan this"
allowed-tools:
  - Read
  - Grep
  - Glob
  - Agent
  - Bash
  - Write
  - AskUserQuestion
---

# /specify — Spec Generator

Generate a spec.md through a structured derivation chain.
Each layer builds on the previous — no skipping, no out-of-order writes.

---

## Core Rules

1. **Write tool is the writer** — All spec output goes to `{specDir}/spec.md` via the Write tool. No CLI, no JSON.
2. **Append, don't overwrite** — Read existing spec.md before writing. Append new sections or update specific sections in place using Edit.
3. **Reference before writing** — Read the reference file for the current layer (`references/L*`) before constructing content.
4. **Validate at layer transitions** — After writing each layer, read spec.md and verify required sections exist.
5. **One layer at a time** — Complete and validate each layer before advancing.

---

## Spec Directory

The spec is written to a project-relative directory:

```
specs/{name}/spec.md
```

`{name}` = kebab-case derived from the goal.

Create the directory at Session Init:
```bash
mkdir -p specs/{name}
```

---

## Layer Flow

Execute layers sequentially. Read each reference file just-in-time.

| Layer | Read Reference | What | Gate |
|-------|----------------|------|------|
| L0 | `references/L0-L1-context.md` | Mirror → Goal, Non-goals, Confirmed Goal | User confirms mirror |
| L1 | (same file) | Codebase research → Research section | Auto-advance |
| L2 | `references/L2-decisions.md` | Interview → Decisions + Constraints | Self-validate + L2-reviewer + User approval |
| L3 | `references/L3-requirements.md` | Derive requirements + sub from decisions | Self-validate + User approval |
| L4 | `references/L4-tasks.md` | Derive tasks + external deps, Plan Summary | Self-validate + User approval |

### Session Init (before L0)

```bash
mkdir -p specs/{name}
```

Then create spec.md with initial content via Write tool.

---

## User Approval Protocol

Three approval gates (L2, L3, L4). Each uses the same pattern:

```
AskUserQuestion(
  question: "Review the {items} above. Ready to proceed?",
  options: [
    { label: "Approve", description: "Looks good — proceed to next layer" },
    { label: "Revise", description: "I want to change something" },
    { label: "Abort", description: "Stop specification" }
  ]
)
```

- **Approve** → advance to next layer
- **Revise** → user provides corrections, update spec.md sections, re-present (loop until approved)
- **Abort** → stop

---

## Checklist Before Stopping

- [ ] spec.md at `specs/{name}/spec.md`
- [ ] `## Goal` section populated
- [ ] `## Confirmed Goal` section populated
- [ ] `## Non-goals` section populated (or "(none)")
- [ ] `## Research` section populated
- [ ] `## Decisions` section populated with at least one decision
- [ ] `## Constraints` section populated (or "(none)")
- [ ] `## Requirements` section with every requirement having at least 1 sub-requirement with GWT
- [ ] `## Tasks` section with every task having `Fulfills` linking to requirements
- [ ] Plan Summary presented to user
- [ ] `Approved by` and `Approved at` written to Meta section after final approval
