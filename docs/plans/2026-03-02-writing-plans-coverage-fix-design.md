# Writing-Plans Coverage Fix — Design Document

> **Date:** 2026-03-02
> **Problem:** The writing-plans skill silently drops entire features when design docs are long (1000+ lines). Out of 10 features, only 8-9 get planned. Missing features are not flagged or deferred — they just disappear. RRI-T PLAN_REVIEW also fails to catch the gaps.
> **User requirement:** Structural mechanism (not prompt tweaks) to guarantee full coverage AND explicitly track intentional deferrals.

---

## RRI-T Discovery Findings (All Approved)

5 personas reviewed the writing-plans skill. 53 raw findings consolidated into 11 unique issues. All approved by user for fixing.

### MISSING — Structural gaps

| # | Finding | Flagged by | Root cause |
|---|---------|-----------|------------|
| 1 | **No feature manifest step** — skill jumps from "read design" to "write tasks" with no numbered feature list extracted first. No ground-truth to verify against. | ALL 5 personas | No enumeration before planning |
| 2 | **No coverage verification after planning** — nothing checks "design features vs plan tasks" after plan is drafted | ALL 5 personas | No post-planning gate |
| 3 | **PLAN_REVIEW receives only the plan, not the design doc** — review personas can't catch dropped features because they don't have the original design to compare against | 4 of 5 | Review inherits information loss |
| 4 | **No deferral mechanism** — no way to record "Feature X intentionally deferred because Y." Intentional and accidental omissions look identical | 4 of 5 | No deferral construct |
| 5 | **No coverage completeness persona in PLAN_REVIEW** — none of the 5 persona focus areas include "does this plan cover all design features?" | 3 of 5 | Gap between all review lenses |
| 6 | **Phase gate proceeds without coverage confirmation** — user approves execution without seeing "10 features in design, 8 in plan" | 3 of 5 | No coverage data at decision point |
| 7 | **No size-aware routing for large docs** — skill treats 50-line and 2000-line designs identically | 2 of 5 | One-size-fits-all process |

### PAINFUL — Works but unreliable

| # | Finding | Flagged by |
|---|---------|-----------|
| 8 | **Single-pass plan generation** — large designs processed in one uninterrupted pass, context exhaustion truncates silently | DevOps |
| 9 | **Plan header missing design doc link** — no provenance showing which design, what version, which features covered/deferred | End User, BA, Security |
| 10 | **No recovery path** — after discovering dropped features, recovery is manual and ad-hoc, no "gap fill" mode | QA, DevOps |
| 11 | **Verbal clarifications lost on context clear** — recommended "Clear Context" path discards brainstorming discussions not captured in design doc | QA |

---

## Approach Chosen: Feature Manifest + Chunked Planning

User chose the most robust approach: create a Feature Manifest as structured ground-truth, then plan features in chunks for large docs.

**Why this approach over alternatives:**
- "Manifest-in-Plan only" — doesn't solve context exhaustion for large docs
- "Separate Manifest File" — adds file management overhead
- "Manifest + Chunked Planning" — solves both the tracking problem AND the context exhaustion problem

---

## Design Decisions Made

### Decision 1: Manifest created in brainstorming, consumed in writing-plans

**Agreed:** The Feature Manifest is created during brainstorming, NOT during writing-plans.

**Placement within brainstorming:** After design sections are approved (step 6), before writing the design doc (step 7). It's the "final seal" on the design.

**Reasoning:**
- Brainstorming has the richest context — all user discussions, clarifications, RRI-T findings
- Re-extracting features from a 1000+ line doc in writing-plans is exactly where features get dropped today
- The manifest captures everything: original requirements + RRI-T additions the user approved
- It becomes part of the committed design doc, survives context clears

**Updated brainstorming flow:**
```
1. Explore project context
2. Ask clarifying questions (one at a time)
3. RRI-T Discovery (5 personas)
4. Present findings → user approves/rejects
5. Propose 2-3 approaches → user picks
6. Present design sections → user approves each
6b. NEW: Extract Feature Manifest from approved design
    → Present numbered list to user
    → User confirms completeness (can add missing ones)
7. Write design doc (now includes Feature Manifest section) → commit
8. Phase gate
```

**Skills modified:** brainstorming (new step 6b), writing-plans (reads manifest instead of re-extracting)

### Decision 2: Feature Manifest format

The manifest lives as a section in the design doc:

```markdown
## Feature Manifest
> Extracted from approved design sections above

| # | Feature | Design Section | Status |
|---|---------|---------------|--------|
| F1 | User authentication | Section 3.1 | — |
| F2 | Order management | Section 3.2 | — |
| F3 | Report export | Section 3.3 | — |

**Total: N features**
```

The Status column starts empty in the design doc. Writing-plans fills it during planning:
- `Planned → Task 1-3` — feature is covered
- `Deferred — [reason]` — intentionally skipped

### Decision 3: Chunked planning for large docs (APPROVED)

**How it works:**
1. Writing-plans reads the Feature Manifest from the design doc
2. If design is small (under ~500 lines): plan all features in one pass — no change from today
3. If design is large (500+ lines): group features by related sections, plan one group at a time
4. After each group: mark those manifest items as "planned" with task numbers
5. After all groups: verify every manifest item is either planned or deferred

**Example for 10-feature design:**
```
Group 1: F1 (Auth), F2 (User mgmt), F3 (RBAC)     → Tasks 1-8
Group 2: F4 (Order), F5 (Pricing), F6 (Invoice)    → Tasks 9-18
Group 3: F7 (Reports), F8 (Export), F9 (Dashboard)  → Tasks 19-25
Group 4: F10 (Notifications)                         → Tasks 26-28

Coverage check: 10/10 features planned ✓
```

**Why this works:** Each group only needs the manifest + relevant design sections in context. Agent's attention stays focused on a manageable chunk.

**Fixes:** Finding #7 (no size-aware routing), #8 (single-pass generation)

---

## RRI-T Discovery: Docs System & Manifest Lifecycle

User raised two new concerns during brainstorming:
1. `.claude/rri-t/` is inside a hidden directory — access/permission problems
2. Feature manifests need a home for easy tracking and reference

5 personas reviewed the docs system design (folder structure + manifest lifecycle). 53 raw findings consolidated below.

### Clear Winners from Persona Analysis

**Folder Structure → Per-feature folders**
- End User: "Option A is the only combination that gives both Khang and AI agent a predictable, scalable path"
- Flat directories don't scale past ~10 features (End User, QA)
- Date-prefixed flat files obscure which file is current (End User, QA)
- Per-feature folders = one Glob returns everything: `docs/features/{feature}/**`

**Manifest Lifecycle → Living document (one per feature, git for history)**
- End User: "Living document is strongly preferable for both users. Git provides history."
- Per-session manifests create ambiguity about which is authoritative (End User, QA)
- AI agent needs predictable path: `docs/features/{feature}/manifest.md` — always one file
- Git history answers "what was the manifest at point X in time" if needed

### Key New Issues Raised (grouped by theme)

**Manifest Identity & Stability:**
- F-IDs (F1, F2) are sequential integers — renaming/reordering breaks all references (QA, BA, Security)
- No tamper detection after brainstorming commits manifest (Security)
- Manifest write-back mechanism unspecified — writing-plans should update Status but no instruction exists (QA, BA)

**Cross-Version Lifecycle:**
- No linkage between V1 and V2 manifests — deferred items have no guaranteed pickup (BA, QA, Security)
- Feature split/merge/rename/abandon scenarios undefined (BA)
- No "Supersedes" or "Previous Version" field (BA)

**Registry & Discoverability:**
- Need a central index file — who maintains it, what triggers updates (ALL 5 personas)
- `_project.md` has no schema and gets stale (DevOps, Security)
- Slug canonicalization undefined — AI could create duplicate folders (End User)

**Operational:**
- No migration path from current `.claude/rri-t/` to new location (DevOps)
- No archival policy for completed features (DevOps)
- Findings files accumulate without archiving — context cost grows per subagent (QA, DevOps)
- No schema versioning for templates (DevOps)
- YAML frontmatter has no enforcement — already missing from live docs (DevOps)

**Coverage Verification Quality:**
- Coverage check is structural (is Status filled?) not semantic (does task content match feature?) (QA)
- PLAN_REVIEW still receives only the plan, not design+manifest — same blind spot (Security, QA)

### Issues Deferred (YAGNI for now)

These are valid but add complexity beyond current scope:
- Schema versioning for templates — solve when format actually changes
- Git-based tamper detection — honor system is sufficient for a single-user PM workflow
- Machine-parseable manifest cross-references — prose references are adequate for AI agents with Grep
- Parallel multi-module review conflicts on shared lead.md — solve when concurrent module reviews actually happen

---

## Decision 4: Docs System — Two-Level Architecture

**Approved.** Artifacts split into two levels with clear separation of concerns.

### Two Levels: Project vs Task

| Level | Purpose | Lifecycle | Location |
|-------|---------|-----------|----------|
| **Project** | Track features, status, decisions | Living, persistent | `docs/features/` |
| **Task** | Investigate current requirement | Per-task, disposable | `investigations/` |

**Project-level artifacts** (persistent, curated documentation):
- `registry.md` — all features at a glance
- `manifest.md` per feature — living feature checklist
- `design.md` per feature — latest approved design
- `plan.md` per feature — latest implementation plan

**Task-level artifacts** (per-task evidence, always fresh):
- Investigation files — one consolidated file per RRI-T audit (all 5 personas in one file)
- Always create new file, never append to existing
- Referenced from manifest as evidence for decisions

### Folder Structure

```
docs/features/                              ← PROJECT LEVEL
├── registry.md                             ← all features index
├── order-management/
│   ├── manifest.md                         ← living, always current
│   ├── design.md                           ← latest design
│   └── plan.md                             ← latest plan
├── user-authentication/
│   └── ... same structure
└── ...

investigations/                             ← TASK LEVEL
├── 2026-03-15-crm-order-management-discover-a3f2.md
├── 2026-03-15-crm-order-management-plan-review-a3f2.md
├── 2026-03-17-crm-order-management-post-verify-a3f2.md
├── 2026-03-15-crm-order-management-discover-b7e1.md   ← different task same feature same day
├── 2026-03-15-crm-invoice-export-discover-c4d9.md     ← different feature
```

### Investigation File Naming Convention

```
{date}-{module}-{feature}-{phase}-{uid}.md
```

- `{date}` — YYYY-MM-DD
- `{module}` — top-level area (crm, hr, accounting)
- `{feature}` — feature slug (order-management, invoice-export)
- `{phase}` — `discover` | `plan-review` | `post-verify`
- `{uid}` — 4-character random hex, generated once per task, reused across phases of same task

**Rigid rules:**
- One RRI-T audit = one new file. Always create, never append.
- The `{uid}` ties phases of the same task together
- Cross-phase reading: `Glob investigations/*-{module}-{feature}-*-{uid}.md`

### Investigation File Format (One File, All 5 Personas)

```markdown
# Investigation: Add Invoice Export to Order Management
> Date: 2026-03-15 | Module: crm | Feature: order-management | Phase: discover | UID: a3f2

## End User Findings
- [MISSING] No bulk export for month-end...
- [PAINFUL] Export button not visible on mobile...

## BA Findings
- [MISSING] VND currency format in exported invoices...

## QA Destroyer Findings
- [MISSING] No handling for 10,000+ row exports...

## DevOps Findings
- [PAINFUL] PDF export requires server-side font embedding...

## Security Findings
- [MISSING] Exported files contain customer PII with no access control...

## Consolidated Decisions
- F9 deferred: external API not available yet (user approved 2026-03-15)
- F10 added: bulk export requirement from End User finding (user approved)
```

**Key change from original RRI-T design:** 5 subagents return findings as output → lead writes ONE consolidated file. Subagents no longer need file write access (solves the permission problem from earlier).

### Manifest Format (Revised from Decision 2)

**Manifest is a separate file** at `docs/features/{feature}/manifest.md`:

```markdown
# Order Management — Feature Manifest

| # | Feature | Design Section | Status | Evidence |
|---|---------|---------------|--------|----------|
| F1 | User authentication | Section 3.1 | Planned → Task 1-3 | — |
| F2 | Order management | Section 3.2 | Planned → Task 4-7 | — |
| F9 | Invoice Export | Section 3.9 | Deferred — API unavailable | [investigation](../investigations/2026-03-15-crm-order-management-discover-a3f2.md) |

**Total: 10 features | 8 planned | 2 deferred | 0 unaccounted**
```

- Separate from design doc (design = historical record, manifest = living tracker)
- Evidence column links to investigation files as proof for decisions
- Writing-plans updates Status column after each planning chunk

### Key Rules

**Slug canonicalization:**
- Always lowercase, hyphens for spaces, no special characters
- Derived from feature name at brainstorming time, confirmed by user
- Example: "Order Management V2" → `order-management` (version NOT in slug)

**Registry file (`docs/features/registry.md`):**
- Updated by AI agent on every manifest create/update
- Format:

```markdown
# Feature Registry

| Feature | Slug | Module | Status | Manifest | Created | Updated |
|---------|------|--------|--------|----------|---------|---------|
| Order Management | order-management | crm | In Progress (8/10) | [manifest](order-management/manifest.md) | 2026-03-01 | 2026-03-02 |
| User Auth | user-authentication | crm | Complete | [manifest](user-authentication/manifest.md) | 2026-02-15 | 2026-02-20 |
```

### Decision 5: Cross-Version Manifest Linkage

**How V2 picks up V1 deferrals:**
1. Brainstorming skill reads existing `manifest.md` if the feature folder already exists
2. Presents deferred items to user: "These were deferred last time — are they in scope now?"
3. User decides per item: pick up, keep deferred, or remove
4. Manifest is updated (living document) — git captures the V1→V2 transition
5. Design doc for V2 is a new file (`design-v2.md` or dated) in the same feature folder

**Feature split/merge/abandon:**
- Split: create new feature folder, update both manifests with cross-reference
- Merge: pick one folder, archive the other, update manifest
- Abandon: update manifest Status to "Abandoned — [reason]", update registry

### Decision 6: Operational Handling

**Write-back mechanism:** Writing-plans skill gets explicit instruction to update `manifest.md` Status column after each chunk.

**Migration:** One-time step — move `.claude/rri-t/` contents into `investigations/` with new naming convention. Update all skill paths. Remove old `.claude/rri-t/` references.

**YAML frontmatter:** Not enforcing — AI agent uses Grep on content and predictable paths. Deferred.

---

## Decision 7: Coverage Verification Step (fixes #2, #6)

**What:** Mandatory cross-check after writing all plan tasks, before phase gate.

**How:**
1. Read `manifest.md` — get all features and their Status
2. Count: planned, deferred, unaccounted
3. If any feature has Status = `—` (unaccounted): **STOP. Flag to user.**
4. Present coverage summary:
   ```
   Coverage: 8 planned, 2 deferred, 0 unaccounted
   F1-F8: Planned → Tasks 1-28
   F9: Deferred — depends on external API
   F10: Deferred — out of scope for MVP
   ```
5. User confirms before proceeding to RRI-T PLAN_REVIEW

**Rule:** The phase gate CANNOT be presented until coverage = 0 unaccounted.

## Decision 8: Deferral Mechanism (fixes #4)

**In manifest.md:** Status = `Deferred — [reason]`, with Evidence column linking to investigation file.

**In plan.md:** New section after all tasks:

```markdown
## Deferred Features

| Manifest # | Feature | Reason | Pickup Condition |
|-----------|---------|--------|-----------------|
| F9 | External API integration | API not available | When vendor delivers API docs |
| F10 | Advanced reporting | Out of MVP scope | V2 planning |
```

**Rule:** Every manifest item must be either Planned or Deferred. No blank statuses allowed after planning completes.

## Decision 9: PLAN_REVIEW Fix (fixes #3, #5)

**Change 1 — Pass manifest to reviewers:**
- `{ARTIFACT_CONTENT}` now includes the plan AND the manifest (not just the plan)
- Manifest is compact — adds minimal context overhead even for large plans

**Change 2 — Add coverage check to ALL personas:**
- Add to every persona's PLAN_REVIEW instructions: "Before your domain review, verify every manifest item appears in at least one plan task. Flag any manifest item with no corresponding task as [MISSING]."
- Every persona does coverage check first, then domain-specific review
- This means 5 independent coverage checks — if even one persona catches a gap, it surfaces

## Decision 10: Plan Header Update (fixes #9)

Updated plan header adds provenance and coverage:

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** Use super-bear:executing-plans to implement this plan.

**Goal:** [One sentence]
**Design:** [link to docs/features/{feature}/design.md]
**Manifest:** [link to docs/features/{feature}/manifest.md]
**Coverage:** 8/10 planned, 2 deferred, 0 unaccounted
**Architecture:** [2-3 sentences]
**Tech Stack:** [Key technologies]
```

## Decision 11: Recovery/Gap-Fill Mode (fixes #10)

**When:** User discovers a feature was dropped after planning (or after partial execution).

**How:** Writing-plans skill gets a "gap fill" invocation mode:
1. Read `manifest.md` — find features with Status = `—` or user flags a gap
2. Plan tasks ONLY for the missing features (don't rewrite existing plan)
3. Append new tasks to existing `plan.md`
4. Update manifest Status for newly planned features
5. Re-run coverage verification

**Trigger:** User says "gap fill" or agent detects unaccounted manifest items.

## Decision 12: Context Clear Safety (fixes #11)

**What:** Before clearing context, brainstorming must capture verbal clarifications.

**How:** Add a step before phase gate in brainstorming:
1. Before offering "Clear Context", agent reviews conversation for undocumented verbal specs
2. If found, appends to design doc under `## Verbal Clarifications` section
3. User confirms design doc captures everything discussed
4. Only THEN offer the phase gate

**Rule:** Context clear is blocked until user confirms "yes, the design doc captures everything."

---

## Summary of All Changes

### Skills Modified

| Skill | Changes |
|-------|---------|
| **brainstorming** | New step 6b (extract manifest), new step before phase gate (capture verbal specs), new step (read existing manifest for V2 deferrals) |
| **writing-plans** | Read manifest instead of re-extracting, chunked planning for large docs, write-back to manifest.md, coverage verification step, gap-fill mode, updated plan header, deferred features section |
| **rri-t** | Consolidated investigation file (1 file not 6), new investigation naming convention, pass manifest to PLAN_REVIEW, add coverage check to all personas, new file paths (investigations/ not .claude/rri-t/) |
| **rri-t persona prompts** | All 5 personas get coverage check instruction in PLAN_REVIEW |

### New Artifacts

| Artifact | Location | Purpose |
|----------|----------|---------|
| Feature Registry | `docs/features/registry.md` | Project-wide feature index |
| Feature Manifest | `docs/features/{feature}/manifest.md` | Per-feature living checklist |
| Investigation files | `investigations/{date}-{module}-{feature}-{phase}-{uid}.md` | Per-task RRI-T evidence |

### Migration Required

- Move `.claude/rri-t/` contents to `investigations/` with new naming
- Create `docs/features/` structure
- Update all skill file paths
- Remove old `.claude/rri-t/` references
