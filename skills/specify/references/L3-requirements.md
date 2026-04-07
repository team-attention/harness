## L3: Requirements + Sub-requirements

**Output**: `## Requirements` 섹션 추가 to spec.md

### Step 1: Derive from decisions

Read the Decisions section from spec.md. Generate requirements:
- `R0` from confirmed goal (the top-level "what")
- `R1`...`Rn` — one per decision (or group of related decisions)

**Coverage is 100% from the start.** No orphan decisions.

### Step 2: Reshape + Fill behaviors

The initial 1:1 decision→requirement mapping is rarely the final structure. Freely reorganize:

- **Split**: One decision often needs multiple requirements (e.g., D1:"JWT auth" → R1:login, R2:token refresh, R3:token expiry)
- **Merge**: Multiple decisions may combine into one requirement (e.g., D1+D2 → R1:password security)
- **Add**: Create new requirements for behaviors not tied to any single decision
- **Delete**: Remove requirements that are redundant after reorganization

### GWT (Given/When/Then) Rule

Every sub-requirement MUST include `Given`, `When`, and `Then` fields. The behavior line serves as a **one-line summary** of the GWT scenario.

- **Behavior** — one-line summary (required, used as fallback display)
- **Given** — precondition / initial state
- **When** — trigger / action performed
- **Then** — observable outcome / expected result

**Behavior quality rules:**
- BANNED: "correctly", "properly", "works", "as expected", "handles" (without what)
- REQUIRED: trigger (who/what initiates) + observable outcome
- BAD: `"Login works correctly"`
- GOOD: `"Valid login returns JWT"` with Given/When/Then details

**Sub-requirement = behavioral acceptance criterion (GWT format):**
- Each sub-req IS an acceptance criterion for the parent requirement
- Tasks that fulfill this requirement must satisfy ALL sub-req GWT scenarios
- **Atomic** (single trigger, single outcome) → 1 sub-req with 1 GWT
- **Compound** (multiple paths) → happy path + error + boundary conditions, each with its own GWT

### Boundary Decomposition Rule

When a single requirement spans multiple implementation boundaries (API↔UI, Service↔Consumer, Producer↔Subscriber), decompose sub-requirements **per boundary**. Each side of a boundary must have its own sub-req with its own GWT.

Principle: if an artifact exists on one side of a boundary, the counterpart that produces or consumes it on the other side MUST also exist as a sub-req (unless it is admin-only or internal-only).

BAD — mixed layers in one sub-req:
```
### R1: Project CRUD
- R1.1: User can create a project
- R1.2: User can delete a project
```

GOOD — boundary-separated (fullstack: API↔UI):
```
### R1: Project CRUD

#### R1.1: Create project via API
- **Given**: Authenticated user with valid session
- **When**: POST /api/projects with name and description
- **Then**: Returns 201 with created project JSON including id

#### R1.2: List projects via API
- **Given**: Two projects exist for the user
- **When**: GET /api/projects
- **Then**: Returns 200 with array of 2 project objects

#### R1.3: Delete project via API
- **Given**: Project with id=42 exists
- **When**: DELETE /api/projects/42
- **Then**: Returns 204 and project is removed from database

#### R1.4: Frontend renders project list
- **Given**: GET /api/projects returns 2 projects
- **When**: User navigates to project list page
- **Then**: Page renders 2 project cards with name and description

#### R1.5: Frontend delete removes project
- **Given**: Project list page shows project id=42
- **When**: User clicks delete button on project id=42
- **Then**: DELETE /api/projects/42 is called and project disappears from list
```

GOOD — boundary-separated (API↔Worker):
```
### R1: Order processing

#### R1.1: Create order returns job ID
- **Given**: Valid order payload with items
- **When**: POST /orders
- **Then**: Returns 202 with job_id in response body

#### R1.2: Worker processes order event
- **Given**: order.created event is published to queue
- **When**: Worker consumes the event
- **Then**: Order status transitions to 'processing' and inventory is decremented

#### R1.3: Order status is queryable
- **Given**: Order id=99 has been processed by worker
- **When**: GET /orders/99
- **Then**: Returns 200 with status='completed'
```

### Write to spec.md

Append/replace the Requirements section in spec.md:

```markdown
## Requirements

### R0: {goal-level requirement}

#### R0.1: {sub behavior}
- **Given**: {precondition}
- **When**: {trigger}
- **Then**: {outcome}

### R1: {requirement from D1}

#### R1.1: {sub behavior}
- **Given**: {precondition}
- **When**: {trigger}
- **Then**: {outcome}

#### R1.2: {sub behavior}
- **Given**: {precondition}
- **When**: {trigger}
- **Then**: {outcome}

...
```

### Coverage checks (agent self-check before approval)
- Every decision has at least one requirement tracing back to it
- Sub-requirements together cover the full behavior of the parent
- No orphan decisions
- **Boundary check**: if a sub-req implies a cross-boundary dependency (e.g., an API endpoint), verify the other side has a matching sub-req
- **GWT completeness check**: every sub-req has all three GWT fields filled (Given, When, Then)

### L3 Approval

Print ALL requirements and sub-requirements as text (show everything, do not truncate), then AskUserQuestion (Approve/Revise/Abort).

### L3 Gate

Read spec.md and verify:
- Every requirement has at least 1 sub-requirement
- Every sub-requirement has Given/When/Then
- Every decision is covered by at least one requirement

Pass → advance to L4.
