---
name: rri-t
description: "Use for structured quality review with 5 persona subagents (End User, BA, QA Destroyer, DevOps, Security). Invoked by brainstorming (discovery), writing-plans (plan review), or directly for post-code verification. 4-level results: PASS/FAIL/PAINFUL/MISSING."
---

# RRI-T Quality Review

## Overview

RRI-T (Reverse Requirements Interview — Testing) replaces the generic design-cross-check with 5 specialized persona subagents that review work through their domain lens. Each persona reads only code relevant to their area and returns findings to the lead, who writes a consolidated investigation file.

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

Each persona is a **subagent** (Task tool) that runs, returns findings as output, and exits. Persistence lives in the investigation files written by the lead.

## Resource Safety

Personas use `model: "sonnet"` — they read code and write structured findings, which does not require Opus-level reasoning. This reduces memory usage per agent by ~50%.

All 5 subagents run in parallel via `run_in_background: true`. Each returns findings as output (no file writes, no shared state conflicts). If the system is resource-constrained, the lead can spawn in batches of 2-3 instead of all 5.

## Investigation Files

RRI-T investigations are written to `investigations/` at the project root as consolidated files (all 5 personas in one file).

**File naming convention:**
```
investigations/{date}-{topic}-{phase}-{uid}.md
```

- `{date}` — YYYY-MM-DD
- `{topic}` — descriptive slug (e.g., crm-order-management, invoice-export)
- `{phase}` — `discover` | `plan-review` | `post-verify`
- `{uid}` — 4-character random hex, generated once per task, reused across phases

**Rules:**
- One RRI-T audit = one new file. Always create, never append.
- 5 subagents run in parallel and **return findings as output** to the lead
- Lead writes ONE consolidated file (not 6 separate files)
- Subagents do NOT need file write access
- The `{uid}` ties phases of the same task together
- Cross-phase reading: `Glob investigations/*-{topic}-*-{uid}.md`

**Consolidated file format:**
```markdown
# Investigation: [Title]
> Date: YYYY-MM-DD | Topic: [topic] | Phase: [phase] | UID: [uid]

## End User Findings
- [MISSING/PAINFUL] Description...

## BA Findings
- [MISSING/PAINFUL] Description...

## QA Destroyer Findings
- [MISSING/PAINFUL] Description...

## DevOps Findings
- [MISSING/PAINFUL] Description...

## Security Findings
- [MISSING/PAINFUL] Description...

## Consolidated Decisions
- [Decisions made by user after reviewing findings]
```

Use the consolidated investigation file format above. The old per-persona templates (`findings-template.md`, `lead-template.md`) are no longer used.

## Phase: DISCOVER

**When:** During brainstorming, after user Q&A and before proposing approaches.

**Purpose:** 5 personas identify hidden requirements from their perspectives.

### Step-by-step:

**1. Generate investigation UID** (4-char random hex, e.g., `a3f1`). This UID ties all phases of this task together.

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
- `{FINDINGS_DIR}` — NOT USED (subagents return output instead of writing files)
- `{PHASE}` — "DISCOVER"
- `{ARTIFACT_CONTENT}` — design sections / requirements / project context
- `{PROJECT_STRUCTURE}` — output of light project scan (file listing)

**Note:** Subagents return their findings as output text. They do NOT write to any files.

**3. Wait for all 5 subagents to complete.** Each returns findings as output text:
```
"[Persona] DISCOVER complete for {MODULE_NAME}. [N] findings: [X] MISSING, [Y] PAINFUL. Key concern: [one sentence or 'none']."
```

**4. Write consolidated investigation file:**
- Read each persona's returned output
- Write ONE consolidated investigation file to `investigations/{date}-{topic}-discover-{uid}.md`
- Create consolidated checklist for the user

**5. Present to user:**
- Show the consolidated checklist grouped by severity (FAIL → PAINFUL → MISSING)
- User approves/rejects/modifies each item

**6. Update the consolidated investigation file's** ## Consolidated Decisions section with user decisions.

## Phase: PLAN_REVIEW

**When:** During writing-plans, after the implementation plan is drafted. Replaces design-cross-check.

**Purpose:** 5 personas review the plan through their specialized lens.

### Step-by-step:

**1. Spawn 5 persona subagents in parallel** (same as DISCOVER, but with `{PHASE}` set to "PLAN_REVIEW"). Include instruction to read the previous DISCOVER investigation file if it exists: `Glob investigations/*-{topic}-discover-{uid}.md`. The `{ARTIFACT_CONTENT}` placeholder now includes BOTH the plan AND the Feature Summary table from the design doc. Personas use the Feature Summary as context for better domain review — they do NOT do coverage counting.

**2. Wait for all 5 subagents to complete.** Each returns findings as output text.

**3. Write ONE consolidated investigation file** to `investigations/{date}-{topic}-plan-review-{uid}.md`.

**4. Present to user:**
- Show findings grouped by severity
- FAIL items require user decision
- PAINFUL items presented as tradeoffs
- MISSING items presented as scope questions
- User decides on each

**5. Update the consolidated investigation file's** ## Consolidated Decisions section with user decisions.

## Phase: POST_CODE_VERIFY

**When:** After implementation, before release. Can be invoked directly.

**Purpose:** 5 personas verify the implementation against all previous findings.

### Step-by-step:

**1. Spawn 5 persona subagents in parallel** (same pattern, `{PHASE}` = "POST_CODE_VERIFY"). Include instruction to read previous investigation files for context: `Glob investigations/*-{topic}-*-{uid}.md`.

**2. Wait for all 5 subagents to complete.** Each returns findings and dimension scores as output text.

**2b. Write ONE consolidated investigation file** to `investigations/{date}-{topic}-post-verify-{uid}.md`.

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

Persistence lives in the **investigation files** in `investigations/`, not in agent processes. Subagents are stateless — they return findings as output, and the lead writes consolidated investigation files.

| Situation | What happens |
|-----------|-------------|
| Session normal | Subagents spawn, read code, return results, lead writes investigation file |
| Auto-compact | No impact — subagents are short-lived |
| Clear context / new session | Spawn fresh subagents, they read existing investigation files via Glob |
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
