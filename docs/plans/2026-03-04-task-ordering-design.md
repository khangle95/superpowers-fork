# Task Ordering — Design Document

**Date:** 2026-03-04
**Status:** Approved — simplified

## Problem

The `writing-plans` skill has no guidance for task ordering. It tells plan writers *what format* to use but not *what order* to put tasks in. The executor (`subagent-driven-development`) blindly follows whatever order was written. If the plan puts Task 5 before its dependency Task 2, execution breaks.

Additionally:
- `TaskCreate` has `addBlockedBy`/`addBlocks` fields but neither execution skill uses them
- No verification between logical groups of tasks
- Security/auth tasks are often placed last, but should come early

## Solution: Ordering Rules + Dependency Annotations + Layer Gates

Simple, practical rules added to existing skills. No new agents, no new methodology.

### Task Ordering Rules

Added to `writing-plans/SKILL.md`:

```
## Task Ordering

Order tasks bottom-up:
1. Data & types first — schema, interfaces, configs, environment setup
2. Business logic & auth second — services, middleware, validation, permission checks
3. Wiring third — API routes, event handlers, external contracts, skill references
4. User-facing last — UI components, CLI output, formatting, templates
5. Tests & docs at the end — integration tests, documentation, cleanup

Security/auth goes in step 2, not last.
If task B imports/uses something task A creates → A before B.
If unsure, put it earlier rather than later.
```

### Dependency Annotations

Each task gets a short annotation enabling future parallel execution:

```
Mark each task:
- (Independent) — doesn't need any same-layer task's output, safe to run in parallel
- (Depends on: Task N) — needs a specific earlier task done first

These annotations are used by the executor for scheduling.
Sequential execution today; parallel when worktree isolation exists.
```

**Why keep these if execution is sequential today?**
- They're the data a future parallel executor needs to know what's safe to run concurrently
- Without them, enabling parallel execution later requires re-analyzing every plan
- They also help the plan writer think about dependencies explicitly

### Layer Gates

After finishing all tasks in a layer, verify before moving on:

```
### Layer Gate: [Layer] Complete
Run: [type check or test command]
If failing: Fix before proceeding to next layer.
```

Not every layer needs a gate. Use judgment:
- After data & types: Almost always (everything depends on these)
- After business logic: If complex logic or auth is involved
- After wiring: If API contracts matter
- After user-facing: No gate needed (tests & docs IS the gate)

### Plan Format

Tasks get a layer annotation in their header:

```markdown
### Task 1: Database schema for users
...

### Task 2: Shared TypeScript types
...

### Layer Gate: Data & Types Complete
Run: `tsc --noEmit`
If failing: Fix before proceeding.

---

### Task 3: User service (Independent)
...

### Task 4: Auth middleware (Independent)
...

### Task 5: Permission rules (Depends on: Task 4)
...

### Layer Gate: Business Logic Complete
Run: `pnpm test -- --filter=core`
If failing: Fix before proceeding.

---

### Task 6: User API routes (Depends on: Task 3, Task 4)
...
```

### Executor Changes

**subagent-driven-development changes:**
- When hitting a `### Layer Gate` step: run the verification command
- If layer gate fails: STOP. Do not proceed. Report to user.
- `Depends on` annotations: read for context when providing scene-setting to implementer subagents
- `(Independent)` annotations: informational today, used for parallel scheduling later

**No parallel execution today.** Current infrastructure uses a single shared worktree. Parallel agents = file conflicts. Parked for a future design with per-task worktree isolation.

### What This Does NOT Change

- Task template format (test → implement → verify → commit) — unchanged
- Executor dispatch model (one subagent at a time) — unchanged
- Review cycle (spec review → quality review) — unchanged
- Coverage section format — unchanged
- Feature Summary format — unchanged

## Feature Summary

> **For planning:** Use this table as your checklist. Every item must appear in the plan's ## Coverage section as either Planned or Deferred.

| # | Feature | Section |
|---|---------|---------|
| F1 | Bottom-up ordering rules in writing-plans | Task Ordering Rules |
| F2 | Dependency annotations (Independent, Depends on) | Dependency Annotations |
| F3 | Layer gate verification steps | Layer Gates |
| F4 | Updated plan format with layer gates and annotations | Plan Format |
| F5 | Executor layer gate enforcement | Executor Changes |

**Total: 5 features**

## RRI-T Findings Addressed

| Finding | How Addressed |
|---------|---------------|
| No task ordering framework | Bottom-up ordering rules |
| Parallel execution unsafe | Sequential only; annotations preserve parallel metadata |
| No quality gates between layers | Layer gate verification |
| Intra-layer ordering unaddressed | Sequential + explicit Depends on |
| Security controls placed last | Auth in layer 2 (business logic), not last |
| No file collision prevention | Sequential execution; parallel deferred |

## Deferred

| Item | Reason |
|------|--------|
| Parallel wave execution | Requires per-task worktree isolation |
| File ownership registry | Only needed for parallel execution |
| Mid-execution replanning | Separate design concern |
