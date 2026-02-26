---
name: rri-t
description: "Use for structured quality review with 5 persona subagents (End User, BA, QA Destroyer, DevOps, Security). Invoked by brainstorming (discovery), writing-plans (plan review), or directly for post-code verification. 4-level results: PASS/FAIL/PAINFUL/MISSING."
---

# RRI-T Quality Review

## Overview

RRI-T (Reverse Requirements Interview — Testing) replaces the generic design-cross-check with 5 specialized persona subagents that review work through their domain lens. Each persona reads only code relevant to their area, writes findings to persistent files, and returns a summary to the lead.

**Announce at start:** "I'm using the RRI-T skill to run a [phase] review with 5 persona subagents."

**This skill is invoked by other skills** — brainstorming (DISCOVER), writing-plans (PLAN_REVIEW), or directly for POST_CODE_VERIFY. It can also be invoked standalone.

## Prerequisites

- Module context — what module/feature is being reviewed
- Phase-specific artifact:
  - DISCOVER: project code + requirements
  - PLAN_REVIEW: implementation plan
  - POST_CODE_VERIFY: implemented code

## Persona Structure

| Persona | Focus | Reads |
|---------|-------|-------|
| **End User** | Workflow, UX, daily tasks, offline, mobile | UI components, page flows, forms, frontend |
| **BA** | Business rules, RBAC, audit, compliance, data | Schema, validation, business rules, calculations |
| **QA Destroyer** | Edge cases, error paths, concurrency, boundaries | Error handling, input validation, state management |
| **DevOps** | Deploy, scaling, monitoring, backup, migration | Dockerfile, CI/CD, deploy scripts, migrations |
| **Security Auditor** | Auth, injection, data exposure, rate limiting | Auth middleware, API routes, sanitization, RBAC |

Each persona is a **subagent** (Task tool) that runs, writes findings, returns a summary, and exits. Persistence lives in the findings files, not in agent processes.

## Resource Safety

Personas use `model: "sonnet"` — they read code and write structured findings, which does not require Opus-level reasoning. This reduces memory usage per agent by ~50%.

All 5 subagents run in parallel via `run_in_background: true`. Each writes to its own findings file (no shared state conflicts). If the system is resource-constrained, the lead can spawn in batches of 2-3 instead of all 5.

## Module Detection and Findings Directory

**First-time setup (no `.claude/rri-t/` exists):**
1. Scan project structure (file listing only — light scan)
2. Propose module map to user for confirmation
3. Create `_project.md` with approved map

**Working on a specific module:**
1. Detect module from conversation context (e.g., "build Order Management for CRM" → module=crm, sub-module=order)
2. If `.claude/rri-t/{module}/` doesn't exist → create structure, notify user
3. If it exists → read `lead.md` to resume from current state

**Directory structure created lazily:**
```
.claude/rri-t/
├── _project.md                    ← Top-level module index
├── {module}/
│   ├── lead.md                    ← Module overview + aggregated findings
│   ├── end-user.md                ← End User persona findings
│   ├── ba.md                      ← BA persona findings
│   ├── qa.md                      ← QA Destroyer persona findings
│   ├── devops.md                  ← DevOps persona findings
│   ├── security.md                ← Security Auditor persona findings
│   └── {sub-module}/              ← Nested, same pattern repeats
│       ├── lead.md
│       ├── end-user.md
│       └── ...
```

Use templates from this skill's directory:
- `findings-template.md` — for persona findings files
- `lead-template.md` — for lead.md summary files

## Phase: DISCOVER

**When:** During brainstorming, after user Q&A and before proposing approaches.

**Purpose:** 5 personas identify hidden requirements from their perspectives.

### Step-by-step:

**1. Detect module and create findings directory (if needed).**

**2. Spawn 5 persona subagents in parallel** (read each template from `skills/rri-t/persona-prompts/`, fill placeholders, spawn):
```
Task(subagent_type="general-purpose", model="sonnet", run_in_background=true,
  description="End User persona DISCOVER",
  prompt=[end-user.md with filled placeholders])
Task(subagent_type="general-purpose", model="sonnet", run_in_background=true,
  description="BA persona DISCOVER",
  prompt=[ba.md with filled placeholders])
Task(subagent_type="general-purpose", model="sonnet", run_in_background=true,
  description="QA Destroyer persona DISCOVER",
  prompt=[qa-destroyer.md with filled placeholders])
Task(subagent_type="general-purpose", model="sonnet", run_in_background=true,
  description="DevOps persona DISCOVER",
  prompt=[devops.md with filled placeholders])
Task(subagent_type="general-purpose", model="sonnet", run_in_background=true,
  description="Security Auditor persona DISCOVER",
  prompt=[security-auditor.md with filled placeholders])
```

Fill these placeholders in each template:
- `{MODULE_NAME}` — detected module name
- `{MODULE_PATH}` — path to module code
- `{FINDINGS_DIR}` — path to `.claude/rri-t/{module}/`
- `{PHASE}` — "DISCOVER"
- `{ARTIFACT_CONTENT}` — design sections / requirements / project context
- `{PROJECT_STRUCTURE}` — output of light project scan (file listing)

**3. Wait for all 5 subagents to complete.** Each writes findings to its file and returns a summary:
```
"[Persona] DISCOVER complete for {MODULE_NAME}. [N] findings: [X] MISSING, [Y] PAINFUL. Key concern: [one sentence or 'none']."
```

**4. Aggregate into lead.md:**
- Read each persona's findings file
- Write aggregated summary to `.claude/rri-t/{module}/lead.md`
- Create ONE consolidated checklist for the user

**5. Present to user:**
- Show the consolidated checklist grouped by severity (FAIL → PAINFUL → MISSING)
- User approves/rejects/modifies each item

**6. Update findings files** with user decisions (lead does this directly — no relay needed since subagents have exited).

## Phase: PLAN_REVIEW

**When:** During writing-plans, after the implementation plan is drafted. Replaces design-cross-check.

**Purpose:** 5 personas review the plan through their specialized lens.

### Step-by-step:

**1. Spawn 5 persona subagents in parallel** (same as DISCOVER, but with `{PHASE}` set to "PLAN_REVIEW" and `{ARTIFACT_CONTENT}` containing the plan). Include instruction to read their existing findings file first if it exists (from DISCOVER phase).

**2. Wait for all 5 subagents to complete.** Each reads previous findings, reviews the plan, updates their findings file, and returns a summary.

**3. Aggregate into lead.md** — update the Plan Review section.

**4. Present to user:**
- Show findings grouped by severity
- FAIL items require user decision
- PAINFUL items presented as tradeoffs
- MISSING items presented as scope questions
- User decides on each

**5. Update findings files** with user decisions.

## Phase: POST_CODE_VERIFY

**When:** After implementation, before release. Can be invoked directly.

**Purpose:** 5 personas verify the implementation against all previous findings.

### Step-by-step:

**1. Spawn 5 persona subagents in parallel** (same pattern, `{PHASE}` = "POST_CODE_VERIFY"). Include instruction to read their existing findings file first.

**2. Wait for all 5 subagents to complete.** Each reads previous findings, verifies implementation, updates their findings file, and returns dimension scores.

**3. Generate coverage matrix:**

| Dimension | Persona | Score |
|-----------|---------|-------|
| UI/UX | End User | 90% |
| API | BA | 85% |
| Performance | QA Destroyer + DevOps | 78% |
| Security | Security Auditor | 92% |
| Data Integrity | BA + QA Destroyer | 88% |
| Infrastructure | DevOps | 95% |
| Edge Cases | QA Destroyer | 72% |

**4. Calculate release gate:**
- GREEN: all dimensions >= 85%
- YELLOW: any dimension 70-84%
- RED: any dimension < 70%

**5. Present to user** with coverage matrix and release gate recommendation.

## Context Resilience

Persistence lives in the **findings files**, not in agent processes. Subagents are stateless — they read findings files at the start of each phase and write updates before exiting.

| Situation | What happens |
|-----------|-------------|
| Session normal | Subagents spawn, read files, do work, return results, exit |
| Auto-compact | No impact — subagents are short-lived |
| Clear context / new session | Spawn fresh subagents, they read existing findings files |
| Different project | Plugin carries skill prompts, user-profile.md provides context |

## Escalation Format

When presenting findings to the user, translate to product language:

```
**[Severity] findings from the quality review:**

### Must Fix (FAIL)
- [What users will experience if not fixed — plain language]

### Works But Painful (PAINFUL)
- [What users will experience — plain language tradeoff]

### Not Covered (MISSING)
- [What's missing — framed as scope question for user to decide]
```

**Translation rules:**
- NEVER mention: file paths, function names, class names, code syntax
- ALWAYS frame as: what users experience, product impact, tradeoff
- Present FAIL items as blocking, PAINFUL as tradeoffs, MISSING as scope decisions

## User Override

If the user says "skip the review" or "just proceed" at any point, skip the review and move on. The user is always in control.
