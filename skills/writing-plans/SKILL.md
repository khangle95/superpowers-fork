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

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

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

## RRI-T Plan Review

After drafting the plan, invoke `super-bear:rri-t` with phase=PLAN_REVIEW. This is the critical quality gate — the last checkpoint before code gets written.

The RRI-T skill spawns 5 persona subagents in parallel (End User, BA, QA Destroyer, DevOps, Security Auditor). Each reads their previous findings files (from DISCOVER phase if available), reviews the plan through their specialized lens, and tags findings as PASS / FAIL / PAINFUL / MISSING.

The lead aggregates findings and presents to the user:
- **FAIL** items require a user decision before proceeding
- **PAINFUL** items presented as tradeoffs
- **MISSING** items presented as scope questions

**Only proceed to Phase Gate after plan review completes and user has decided on all FAIL items.**

## Phase Gate — Execution Handoff

After the plan passes RRI-T review, commit the implementation plan. Then use the `AskUserQuestion` tool to present transition options:

Question: "Implementation plan is complete, reviewed, and committed. How would you like to start building?"

Options:
1. **"Clear Context, Commit All Plans, and Start Execution" (Recommended)** — Commits both design doc and implementation plan (if not already committed), clears conversation context, then starts execution fresh. Uses subagent-driven-development with the committed plan as input.
2. **"Subagent-Driven (this session)"** — Stay in this session. Dispatch fresh subagent per task, review between tasks.
3. **"Parallel Session (separate)"** — Open new session in worktree with executing-plans skill for batch execution.

**If user chooses "Clear Context and Start Execution":**
- Ensure both `docs/plans/<design>.md` and `docs/plans/<plan>.md` are committed
- Summarize: "Starting fresh execution session. Plans committed at `docs/plans/`."
- Use /clear or equivalent to reset context
- Then invoke subagent-driven-development skill, reading the committed plan as the task list

**If user chooses "Subagent-Driven (this session)":**
- **REQUIRED SUB-SKILL:** Use super-bear:subagent-driven-development
- Stay in this session
- Fresh subagent per task + code review

**If user chooses "Parallel Session":**
- Guide them to open new session in worktree
- **REQUIRED SUB-SKILL:** New session uses super-bear:executing-plans
