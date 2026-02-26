# RRI-T Integration into Super Bear — Design Document

## Goal

Replace the generic design-cross-check (Author vs Reviewer team debate) with a structured RRI-T quality methodology: 5 persistent persona agents that discover hidden requirements, review plans, and verify implementations across the full development lifecycle.

## Problem

1. Current design-cross-check is expensive and generic — one reviewer trying to catch everything
2. No structured framework for what to look for — findings tend to be generic/theoretical
3. No post-code verification methodology
4. Binary PASS/FAIL misses "works but painful" and "feature missing" categories
5. No persistent quality knowledge per module — every review starts from scratch

## Decisions Made

- **Subagents** over Agent Team — persistence lives in findings files, not agent processes. Agent Teams caused OOM crashes (5 Opus agents + child processes like rust-analyzer consumed 15GB+). Subagents spawn, do work, return results, and exit cleanly.
- **5 personas** from RRI-T: End User, BA, QA Destroyer, DevOps, Security Auditor
- **7 testing dimensions**: UI/UX, API, Performance, Security, Data Integrity, Infrastructure, Edge Cases
- **8 stress axes**: Time, Data, Error, Collab, Emergency, Security, Infra, Locale
- **4-level results**: PASS / FAIL / PAINFUL / MISSING (replaces binary)
- **Vietnamese-specific testing** baked into core (diacritics, VND, timezone, address format, etc.)
- **Findings organized per module** mirroring project structure, not per-project
- **Each persona reads only relevant code** — lead scans structure, personas read deep on their area
- **lead.md as index** — lead reads 1 summary file per level, drills into persona files only when needed
- **Update after each unit of work** — like git commits, findings must stay current
- **Delete design-cross-check** entirely — replaced by RRI-T review phase
- **User profile** already baked into plugin (user-profile.md + session-start hook)

## Architecture

### Subagent Structure

```
Lead (main session — user talks to lead only)
│
├── Spawns 5 subagents in parallel (Task tool, model="sonnet", run_in_background=true)
│
├── End User Subagent — workflow, UX, daily tasks, offline, mobile
├── BA Subagent — business rules, RBAC, audit, compliance, data consistency
├── QA Destroyer Subagent — edge cases, error paths, concurrent ops, boundary values
├── DevOps Subagent — deploy, scaling, monitoring, backup, migration
└── Security Auditor Subagent — auth, injection, data exposure, rate limiting
```

Subagents spawn per phase, read findings files for context, do work, write findings, return summary, and exit. No persistent processes. No orphan risk.

**Resource safety:**
- Model: Sonnet (not Opus) — sufficient for code reading + structured findings
- No Bash access — personas use Read/Glob/Grep/Write only, preventing accidental spawns of heavy processes (rust-analyzer, cargo, etc.)
- Short-lived — subagents exit after completing their review

### Findings Directory Structure

```
.claude/rri-t/
├── _project.md                          ← Top-level: lists all modules
│
├── crm/
│   ├── lead.md                          ← CRM overview + summary of sub-modules
│   ├── end-user.md                      ← CRM-level end-user findings
│   ├── ba.md
│   ├── qa.md
│   ├── devops.md
│   ├── security.md
│   │
│   ├── order/
│   │   ├── lead.md                      ← Order module context + summarized findings
│   │   ├── end-user.md                  ← Detailed findings — agent writes here
│   │   ├── ba.md
│   │   ├── qa.md
│   │   ├── devops.md
│   │   └── security.md
│   │
│   └── quotation/
│       └── ...same pattern
│
└── mrp/
    └── ...same pattern
```

**Pattern per module:**
```
any-module/
├── lead.md              ← Index: overview + summarized findings (lead reads this)
├── end-user.md          ← Detailed findings (agent writes concurrent)
├── ba.md
├── qa.md
├── devops.md
├── security.md
└── [sub-module]/        ← Nested, same structure repeats
```

**Reading rules:**
- Lead reads ONLY lead.md at each level — never all 5 persona files
- Lead drills into persona file only when detail needed
- Working on Order: read `crm/order/lead.md` (deep) + `crm/lead.md` (light) + `_project.md` (light)

**Creation rules — on-demand, not upfront:**
- Directory structure is created lazily — only when actually working on a module
- Lead detects module from conversation context (user says "build Order Management for CRM" → module=crm, sub-module=order)
- If `.claude/rri-t/crm/order/` doesn't exist → create structure + notify user
- If it exists → read lead.md to resume from current state
- First time using RRI-T in a project: lead scans codebase structure, proposes module map to user for confirmation, creates `_project.md` with approved map
- Detail directories (5 persona files) only created when starting actual work on that module — not pre-created for entire project
- User may correct or add modules the lead didn't detect (e.g., modules planned but not yet in code)

**Update rules:**
- After completing each unit of work: persona updates their file, lead updates lead.md
- Like git commits — not realtime, not end-of-project, but after each completed thing
- If a finding is resolved (e.g., FAIL item fixed), update status in findings file — don't delete, mark as resolved with date

### Lifecycle Flow

```
Brainstorming starts (lead + user Q&A)
│
├─ User gives requirements
│
├─ Lead scans project structure (light — file listing only)
├─ Lead creates/updates _project.md + module _overview
├─ PHASE: DISCOVER (RRI)
│   Lead spawns 5 subagents in parallel (sonnet, no Bash)
│   Each reads ONLY their relevant code area
│   → Each writes to their findings file, returns summary, exits
│   → Lead aggregates into lead.md
│   → Lead presents ONE checklist to user
│   → User approves/rejects/modifies each item
│   → Lead updates findings files with user decisions
│
├─ Brainstorming continues: propose approaches → present design
│
├─ PHASE: PLAN REVIEW (replaces design-cross-check)
│   Lead spawns 5 fresh subagents (they read previous findings files for context)
│   Each reviews plan through their lens + 7 dimensions
│   → Results tagged: PASS / FAIL / PAINFUL / MISSING
│   → Each updates their findings file, returns summary, exits
│   → Lead aggregates into lead.md + presents to user
│   → User decides on FAIL/PAINFUL/MISSING items
│
├─ Execution (code)
│
├─ PHASE: POST-CODE VERIFY (new)
│   Lead spawns 5 fresh subagents (they read previous findings files)
│   Each checks their relevant code area
│   → Results tagged with 4 levels per 7 dimensions
│   → Coverage matrix generated
│   → Release gate: green (>=85%) / yellow (70-84%) / red (<70%)
│   → Lead presents to user for release decision
│
└─ Final update to all findings files
```

### Vietnamese-Specific Testing (baked into core)

Every persona's prompt includes these Vietnamese-specific checks where relevant:

| # | Area | Test | Expected |
|---|---|---|---|
| 1 | Diacritics | Search "nguyen" finds "Nguyen" | Diacritic-insensitive search |
| 2 | Unicode sorting | Sort: An, Binh, Cuong, Duc | Vietnamese order (D after D) |
| 3 | VND Currency | Display 1234567 VND | 1.234.567 d (dot separator) |
| 4 | Phone format | +84 912 345 678 or 0912345678 | Accept both, normalize |
| 5 | Date format | Display date | DD/MM/YYYY |
| 6 | Timezone | Server UTC, display local | GMT+7 consistently |
| 7 | Address | Vietnamese address structure | So nha/Duong/Phuong/Quan/TP |
| 8 | CCCD/CMND | 12 digits (CCCD), 9 digits (CMND) | Accept both |
| 9 | Tax ID (MST) | 10 or 13 digits | Validate format |
| 10 | Text overflow | VN text ~30% longer than EN | UI must not break |
| 11 | Font rendering | Diacritics at small font 10px | Clear, not clipped |
| 12 | Input methods | Telex, VNI, VIQR | All smooth |
| 13 | PDF export | Export with Vietnamese content | Diacritics correct in PDF |

These are not a separate checklist — they're woven into each persona's prompt as part of their domain.

### Stress Combination Testing

During review and verification phases, personas test intersections:

| Combination | Scenario | Example |
|---|---|---|
| TIME x DATA | Deadline + large data | Bulk approve 500 records in 3 minutes |
| COLLAB x ERROR | Multi-user + conflict | 3 users edit, 1 save fails, others? |
| COLLAB x SECURITY | Multi-user + permissions | Admin and Viewer open same record |
| INFRA x DATA | Server stress + large data | Migration 1M records + concurrent users |
| SECURITY x LOCALE | Auth + Vietnamese | Username "Nguyen Van A" in JWT |
| INFRA x EMERGENCY | Server incident + deadline | Prod down during budget close |

### Context Resilience

| Situation | What happens |
|---|---|
| Session normal | Subagents spawn, read findings files, do work, return results, exit |
| Auto-compact | No impact — subagents are short-lived, findings persist in files |
| Clear context / new session | Spawn fresh subagents, they read existing findings files |
| Different VPS / project | Plugin carries skill prompts, user-profile.md provides context |

## Changes to Existing Files

### DELETE
- `skills/design-cross-check/` — entire directory (SKILL.md, escalation-guide.md, author-teammate-prompt.md, reviewer-teammate-prompt.md)
- `agents/design-reviewer.md` — replaced by 5 persona agents

### MODIFY
- `skills/brainstorming/SKILL.md` — add RRI discovery phase after user Q&A, replace design-reviewer sanity check with 5-persona review
- `skills/writing-plans/SKILL.md` — replace design-cross-check invocation with RRI-T plan review phase
- `skills/using-superpowers/SKILL.md` — update skill list (remove design-cross-check, add rri-t)

### CREATE
- `skills/rri-t/SKILL.md` — core RRI-T skill with team setup, phase management, findings format
- `skills/rri-t/persona-prompts/end-user.md` — spawn template for End User agent
- `skills/rri-t/persona-prompts/ba.md` — spawn template for BA agent
- `skills/rri-t/persona-prompts/qa-destroyer.md` — spawn template for QA Destroyer agent
- `skills/rri-t/persona-prompts/devops.md` — spawn template for DevOps agent
- `skills/rri-t/persona-prompts/security-auditor.md` — spawn template for Security Auditor agent
- `skills/rri-t/findings-template.md` — template for persona findings files
- `skills/rri-t/lead-template.md` — template for lead.md summary files
- `agents/code-reviewer.md` — update to use 4-level results instead of binary

### Findings File Format (template)

```markdown
# [Module Name] — [Persona] Findings

## Module Context
- Parent: [parent module]
- Related: [linked modules]
- Last updated: [date]

## Discovery
- [MISSING] [finding with code reference]
- [PAINFUL] [finding with code reference]
...

## Plan Review
- [PASS] [what was covered]
- [FAIL] [what's wrong — file:line reference]
- [PAINFUL] [works but bad UX — specific scenario]
- [MISSING] [not in plan but should be]
...

## Post-code Verification
- [PASS] [verified working — test reference]
- [FAIL] [broken — file:line reference]
- [PAINFUL] [works but UX issue — specific scenario]
- [MISSING] [still not implemented]
...
```

### lead.md Format (template)

```markdown
# [Module Name] — RRI-T Summary

## Module Context
- Parent: [parent module]
- Sub-modules: [list]
- Related: [linked modules]
- Schema: [key tables]
- Last updated: [date]

## Status
- Discovery: [done/in-progress] — [date]
- Plan Review: [done/in-progress] — [date]
- Verification: [done/in-progress] — [date]

## Key Findings (aggregated)
### Critical (FAIL)
- [summary — see qa.md for detail]

### Painful
- [summary — see end-user.md for detail]

### Missing
- [summary — see ba.md for detail]

## Coverage Matrix
| Dimension | Status | Score |
|-----------|--------|-------|
| UI/UX | green | 90% |
| API | yellow | 78% |
| ... | ... | ... |

## Release Gate: [GREEN/YELLOW/RED]
```
