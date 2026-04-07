## L4: Tasks + External Dependencies + Plan Summary

**Output**: `## Tasks`, `## External Dependencies` 섹션 추가 to spec.md

### Step 1: Derive from requirements

Read the Requirements section from spec.md. Generate task stubs:
- `T1`...`Tn` — one per requirement, with `Fulfills` linking to requirements
- No separate TF (Final Verify) task — holistic verification is handled in the execute pipeline

**Coverage is 100% from the start.** No orphan requirements.

### Step 2: Restructure into Vertical Slices

The stubs are a **starting point**. Restructure into **vertical slices** — each task delivers a user-visible feature end-to-end (BE + FE + connection verification).

#### Splitting Principle: Vertical Slice First

A task = BE endpoint + FE UI + the connection between them.
One task must **complete the interface internally** — the producer and consumer of an API live in the same task.

Horizontal splits (BE-only / FE-only) are allowed ONLY for **shared infrastructure**:
- DB schema, common middleware, adapter patterns, shared utilities
- These have no 1:1 mapping to a specific UI

```
BAD (horizontal — interface mismatch risk):
  T1: All backend APIs (projects CRUD + lyrics + generate + export)
  T2: All frontend pages
  → Parallel execution → schema mismatch between T1 and T2

GOOD (vertical slices):
  T1: Scaffolding (DB, router, common config)           ← horizontal, infra
  T2: Adapter pattern (ABC + Factory + rate limiter)     ← horizontal, infra
  T3: Project creation flow (POST /projects + new page)  ← vertical slice
  T4: Lyrics pipeline (WhisperX + LRC parser, internal)  ← BE-only service, no UI yet
  T5: Sync editor (PATCH /projects/:id + editor UI + Save roundtrip) ← vertical slice
  T6: Video generation + progress (BE pipeline + FE progress + WS)   ← vertical slice
  T7: Preview + Export (BE composition + FE preview/export + download) ← vertical slice
```

#### Parallelism Rule

Two tasks can run in parallel ONLY when ALL three conditions hold:
1. **No file overlap**: they don't modify the same files or directories
2. **No interface dependency**: one's output is not the other's input
3. **No model dependency**: they don't produce+consume the same DB table or API endpoint

If any condition is violated → `Depends on` is mandatory.

**File overlap check**: Before finalizing dependencies, mentally list the files each task will likely touch (routes, models, configs, shared modules). If two tasks touch the same file — even for independent changes (e.g., both add routes to the same router file) — they must be serialized via depends_on. Parallel workers editing the same file causes merge conflicts.

**Maximize parallelism** within these constraints — don't add false dependencies.
The goal is a wide DAG of independent vertical slices, not a linear chain.

```
GOOD parallelism:
  T3: Project creation flow     ]
  T4: Lyrics pipeline (service) ] → parallel (no shared interface)
  T5: Sync editor → depends_on: [T3, T4] (uses Project model + lyrics data)

BAD parallelism:
  T3: Backend project API  ]
  T4: Frontend project UI  ] → parallel but SHARE the same API contract
```

#### When Horizontal Split Is Acceptable

A task may be BE-only or FE-only when:
- **Pure infrastructure**: DB models, adapter patterns, shared config (no UI counterpart)
- **Internal service**: processing logic not yet exposed via API (e.g., WhisperX extraction)
- **Pure UI component**: a component that calls an API already built and verified in a prior task

In the third case, the task must have `Depends on` pointing to the task that built the API.

### Write to spec.md

Append the Tasks section to spec.md:

```markdown
## Tasks

### T1: {action} [infra]
- **Fulfills**: R0
- **Depends on**: (none)

### T2: {action} [vertical]
- **Fulfills**: R1
- **Depends on**: T1

### T3: {action} [vertical]
- **Fulfills**: R2
- **Depends on**: T1  ← parallel with T2

### T4: {action} [vertical]
- **Fulfills**: R3, R4
- **Depends on**: T2, T3
```

**Task rules:**
- Every work task: `Fulfills` linking to requirements
- `Depends on` for ordering. No circular dependencies.
- Acceptance criteria = sub-req behaviors from fulfills (no separate AC field — Worker reads requirements directly)
- Build/lint/typecheck = Worker runs these automatically
- Agent may consolidate: merge T1+T2 into one task that fulfills both R1 and R2

### External Dependencies

Scan tasks and decisions for actions outside of code.

```markdown
## External Dependencies

### Pre-work
- {action needed before coding starts}
(or "(none)")

### Post-work
- {action needed after coding completes}
(or "(none)")
```

### L4 Gate

Read spec.md and verify:
- Every requirement is fulfilled by at least one task
- No circular dependencies in task DAG
- External dependencies section exists

### Plan Summary

After gate passes, present the full plan:

```
spec.md ready! {specDir}/spec.md

Goal
────────────────────────────────────────
{confirmed goal}

Non-goals
────────────────────────────────────────
{non_goals or "(none)"}

Key Decisions ({n} total)
────────────────────────────────────────
D1: {decision}
D2: {decision}

Requirements ({n} total, {m} sub-requirements)
────────────────────────────────────────
R1: {behavior}
  R1.1: {sub behavior}
  R1.2: {sub behavior}

Known Gaps
────────────────────────────────────────
{known_gaps or "(none)"}

Pre-work
────────────────────────────────────────
{pre_work items or "(none)"}

Tasks (DAG)
────────────────────────────────────────
T1: {action} [infra] — pending
T2: {action} [vertical] — pending (depends: T1)
T3: {action} [vertical] — pending (depends: T1)    ← parallel with T2
T4: {action} [vertical] — pending (depends: T2, T3)

Post-work
────────────────────────────────────────
{post_work items or "(none)"}
```

### Final Approval

```
AskUserQuestion(
  question: "Review the plan above.",
  options: [
    { label: "Execute", description: "Start implementation" },
    { label: "Revise requirements (L3)", description: "Go back to L3" },
    { label: "Revise tasks (L4)", description: "Adjust task breakdown" },
    { label: "Abort", description: "Stop" }
  ]
)
```

On approval, write `Approved by: user` and `Approved at: {date}` to spec.md Meta section.
