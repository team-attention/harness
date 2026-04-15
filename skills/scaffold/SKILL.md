---
name: scaffold
description: |
  Greenfield project architecture + harness scaffolding for AI Agent productivity.
  Interview-driven decisions -> markdown spec output.
  Produces: Code Structure (vertical slice exemplar), Test Infrastructure, Guard Rails,
  conditional extensions, AND Harness (CLAUDE.md with domain/team context, rules, skills, hooks).
  L2: architecture decisions, L3: harness setup, L4: unified plan (requirements + tasks).
  Use when: "/scaffold", "scaffold", "new project", "set up project", "프로젝트 세팅", "초기 구조"
allowed-tools:
  - Read
  - Grep
  - Glob
  - Task
  - Bash
  - Write
  - Edit
  - Agent
  - AskUserQuestion
---

# /scaffold — Greenfield Architecture Scaffolding

Generate a scaffold spec through an architecture-focused derivation chain.
Produces a complete development foundation that AI agents can extend consistently.

All outputs are written as **markdown files** — no external CLI dependencies.

---

## Core Identity

scaffold is specify's **architecture variant**. Same structure, different weight center.

| | specify | scaffold |
|---|---------|----------|
| Focus | What to build (features) | How to structure (architecture + harness) |
| L2 weight | Moderate (feature decisions) | **Heavy** (tech stack, patterns, infra) |
| L3 | Requirements (behavioral) | **Harness** (domain, team, skills, hooks, rules) |
| L4 | Tasks | **Plan** (requirements + tasks unified) |
| Tasks | Feature implementation | Project initialization + exemplar + harness |
| Output | Code changes | Complete development environment + AI harness |
| When | Feature on existing codebase | Greenfield or major restructure |

---

## Output Format

All layer results are written as sections in a single markdown file:

```
{project_dir}/scaffold-spec.md
```

The file accumulates as each layer completes. Structure:

```markdown
# Scaffold Spec: {project name}

Created: {date}
Goal: {one-line goal}

## L0: Goal
...

## L1: Environment
...

## L2: Architecture Decisions
...

## L3: Harness Setup
...

## L4: Plan
...
```

---

## Layer Flow

| Layer | What | Gate |
|-------|------|------|
| L0 | Mirror -> confirmed_goal, non_goals | User confirms mirror |
| L1 | Environment scan (greenfield detection) | Auto-advance |
| L2 | **Architecture interview** -> decisions + constraints (HEAVY) | User approval |
| L3 | **Harness setup** -> domain, team, rules, skills, hooks | User approval |
| L4 | **Plan** -> requirements + tasks (unified from L2+L3) | User approval |

### Session Init (before L0)

Determine the project name from the user's goal. Create the output file:

```bash
touch {project_dir}/scaffold-spec.md
```

---

## L0: Goal

**Output**: Goal section in scaffold-spec.md

### Mirror Protocol

Mirror the user's goal with scaffold-specific framing:

```
"I understand you want to build [product/system].
 Architecture scope: [what the scaffold will set up].
 NOT in scaffold scope: [features, business logic -- those come later via /specify].
 Done when: [agent can extend the codebase consistently].
 Does this match?"
```

**Key distinction**: scaffold's goal is the **foundation**, not the product. If user says "I want to build a todo app", the scaffold goal is "Set up a web application foundation (server + client + DB) that an agent can extend to build features like a todo app."

### Write to spec

After user confirms, write the L0 section:

```markdown
## L0: Goal

**Confirmed Goal**: Set up a [framework] foundation with [key patterns] that an agent can extend to build [product] features

**Non-Goals**:
- Feature implementation
- Production deployment
- [other non-goals from discussion]
```

### Gate

User confirms mirror -> advance to L1.

---

## L1: Environment Scan

**Output**: Environment section in scaffold-spec.md

Unlike specify's L1 (which scans existing code), scaffold's L1 scans the **environment**:

### Scan Targets

| Target | How | Why |
|--------|-----|-----|
| Working directory | `ls -la`, check for existing files | Greenfield confirmation |
| Package managers | `which npm`, `which yarn`, `which pnpm`, `which bun` | Available tooling |
| Runtime versions | `node -v`, `python3 --version`, `go version`, etc. | Compatibility constraints |
| Docker | `docker --version`, `docker compose version` | Infra capability |
| Git | `git status` | Repo state |
| OS/platform | `uname -a` | Platform constraints |

### Write to spec

```markdown
## L1: Environment

- **Directory**: empty (greenfield confirmed) / has existing files
- **Node**: v22.x, pnpm available
- **Docker**: installed, compose v2
- **Git**: initialized, clean
- **Platform**: macOS ARM64
```

### Gate

Auto-advance to L2 (no user approval needed).

---

## L2: Architecture Decisions (HEAVY)

**Output**: Decisions section in scaffold-spec.md

This is scaffold's core. The interview determines the entire project architecture.

### Step 0: Checkpoint Generation

Read L1 environment scan + confirmed goal, then generate checkpoints per **architecture dimension**.

**Complexity classification** -- based on confirmed goal:

| Signal | Examples |
|--------|----------|
| Client-server boundary | web app, mobile + API, microservices |
| Multiple data stores | DB + cache + queue |
| Real-time communication | WebSocket, SSE, polling |
| External service integration | payment, auth provider, AI API |
| Multi-environment deployment | dev/staging/prod, Docker |
| Background processing | workers, cron, queues |

- **Simple** (0-1 signals) -> 2-3 per dimension
- **Medium** (2-3 signals) -> 4-5 per dimension
- **Complex** (4+ signals) -> 6-8 per dimension

### Architecture Dimensions

| # | Dimension | Weight | Example Checkpoints |
|---|-----------|--------|-------------------|
| 1 | **Tech Stack** | 25% | Language/runtime, framework, package manager |
| 2 | **Communication** | 20% | Client-server protocol, API style, type safety strategy |
| 3 | **Data & State** | 20% | Database choice, ORM/query builder, migration strategy, caching |
| 4 | **Testing** | 15% | Test framework, test patterns, coverage strategy |
| 5 | **DevOps & Environment** | 20% | Containerization, CI/CD, env config, deployment target |

**L1 Auto-Resolve**: Check each checkpoint against environment scan. Node installed -> resolve "runtime" checkpoint. Docker available -> partially resolve containerization.

### Interview Loop (score-driven)

Each round:
1. **Score** -- coverage per dimension
2. **Target** -- lowest-scoring dimension(s)
3. **Ask** -- 2 scenario questions targeting those checkpoints (via AskUserQuestion)
4. **Resolve** -- mark covered, record decisions
5. **Scan** -- detect cross-decision tensions
6. **Display** -- scoreboard

**Question format -- RIGHT (concrete scenario):**
```
AskUserQuestion(
  question: "Your API needs to serve both a React frontend and a future mobile app. How should client-server communication work?",
  options: [
    { label: "REST + OpenAPI + code-gen", description: "OpenAPI spec -> Orval/openapi-typescript. Type-safe, well-tooled." },
    { label: "tRPC", description: "End-to-end type safety, no code-gen. TypeScript only." },
    { label: "GraphQL", description: "Flexible queries, schema-first. Higher complexity." },
    { label: "Agent decides", description: "Let scaffold choose based on project context" }
  ]
)
```

**"Agent decides" handling**: When user selects this, scaffold makes an opinionated choice based on:
1. L1 environment capabilities
2. Decisions already made (consistency)
3. Project complexity (simpler for simple projects)
4. Agent productivity criteria (prefer type-safe, convention-over-config)

Record as decision with `assumed: true`.

### Agent Productivity Bias

When making or recommending decisions, bias toward the 4 quality criteria:

| Criteria | Bias |
|----------|------|
| Agent extensibility | Prefer convention-over-config, clear naming, predictable patterns |
| Testability | Prefer dependency injection, pure functions, mockable boundaries |
| Drift resistance | Prefer strict linting, type checking, boundary enforcement |
| Type-safe communication | Prefer code-gen over manual types, schema-first over code-first |

### Conditional Extension Detection

During the interview, detect which conditional extensions to activate:

| Signal from Interview | Extension Activated |
|----------------------|-------------------|
| Client-server boundary detected | **Type Contracts** (OpenAPI/tRPC/GraphQL schema) |
| Database mentioned or implied | **Data Layer** (migrations, connection, seed) |
| Docker available + multi-service | **Docker/Infra** (compose, Dockerfile) |
| Long-running server process | **Runtime Patterns** (health check, graceful shutdown) |

### Termination

Composite score uses **weighted average** across dimensions (weights from the table above).
Terminate when: composite >= 0.80, every dimension >= 0.60, unknowns == 0.

### Inversion Probe

Two architecture-specific questions:

1. **Inversion**: "Given these architecture decisions, what scenario would cause a complete restructure even if every component works individually?"
2. **Implication**: "You chose [most impactful decision]. Does that also mean [architectural consequence]?"

If the probe reveals a critical issue (e.g., contradictory decisions, missing dimension coverage):
- Record the issue as a `known_gap`
- Re-enter the interview loop targeting the affected dimension(s)
- Continue until termination criteria are met again

### L2 Approval

Present all decisions + constraints + activated extensions, then ask user to approve via AskUserQuestion (Approve / Revise / Abort).

### Write to spec

```markdown
## L2: Architecture Decisions

### Decisions

| ID | Decision | Rationale | Assumed? |
|----|----------|-----------|----------|
| D1 | TypeScript + Node.js runtime | User preference, team familiarity | No |
| D2 | REST + OpenAPI for API | Multi-client support, code-gen | No |
| ... | ... | ... | ... |

### Constraints

- C1: pnpm workspace -- always use pnpm, never npm/yarn
- C2: ...

### Activated Extensions

- [x] Type Contracts (OpenAPI + Orval)
- [x] Data Layer (PostgreSQL + Prisma)
- [ ] Docker/Infra (not needed)
- [ ] Runtime Patterns (not needed)

### Known Gaps

- (none, or list gaps found during inversion probe)
```

---

## L3: Harness Setup

**Output**: Harness section in scaffold-spec.md

L3 determines the AI work environment for this project.

### 3-1. Domain Context (interactive)

```
AskUserQuestion(
  question: "Does this project have domain-specific terms or business rules?",
  options: [
    { label: "Yes, I'll describe them", description: "You'll provide domain terms and key business rules" },
    { label: "None yet", description: "Skip -- can add later to CLAUDE.md" },
    { label: "Agent decides", description: "Infer from project goal if possible" }
  ]
)
```

### 3-2. Team Context (interactive)

```
AskUserQuestion(
  question: "Are there team conventions for commits, PRs, branching, or code review?",
  options: [
    { label: "Conventional Commits + GitHub Flow", description: "feat/fix/chore prefixes, feature branches, squash merge" },
    { label: "Trunk-based development", description: "Short-lived branches, no long-running feature branches" },
    { label: "Custom -- I'll describe", description: "You'll specify your team's rules" },
    { label: "Solo project, no conventions", description: "Skip team context" }
  ]
)
```

### 3-3. Constraints -> Rules (auto + confirm)

Scan L2 constraints and propose converting them to `.claude/rules/` files:

```
AskUserQuestion(
  question: "Convert these constraints to .claude/rules/ for automatic enforcement?",
  options: [
    { label: "Yes, all of them", description: "All constraints become rules files" },
    { label: "Let me pick", description: "Choose which constraints to enforce" },
    { label: "Skip", description: "Keep constraints in spec only" }
  ]
)
```

### 3-3.5. CLAUDE.md Structure (Progressive Disclosure)

CLAUDE.md should stay lean — put stable rules in `docs/` and load conditionally with `@` refs.

```markdown
# {project}

{3-5 lines: purpose, stack, directory map}

## Context (loaded on demand)
@docs/domain.md       # business terms, invariants
@docs/policies.md     # team conventions, commit style
@docs/architecture.md # decisions, boundaries
```

Rationale: monolithic CLAUDE.md inflates context every turn. `check-harness` axis 2 (C6) checks `conditional_load_evidence ≥ 1`. Keep the root file under ~80 lines.

### 3-4. Skills Detection (auto-suggest + ask)

**Start minimal.** Only scaffold skills that the L2 tech stack directly requires — dead skills pollute trigger matching (check-harness axis 1). Rule of thumb: *if you can't name 3 times you'd call it next week, don't scaffold it.*

Scan L2 decisions for recurring task patterns and suggest project-specific skills:

| L2 Decision Signal | Auto-Suggested Skill | Description |
|-------------------|---------------------|-------------|
| DB + ORM (Prisma, Drizzle, SQLAlchemy) | `/migrate` | Run migration + regenerate types |
| DB detected | `/seed-data` | Generate development seed data |
| Docker / docker-compose | `/deploy` | Build, push, run with health check |
| API server (REST, GraphQL, tRPC) | `/api-test` | Test endpoint with curl/httpie |
| Async workers (Celery, BullMQ) | `/worker-test` | Dispatch test task + verify result |
| CLI binary (Rust, Go) | `/release` | Version bump + build + tag + publish |
| Frontend framework | `/new-component` | Scaffold component + test + story |
| Payment integration (Stripe, etc.) | `/test-webhook` | Forward + trigger webhook locally |

```
AskUserQuestion(
  question: "These skills will be scaffolded based on your tech stack. Any other tasks you'll repeat frequently?",
  options: [
    { label: "These are enough", description: "Proceed with auto-suggested skills only" },
    { label: "Add more", description: "I'll describe additional recurring tasks" },
    { label: "Skip all skills", description: "Don't generate any project skills" }
  ]
)
```

### 3-5. Hooks Detection (auto + confirm)

Auto-detect hooks from L2 tech stack decisions:

| L2 Decision | Auto-Detected Hook | Type |
|------------|-------------------|------|
| TypeScript (tsconfig.json) | `tsc --noEmit` on Edit/Write to .ts | PostToolUse |
| Prettier configured | `prettier --write` on Edit/Write | PostToolUse |
| ESLint configured | `eslint --fix` on Edit/Write | PostToolUse |
| Ruff / Black (Python) | `ruff format` on Edit/Write | PostToolUse |
| rustfmt (Rust) | `rustfmt` on Edit/Write | PostToolUse |
| gofmt (Go) | `gofmt -w` on Edit/Write | PostToolUse |
| .env files will exist | Block Edit/Write to `.env*` | PreToolUse |
| Lock files will exist | Block Edit/Write to lock files | PreToolUse |

```
AskUserQuestion(
  question: "These hooks will be added to .claude/settings.json. Approve?",
  options: [
    { label: "Approve all", description: "Add all detected hooks" },
    { label: "Let me pick", description: "Choose which hooks to enable" },
    { label: "Skip hooks", description: "Don't set up any hooks" }
  ]
)
```

### Write to spec

```markdown
## L3: Harness Setup

### Domain Context
{domain terms and rules, or "None"}

### Team Conventions
{commit style, branching strategy, or "Solo project"}

### Rules (from Constraints)
| Constraint | Rule File | Status |
|-----------|-----------|--------|
| C1: pnpm only | `.claude/rules/pnpm-only.md` | Approved |
| ... | ... | ... |

### Skills
| Skill | Description | Source |
|-------|-------------|--------|
| `/migrate` | Run migration + regenerate types | D3 (Prisma) |
| ... | ... | ... |

### Hooks
| Hook | Type | Trigger |
|------|------|---------|
| `tsc --noEmit` | PostToolUse | Edit/Write .ts files |
| ... | ... | ... |
```

### L3 Gate

Present harness summary -> AskUserQuestion (Approve / Revise / Abort).

---

## L4: Plan (Requirements + Tasks)

**Output**: Plan section in scaffold-spec.md

L4 unifies requirements derivation and task generation in one step.

### Step 1: Derive Requirements

**Code Requirements (from L2):**

```
R1: "Code Structure -- Project directories, base configs, and a complete vertical slice exemplar"
  R1.1: "Directory structure follows [framework] conventions with clear layer separation"
  R1.2: "Vertical slice exemplar implements one complete flow (route -> service -> data -> test) with importable utilities (logger, config, errors)"
  R1.3: "All exemplar utilities are importable modules, not inline code"

R2: "Test Infrastructure -- Framework setup with patterns matching the exemplar"
  R2.1: "[Test framework] configured with [runner] and example test matching exemplar flow"
  R2.2: "Test directory structure mirrors source structure"

R3: "Guard Rails -- CLAUDE.md + enforcement mechanisms for drift resistance"
  R3.1: "CLAUDE.md (lean — under ~80 lines) with architectural rules, domain context, team conventions, dependency direction, file placement conventions, available project skills summary, and active hooks summary. Stable long-form content lives in `docs/` and loads via Progressive Disclosure `@docs/*.md` references."
  R3.2: "Linter + formatter configured with project-specific rules"
  R3.3: "CI pipeline running lint + typecheck + test"
  R3.4: ".env.example with all required environment variables documented"
```

**Conditional Code Extensions (from L2):**

```
R4: "Type Contracts -- Schema-driven type safety across client-server boundary" (if activated)
R5: "Data Layer -- Database connection, schema management, and seed data" (if activated)
R6: "Docker/Infra -- Containerized local development environment" (if activated)
R7: "Runtime Patterns -- Production-readiness baseline for long-running server" (if activated)
```

**Harness Requirements (from L3):**

```
R8: "Project Rules -- Constraints converted to .claude/rules/" (if approved)
R9: "Domain Skills -- Project-specific repeatable task recipes" (if approved)
R10: "Project Hooks -- Automated code quality enforcement" (if approved)
```

### Step 2: Derive Tasks

**Task DAG:**

```
T1: Project initialization (package.json, tsconfig, base configs)
    fulfills: [R1]

T2: Guard Rails setup (CLAUDE.md, lint, format, CI, .env.example, .claude/rules/)
    fulfills: [R3] + [R8 if approved]
    depends_on: [T1]

T4: Test infrastructure (test config, test dirs, path aliases -- framework setup only)
    fulfills: [R2]
    depends_on: [T1]

T3: Vertical slice exemplar (THE reference implementation + exemplar test)
    fulfills: [R1]
    depends_on: [T2, T4]
    <-- THIS IS THE MOST IMPORTANT TASK

--- Conditional code tasks (parallel where possible) ---

T5: Type Contracts setup (if R4)     fulfills: [R4]  depends_on: [T3]
T6: Data Layer setup (if R5)         fulfills: [R5]  depends_on: [T1]
T7: Docker/Infra setup (if R6)       fulfills: [R6]  depends_on: [T1]
T8: Runtime Patterns (if R7)         fulfills: [R7]  depends_on: [T3]

--- Harness tasks ---

T_SKILL: Domain Skills generation (if R9)
    fulfills: [R9]  depends_on: [T3]

T_HOOK: Project Hooks setup (if R10)
    fulfills: [R10] depends_on: [T1]

TF: Scaffold verification
    depends_on: all above
```

### T3: Vertical Slice Exemplar (Critical Task)

The exemplar is the scaffold's highest-value output. It must demonstrate:

1. **The complete flow** -- from entry point to data layer and back
2. **Importable utilities** -- `lib/logger.ts`, `lib/config.ts`, `lib/errors.ts` (not inline)
3. **The naming convention** -- how files, functions, and variables are named
4. **The test pattern** -- how to test this flow
5. **Error handling** -- how errors propagate through layers
6. **Type safety** -- how types flow across boundaries

The exemplar answers: "If an agent reads only this one feature, can it build the next feature correctly?"

### TF: Scaffold Verification

```
Steps:
- Build: all build/lint/typecheck commands pass
- Tests: all exemplar tests pass
- CLAUDE.md: includes architectural rules + domain context + team conventions + available skills + active hooks
- Rules: .claude/rules/ files match selected constraints from L3
- Exemplar: vertical slice is complete (entry -> data -> response -> test)
- Utilities: logger, config, errors are importable and used in exemplar
- Skills: each generated skill has valid SKILL.md with project-specific commands
- Hooks: .claude/settings.json hooks reference correct tool commands
- Agent test: could an agent read this codebase AND its harness and build a new feature consistently?
```

### Write to spec

```markdown
## L4: Plan

### Requirements

| ID | Requirement | Source | Conditional? |
|----|------------|--------|-------------|
| R1 | Code Structure | L2 | No |
| R2 | Test Infrastructure | L2 | No |
| R3 | Guard Rails | L2 | No |
| R4 | Type Contracts | L2 | If activated |
| ... | ... | ... | ... |

### Task DAG

| ID | Task | Fulfills | Depends On | Status |
|----|------|----------|------------|--------|
| T1 | Project init | R1 | - | pending |
| T2 | Guard Rails | R3, R8 | T1 | pending |
| T4 | Test infra | R2 | T1 | pending |
| T3 | Vertical slice exemplar | R1 | T2, T4 | pending |
| ... | ... | ... | ... | ... |
| TF | Verification | - | all | pending |

### Quality Criteria

- Agent extensibility: vertical slice exemplar (T3)
- Testability: test infrastructure + exemplar tests (T4)
- Drift resistance: CLAUDE.md + rules + lint + CI (T2)
- Type safety: [type contract strategy] (T5)
- Cross-session continuity: CLAUDE.md with domain/team context (T2)
- Task automation: domain skills (T_SKILL)
- Code quality enforcement: project hooks (T_HOOK)
```

### L4 Gate

Present full plan summary, then:

```
AskUserQuestion(
  question: "Review the scaffold plan above.",
  options: [
    { label: "Approve", description: "Plan looks good" },
    { label: "Revise architecture (L2)", description: "Change architecture decisions" },
    { label: "Revise harness (L3)", description: "Change harness setup" },
    { label: "Revise plan (L4)", description: "Adjust requirements or tasks" },
    { label: "Abort", description: "Stop" }
  ]
)
```

---

## Final Step: Next Action

After the user approves the full plan at L4, ask:

```
AskUserQuestion(
  question: "scaffold-spec.md is ready. What would you like to do?",
  options: [
    { label: "Implement now", description: "Start scaffolding the project based on this spec" },
    { label: "Later", description: "Save the spec and implement in a future session" }
  ]
)
```

- If **Implement now**: proceed with task execution following the Task DAG order.
- If **Later**: confirm the spec file location and end.

### After Implementation: Establish Harness Baseline

Once TF (verification) passes, suggest running `/check-harness` to establish a maturity baseline for the new project. This closes the scaffold → audit → compound loop.

```
AskUserQuestion(
  question: "Run /check-harness now to baseline the new harness?",
  options: [
    { label: "Yes, baseline now", description: "Audit axes 1-6 and save report to .harness/check-reports/" },
    { label: "Skip", description: "Baseline later" }
  ]
)
```

---

## User Approval Protocol

Three approval gates (L2, L3, L4). Same pattern at each:

```
AskUserQuestion(
  question: "Review the {items} above. Ready to proceed?",
  options: [
    { label: "Approve", description: "Looks good -- proceed to next layer" },
    { label: "Revise", description: "I want to change something" },
    { label: "Abort", description: "Stop specification" }
  ]
)
```

---

## Checklist Before Stopping

- [ ] scaffold-spec.md exists in project directory
- [ ] confirmed_goal is architecture-framed (not feature-framed)
- [ ] Non-goals include "feature implementation" or similar
- [ ] L2: decisions cover all 5 architecture dimensions
- [ ] L2: Conditional extensions detected and recorded
- [ ] L3: Applicable harness decisions (domain, team, rules, skills, hooks)
- [ ] L3: Constraints -> rules conversion offered to user
- [ ] L3: Skills auto-suggested from tech stack + user input
- [ ] L3: Hooks auto-detected from formatter/linter choices
- [ ] L4: Requirements include Code (R1-R3) + Conditional (R4-R7) + Harness (R8-R10)
- [ ] R1 includes mandatory vertical slice exemplar requirement
- [ ] R3.1 CLAUDE.md includes domain context, team conventions, available skills, active hooks
- [ ] T3 (exemplar) includes importable utilities (logger, config, errors)
- [ ] T_SKILL produces skills with project-specific commands (not generic placeholders)
- [ ] T_HOOK produces hooks matching actual L2 tooling decisions
- [ ] TF includes agent extensibility + harness check
- [ ] Plan Summary presented to user
- [ ] Final AskUserQuestion: "Implement now" vs "Later"
