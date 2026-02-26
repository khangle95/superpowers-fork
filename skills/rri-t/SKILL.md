---
name: rri-t
description: "Use for structured quality review with 5 persona agents (End User, BA, QA Destroyer, DevOps, Security). Invoked by brainstorming (discovery), writing-plans (plan review), or directly for post-code verification. 4-level results: PASS/FAIL/PAINFUL/MISSING."
---

# RRI-T Quality Review

## Overview

RRI-T (Reverse Requirements Interview — Testing) replaces the generic design-cross-check with 5 specialized persona agents that review work through their domain lens. Each persona reads only code relevant to their area, writes findings to persistent files, and reports a summary to the lead.

**Announce at start:** "I'm using the RRI-T skill to run a [phase] review with 5 persona agents."

**This skill is invoked by other skills** — brainstorming (DISCOVER), writing-plans (PLAN_REVIEW), or directly for POST_CODE_VERIFY. It can also be invoked standalone.

## Prerequisites

- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` must be enabled
- Module context — what module/feature is being reviewed
- Phase-specific artifact:
  - DISCOVER: project code + requirements
  - PLAN_REVIEW: implementation plan
  - POST_CODE_VERIFY: implemented code

## Team Structure

| Persona | Focus | Reads |
|---------|-------|-------|
| **End User** | Workflow, UX, daily tasks, offline, mobile | UI components, page flows, forms, frontend |
| **BA** | Business rules, RBAC, audit, compliance, data | Schema, validation, business rules, calculations |
| **QA Destroyer** | Edge cases, error paths, concurrency, boundaries | Error handling, input validation, state management |
| **DevOps** | Deploy, scaling, monitoring, backup, migration | Dockerfile, CI/CD, deploy scripts, migrations |
| **Security Auditor** | Auth, injection, data exposure, rate limiting | Auth middleware, API routes, sanitization, RBAC |

Team is created ONCE per session. If context was cleared and findings files exist, spawn a new team and each persona reads their previous findings to resume.

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

**2. Create the team (if not already created this session):**
```
TeamCreate("rri-t")
```

**3. Spawn 5 personas** (read each template from `skills/rri-t/persona-prompts/`, fill placeholders, spawn):
```
Task(team_name="rri-t", name="end-user", prompt=[end-user.md with filled placeholders])
Task(team_name="rri-t", name="ba", prompt=[ba.md with filled placeholders])
Task(team_name="rri-t", name="qa-destroyer", prompt=[qa-destroyer.md with filled placeholders])
Task(team_name="rri-t", name="devops", prompt=[devops.md with filled placeholders])
Task(team_name="rri-t", name="security", prompt=[security-auditor.md with filled placeholders])
```

Fill these placeholders in each template:
- `{MODULE_NAME}` — detected module name
- `{MODULE_PATH}` — path to module code
- `{FINDINGS_DIR}` — path to `.claude/rri-t/{module}/`
- `{PHASE}` — "DISCOVER"
- `{ARTIFACT_CONTENT}` — design sections / requirements / project context
- `{PROJECT_STRUCTURE}` — output of light project scan (file listing)

**4. Kick off discovery:**
```
SendMessage(type="broadcast",
  content="Phase: DISCOVER for {MODULE_NAME}. Read your relevant code areas, identify hidden requirements from your perspective. Write findings to your file, then message me with your summary.")
```

**5. Wait for 5 summary messages.** Each persona writes findings and reports:
```
"[Persona] DISCOVER complete for {MODULE_NAME}. [N] findings: [X] MISSING, [Y] PAINFUL. Key concern: [one sentence or 'none']."
```

**6. Aggregate into lead.md:**
- Read each persona's findings file
- Write aggregated summary to `.claude/rri-t/{module}/lead.md`
- Create ONE consolidated checklist for the user

**7. Present to user:**
- Show the consolidated checklist grouped by severity (FAIL → PAINFUL → MISSING)
- User approves/rejects/modifies each item
- Relay user decisions to personas via SendMessage
- Personas update their findings files with user decisions

**8. Keep team alive** for subsequent phases.

## Phase: PLAN_REVIEW

**When:** During writing-plans, after the implementation plan is drafted. Replaces design-cross-check.

**Purpose:** 5 personas review the plan through their specialized lens.

### Step-by-step:

**1. If team exists from DISCOVER phase, reuse it. If not (context was cleared), create a new team and have personas read their existing findings files.**

**2. Send plan review instructions:**
```
SendMessage(type="broadcast",
  content="Phase: PLAN_REVIEW for {MODULE_NAME}. Review this implementation plan through your lens. For each finding, tag as PASS/FAIL/PAINFUL/MISSING. Write findings to your file, then message me with your summary.\n\nPlan:\n{PLAN_CONTENT}")
```

**3. Wait for 5 summary messages.**

**4. Aggregate into lead.md** — update the Plan Review section.

**5. Present to user:**
- Show findings grouped by severity
- FAIL items require user decision
- PAINFUL items presented as tradeoffs
- MISSING items presented as scope questions
- User decides on each

**6. Relay decisions to personas. Personas update files.**

**7. Keep team alive** (or shut down if transitioning to execution in a new session).

## Phase: POST_CODE_VERIFY

**When:** After implementation, before release. Can be invoked directly.

**Purpose:** 5 personas verify the implementation against all previous findings.

### Step-by-step:

**1. Create team (or reuse). Personas read their existing findings files.**

**2. Send verification instructions:**
```
SendMessage(type="broadcast",
  content="Phase: POST_CODE_VERIFY for {MODULE_NAME}. Read the implementation. Verify against your previous findings. Update statuses. Score your dimensions (0-100%). Message me with your summary.")
```

**3. Wait for 5 summary messages.** Each includes dimension scores.

**4. Generate coverage matrix:**

| Dimension | Persona | Score |
|-----------|---------|-------|
| UI/UX | End User | 90% |
| API | BA | 85% |
| Performance | QA Destroyer + DevOps | 78% |
| Security | Security Auditor | 92% |
| Data Integrity | BA + QA Destroyer | 88% |
| Infrastructure | DevOps | 95% |
| Edge Cases | QA Destroyer | 72% |

**5. Calculate release gate:**
- GREEN: all dimensions >= 85%
- YELLOW: any dimension 70-84%
- RED: any dimension < 70%

**6. Present to user** with coverage matrix and release gate recommendation.

## Shutting Down the Team

When all phases are complete or the user says "skip":
```
SendMessage(type="shutdown_request", recipient="end-user")
SendMessage(type="shutdown_request", recipient="ba")
SendMessage(type="shutdown_request", recipient="qa-destroyer")
SendMessage(type="shutdown_request", recipient="devops")
SendMessage(type="shutdown_request", recipient="security")
TeamDelete
```

Final update to all findings files and lead.md.

## Context Resilience

| Situation | What happens |
|-----------|-------------|
| Session normal | Team uses in-memory context — fast |
| Auto-compact | Personas re-read their findings file — recover |
| Clear context / new session | Spawn new team, each persona reads their findings file |
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

If the user says "skip the review" or "just proceed" at any point, shut down the team and move on. The user is always in control.
