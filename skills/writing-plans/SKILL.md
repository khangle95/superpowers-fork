---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

<HARD-GATE>
The plan MUST have a ## Coverage section with 0 unaccounted items before proceeding to the phase gate. The independent coverage agent must return PASS. Do NOT present the phase gate until coverage is verified.
</HARD-GATE>

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use super-bear:executing-plans to implement this plan task-by-task.

**Goal:** [One sentence describing what this builds]
**Design:** [link to docs/plans/YYYY-MM-DD-<topic>-design.md]
**Coverage:** [N planned, N deferred, 0 unaccounted]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Feature Summary — Read First

Before writing any tasks, read the design doc's **## Feature Summary** section. This is a compact table (10-20 lines) listing all features the design covers. Use it as your planning checklist.

**Process:**
1. Read the Feature Summary table from the design doc (use Explore subagent for the doc, but the Feature Summary is small enough to keep in main context)
2. Plan tasks for each feature in the summary
3. After all tasks are written, write the Coverage section (see below)

For large designs (500+ lines): load one design section at a time while keeping the Feature Summary table in context throughout. This prevents context exhaustion while ensuring no features are missed.

## Coverage Section — Write Last

After all tasks are written, add a **## Coverage** section at the end of the plan. The Coverage section includes a self-describing instruction so any agent reading the plan knows what it is:

~~~markdown
## Coverage
> **Verification:** An independent coverage agent checks this table against the Feature Summary. Every item must be Planned or Deferred. 0 unaccounted required.

(Source: [design doc filename], Feature Summary)

| F# | Feature | Status | Tasks |
|----|---------|--------|-------|
| F1 | [name] | Planned | Task 1-3 |
| F2 | [name] | Deferred — [reason] | — |

**N planned, N deferred, 0 unaccounted**
~~~

**Rules:**
- The `> **Verification:**` instruction line makes the artifact self-describing — any agent reading the plan knows the Coverage section is verified by an independent agent
- Every Feature Summary item MUST have an entry (Planned or Deferred)
- Zero unaccounted items required
- Deferred items must have a reason
- If this is a V2+ plan, include carried-forward deferrals from the previous plan's Coverage section

## Deferral Carry-Forward (V2+)

If the design doc references a previous design or was produced by a sub-feature/redesign triage:
1. Read the previous plan's ## Coverage section (use Explore subagent)
2. Find deferred items that are NOT in the current design's Feature Summary
3. Include them in the Coverage section as: `Deferred — [original reason] (carried from [previous plan])` or `Dropped — [reason]`
4. The Coverage section must be self-contained — the latest plan always has the full picture

## Task Ordering

Order tasks bottom-up:
1. **Data & types first** — schema, interfaces, configs, environment setup
2. **Business logic & auth second** — services, middleware, validation, permission checks
3. **Wiring third** — API routes, event handlers, external contracts, skill references
4. **User-facing last** — UI components, CLI output, formatting, templates
5. **Tests & docs at the end** — integration tests, documentation, cleanup

Security/auth goes in step 2, not last.
If task B imports/uses something task A creates → A before B.
If unsure, put it earlier rather than later.

### Dependency Annotations

Mark each task:
- **(Independent)** — doesn't need any same-layer task's output, safe to run in parallel later
- **(Depends on: Task N)** — needs a specific earlier task done first

Example: `### Task 5: Permission rules (Depends on: Task 4)`

These annotations enable future parallel execution. Sequential today.

### Layer Gates

After finishing all tasks in a layer, add a verification step:

```markdown
### Layer Gate: [Layer] Complete
Run: [type check or test command]
If failing: Fix before proceeding.
```

Not every layer needs a gate. Use judgment:
- After data & types: Almost always (everything depends on these)
- After business logic: If complex logic or auth is involved
- After wiring: If API contracts matter
- After user-facing: No gate needed (tests & docs IS the gate)

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

**Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

**Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## Remember
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD, frequent commits
- **Context efficiency** — All `.md` files under `docs/` must be read via Explore subagent (returns only relevant sections). Source code: native Claude Code behavior.

## RRI-T Plan Review

After drafting the plan, run PLAN_REVIEW as an **agent team**. This is the critical quality gate — the last checkpoint before code gets written.

**Why agent team (not fire-and-forget)?** PLAN_REVIEW reviews a document, not code. Personas may flag issues that the codebase already handles. A Triage Analyst investigates the codebase and debates with personas to resolve false positives. This keeps the debate off your context window.

### Step-by-step:

**1. Create agent team** named `rri-t-{uid}` (reuse UID from DISCOVER if exists).

**2. Create tasks** in the team task list:
- One task per persona: "Review plan as [Persona]"
- One task for Triage Analyst: "Investigate and resolve FAIL findings" (blocked by all 5 persona tasks)

**3. Spawn 6 team members:**

5 personas — use templates from `skills/rri-t/persona-prompts/`. Append these instructions AFTER the template content:
```
--- PLAN_REVIEW INSTRUCTIONS (appended by orchestrator) ---
- You have NO codebase access. Review the plan document only.
- Find problems, not solutions. Describe what's wrong and why.
  A Triage Analyst with codebase access will handle fixes.
- A Triage Analyst may challenge your FAIL findings with codebase
  evidence. Be ready to defend with reasoning from the plan/design.
- After review, create one task per FAIL finding in the team task list
  (tagged with your persona name), then mark your review task complete.
- Previous findings (if any): {PREVIOUS_FINDINGS}
```

Placeholders for PLAN_REVIEW personas:
- `{MODULE_NAME}` — feature name
- `{PHASE}` — "PLAN_REVIEW"
- `{ARTIFACT_CONTENT}` — the full plan + Feature Summary from design doc
- `{PROJECT_CONTEXT}` — read from `CLAUDE.md` `## Project Context` section. If missing, ask user to add it.
- **No `{MODULE_PATH}` or `{PROJECT_STRUCTURE}`** — personas don't read code

```
Agent(subagent_type="general-purpose", model="sonnet",
  team_name="rri-t-{uid}", name="end-user",
  prompt=[end-user.md + PLAN_REVIEW instructions + placeholders])
```
Repeat for ba, qa-destroyer, devops, security-auditor.

1 Triage Analyst (full codebase access):
```
Agent(subagent_type="general-purpose", model="sonnet",
  team_name="rri-t-{uid}", name="triage-analyst",
  prompt=[skills/rri-t/persona-prompts/triage-analyst.md with filled placeholders])
```

**4. Personas review the plan,** create FAIL finding tasks, mark their review task complete, go idle.

**5. Triage Analyst picks up FAIL findings** (unblocked after personas complete):
- Investigates the codebase (Glob, Grep, Read)
- For each finding: FIX NEEDED / NO FIX NEEDED / DOWNGRADE
- If NO FIX or DOWNGRADE: messages the **original persona** with evidence
- Persona responds — agrees or defends
- **Max 3 rounds per finding**

**6. Triage Analyst classifies and resolves:**
- **Technical fix** (wrong types, missing error handling, framework misuse) → auto-applies to plan. No user approval.
- **Requirement/scope decision** (feature choices, UX, business rules) → **NEVER assume. Escalate to user.**
- **Unresolved after 3 rounds** → escalate with both arguments.

**7. Triage Analyst writes investigation file** to `investigations/{date}-{topic}-plan-review-{uid}.md` and notifies lead.

**8. Lead reads investigation file,** presents ONLY items needing user decision:
- Requirement/scope questions
- Unresolved debates (both arguments shown)
- PAINFUL items as tradeoffs
- MISSING items as scope questions

**9. Lead updates investigation file** with user decisions. Shuts down team.

### Auto-Apply vs Ask User

| Finding type | Action | Example |
|-------------|--------|---------|
| Technical bug in plan | Auto-fix, no approval | Wrong transaction type → fix it |
| Missing error handling | Auto-fix, no approval | No version check → add it |
| Feature scope question | **ASK USER** | "Undo toast or immediate commit?" |
| Business rule ambiguity | **ASK USER** | "Should deleted items keep history?" |
| Unresolved debate | **ASK USER** with both arguments | Triage says X, persona says Y |

**When in doubt: ASK. Never assume the user's requirements.**

**After plan review resolves all items, run the Independent Coverage Agent (see below). Only proceed to Phase Gate after coverage agent returns PASS.**

## Independent Coverage Agent

After PLAN_REVIEW completes and user has decided on findings, spawn an independent coverage agent to verify structural integrity. This is the last gate before committing the plan.

**What to spawn:** An Explore subagent (lightweight, read-only) with this task:

> Read the Feature Summary table from [design doc path] and the Coverage section + task headers from [plan doc path]. Verify:
> 1. Every Feature Summary item has an entry in Coverage (Planned or Deferred)
> 2. Every "Planned" entry references task numbers that exist as ### Task N headers in the plan
> 3. Total counts (planned + deferred) match the Feature Summary total
> 4. Zero unaccounted items
>
> Return: PASS with summary, or FAIL with specific gaps.

**If PASS:** Proceed to commit plan and phase gate.

**If FAIL:** Plan the missing features (append tasks to existing plan), update Coverage section, re-run coverage agent. Max 2 loops before flagging to user.

**Why this exists:** The coverage agent is a DIFFERENT agent from the one that wrote the plan. This provides independent verification — the agent that might drop features is not the one checking for dropped features.

## Phase Gate — Execution Handoff

After the plan passes RRI-T review, commit the implementation plan. Then use the `AskUserQuestion` tool to present execution options:

Question: "Implementation plan is complete, reviewed, and committed. How would you like to start building?"

Options:
1. **"Subagent-Driven (this session)" (Recommended)** — Stay in this session. Dispatch fresh subagent per task, review between tasks. Suggest user `/clear` first if context is heavy.
2. **"Parallel Session (separate)"** — Open new session in worktree with executing-plans skill for batch execution.

**If user chooses "Subagent-Driven (this session)":**
- Ensure both `docs/plans/<design>.md` and `docs/plans/<plan>.md` are committed
- Summarize: "Plans committed at `docs/plans/`. Starting execution. (Tip: `/clear` first if you want fresh context.)"
- **REQUIRED SUB-SKILL:** Use super-bear:subagent-driven-development
- Fresh subagent per task + code review

**If user chooses "Parallel Session":**
- Guide them to open new session in worktree
- **REQUIRED SUB-SKILL:** New session uses super-bear:executing-plans
