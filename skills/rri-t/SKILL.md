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
- `{PROJECT_CONTEXT}` — **REQUIRED for all phases.** Read from the project's `CLAUDE.md` file, under `## Project Context`. This section is written once per project and contains:
  ```
  ## Project Context
  - User count: ~20 internal users
  - Deployment: single VPS with PM2, internal network
  - Public vs internal: internal ERP behind auth
  - Auth: session-based, role-based with 5 roles
  - Stack: Next.js 14, Drizzle ORM, PostgreSQL
  - Threat model: trusted employees, no public API
  ```
  The orchestrating skill (brainstorming or writing-plans) reads this section and passes it to agents via the `{PROJECT_CONTEXT}` placeholder. If the section doesn't exist in CLAUDE.md, ask the user to add it before proceeding.

## Phase Structures

Each phase uses a different agent structure:

| Phase | Structure | Personas get codebase? | Triage Analyst? |
|-------|-----------|----------------------|-----------------|
| DISCOVER | Fire-and-forget subagents | Yes | No |
| PLAN_REVIEW | Agent team with debate | No (plan only) | Yes |
| POST_CODE_VERIFY | Fire-and-forget subagents | Yes | No |

## Persona Structure

| Persona | Focus | Reads |
|---------|-------|-------|
| **End User** | Workflow, UX, daily tasks, offline, mobile | UI components, page flows, forms, frontend |
| **BA** | Business rules, RBAC, audit, compliance, data | Schema, validation, business rules, calculations |
| **QA Destroyer** | Edge cases, error paths, concurrency, boundaries | Error handling, input validation, state management |
| **DevOps** | Deploy, scaling, monitoring, backup, migration | Dockerfile, CI/CD, deploy scripts, migrations |
| **Security Auditor** | Auth, injection, data exposure, rate limiting | Auth middleware, API routes, sanitization, RBAC |

Each persona is spawned differently depending on phase (see Phase Structures above). In DISCOVER/POST_CODE_VERIFY they are fire-and-forget subagents. In PLAN_REVIEW they are agent team members that persist for debate with the Triage Analyst.

## Resource Safety

All agents use `model: "sonnet"` — they read code/plans and write structured findings, which does not require Opus-level reasoning.

DISCOVER/POST_CODE_VERIFY: 5 subagents run in parallel via `run_in_background: true`.
PLAN_REVIEW: 6 team members (5 personas + Triage Analyst). Personas go idle after findings; Triage Analyst does the heavy work.

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

## Phase-Specific Orchestration

**DISCOVER** and **PLAN_REVIEW** orchestration lives in the calling skills:
- DISCOVER → see `skills/brainstorming/SKILL.md` (## RRI-T Discovery Phase)
- PLAN_REVIEW → see `skills/writing-plans/SKILL.md` (## RRI-T Plan Review)

POST_CODE_VERIFY is invoked standalone, so its orchestration stays here.

## Phase: POST_CODE_VERIFY

**When:** After implementation, before release. Can be invoked directly.

**Purpose:** 5 personas verify the implementation against all previous findings.

### Step-by-step:

**1. Spawn 5 persona subagents in parallel** — fire-and-forget, WITH codebase access (same as DISCOVER). Set `{PHASE}` = "POST_CODE_VERIFY". Append instruction to read previous investigation files: `Glob investigations/*-{topic}-*-{uid}.md`.

**2. Wait for all 5 subagents to complete.** Each returns findings and dimension scores as output text.

**3. Write ONE consolidated investigation file** to `investigations/{date}-{topic}-post-verify-{uid}.md`.

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
