# Simplified Coverage Tracking — Design Document

> **Date:** 2026-03-04
> **Problem:** The proposed per-feature doc structure (manifest.md, registry.md, per-feature folders) from 2026-03-02 design solves the feature-drop problem but introduces 13 new issues: no iteration support, context bloat from extra files, AI self-verification, and operational fragility.
> **Solution:** Replace the file-heavy approach with inline Feature Summary + independent coverage agent. Zero new files, minimal context cost.
> **Supersedes:** The per-feature folder structure, manifest.md, and registry.md from `2026-03-02-writing-plans-coverage-fix-design.md`. The RRI-T consolidated investigation files fix is kept.
> **Discovery findings:** See `2026-03-03-doc-structure-rri-t-discover.md` for the 13 issues that led to this redesign.

---

## Part 1: Why The Original Proposal Was Wrong

The original proposal (2026-03-02) tried to solve "AI drops features during planning" by adding structural tracking files: manifest.md, registry.md, per-feature folders (design.md, plan.md per feature), evidence links, Status write-backs.

**The paradox:** We were asking an AI that can't reliably track 10 features in one document to reliably maintain 4-5 files per feature across multiple sessions.

**The 4 root problems discovered by RRI-T:**
1. **No iteration support** — structure works for Day 1 creation, breaks on Day 30+ edits/additions
2. **Wrong 1:1 assumption** — one brainstorming session can design many features; per-feature folders can't hold this
3. **AI verifies its own work** — same agent creates, updates, AND checks the manifest
4. **Operational fragility** — 10-14 workflow commits per feature, registry maintenance burden, slug identity problems

**Context is the bottleneck.** The more files the AI must read to understand state, the less context remains for actual work. The solution must REDUCE context overhead, not increase it.

---

## Part 2: The New Approach — Two Simple Additions

### Addition 1: Feature Summary (section in the design doc, NOT a separate file)

After brainstorming approves all design sections, extract a compact numbered table and add it to the end of the design doc:

```markdown
## Feature Summary

| # | Feature | Section |
|---|---------|---------|
| F1 | User Authentication | 3.1 |
| F2 | Order Management | 3.2 |
| F3 | Report Export | 3.3 |
| F4 | Invoice System | 3.4 |
| F5 | Dashboard | 3.5 |

**Total: 5 features**
```

**Rules:**
- Created during brainstorming step 6b (after design sections approved, before commit)
- User confirms completeness: "I found N features. Is this complete?"
- **Immutable after commit** — this is a snapshot of what was agreed, not a living document
- Compact: just F#, name, section reference. 10-20 lines for even large designs
- Multi-feature designs are fine: one table with 10 rows, no need to split

### Addition 2: Coverage Section (section in the plan doc, NOT a separate file)

After writing-plans produces all tasks, add a coverage table at the end of the plan doc:

```markdown
## Coverage
(Source: 2026-03-03-crm-system-design.md, Feature Summary)

| F# | Feature | Status | Tasks |
|----|---------|--------|-------|
| F1 | User Auth | Planned | Task 1-3 |
| F2 | Orders | Planned | Task 4-7 |
| F3 | Report Export | Planned | Task 8-10 |
| F4 | Invoice System | Deferred — external API unavailable | — |
| F5 | Dashboard | Deferred — out of MVP scope | — |

**3 planned, 2 deferred, 0 unaccounted**
```

### Addition 3: Independent Coverage Agent (quality gate)

A lightweight agent (separate from the one that wrote the plan) that checks structural integrity.

**When it runs:** After PLAN_REVIEW, before user approves execution.

**What it receives (~30-50 lines total):**
- The Feature Summary table from the design doc
- The task headers from the plan doc (just `### Task N: [Name]` lines)
- The Coverage section from the plan doc

**What it checks:**
1. Every Feature Summary item has an entry in Coverage (Planned or Deferred)
2. Every "Planned" entry references task numbers that exist in the task headers
3. Total counts are correct
4. Zero unaccounted items

**Output:** PASS or FAIL with specific gaps identified.

**If FAIL:** Loop back to writing-plans to plan missing features, then re-check. Max 2 loops before flagging to user.

**Why this solves the trust problem:** A different agent verifies. The agent that might drop features is not the one checking for dropped features.

---

## Part 3: What's Kept From The Original Proposal

### Consolidated Investigation Files (unchanged)

RRI-T investigations move from `.claude/rri-t/` (hidden, permission issues, 6 files per audit) to `investigations/` (visible, 1 file per audit).

```
investigations/
├── 2026-03-15-crm-order-management-discover-a3f2.md    ← all 5 personas in 1 file
├── 2026-03-15-crm-order-management-plan-review-a3f2.md
└── 2026-03-17-crm-order-management-post-verify-a3f2.md
```

**Rules:**
- 5 subagents return findings as output → lead writes ONE consolidated file
- Subagents no longer need file write access
- Always create new file, never append
- `{uid}` links phases of the same task together

### Chunked Planning for Large Designs (unchanged)

For large designs (500+ lines), writing-plans loads one design section at a time while keeping the Feature Summary table in context throughout. This prevents context exhaustion.

### Context Clear Safety (unchanged)

Before clearing context, agent captures verbal clarifications in the design doc. User confirms design doc is complete before context clear is offered.

---

## Part 4: What's Dropped From The Original Proposal

| Dropped | Why |
|---------|-----|
| `docs/features/` per-feature folders | No iteration support, 1:1 assumption wrong, context overhead |
| `manifest.md` (separate file) | Replaced by Feature Summary section in design doc |
| `registry.md` | Unnecessary — `Glob docs/plans/*.md` finds everything |
| `design.md` per feature | Design docs stay in `docs/plans/` with date prefix (flat, no versioning needed) |
| `plan.md` per feature | Plans stay in `docs/plans/` with date prefix |
| Slug canonicalization rules | No folders to name |
| Evidence links in manifest | No manifest. Investigation files stand alone. |
| Manifest Status write-backs | No manifest to write back to. Coverage section is in the plan. |
| Gap-fill mode (as separate concept) | Replaced by coverage agent loop-back |

---

## Part 5: Updated Pipeline

```
BRAINSTORMING
├── 1a. Explore project context (see context efficiency rule below)
├── 1b. Triage (if existing docs found — see Section 6)
├── 2. Ask questions (one at a time)
├── 3. RRI-T DISCOVER → writes investigations/{date}-{topic}-discover-{uid}.md
├── 4. Present findings → user approves/rejects
├── 5. Propose approaches → user picks
├── 6. Present design → user approves each section
├── 6b. NEW: Extract Feature Summary → user confirms completeness
├── 6c. Capture verbal clarifications if clearing context
├── 7. Write design doc (includes Feature Summary section) → commit
└── 8. Phase gate

WRITING PLANS
├── 1. Read design doc's Feature Summary FIRST (10-20 lines)
├── 2. If large doc → chunk by design sections (Feature Summary stays in context)
├── 3. Write tasks, referencing Feature Summary items
├── 4. Write Coverage section at end of plan
├── 5. Read previous plan's Coverage for carried-forward deferrals (if V2+)
├── 6. RRI-T PLAN_REVIEW (personas receive plan + Feature Summary as context)
│        → writes investigations/{date}-{topic}-plan-review-{uid}.md
├── 7. Present PLAN_REVIEW findings → user decides
├── 8. NEW: Independent Coverage Agent checks Feature Summary vs plan
│        → PASS: proceed
│        → FAIL: loop back to plan missing features, re-check (max 2 loops)
├── 9. Commit plan
└── 10. Phase gate

EXECUTION (no change)
├── 1. Execute tasks from plan
├── 2. Code review between tasks
├── 3. RRI-T POST_CODE_VERIFY
│        → writes investigations/{date}-{topic}-post-verify-{uid}.md
└── 4. Release gate
```

---

## Part 6: Triage — How Brainstorming Handles Different Scenarios

When the user triggers brainstorming, step 1 scans for existing docs. If related docs exist, a triage question routes to the right path.

### Step 1b: Triage Question

Presented to user when existing related docs are found:

> "I found existing design and plan for [topic]. What kind of change is this?"
>
> **(a) Small implementation change** (format change, library swap, parameter tweak)
> → Skip brainstorming. Edit the plan or go straight to execution.
>
> **(b) New sub-feature addition** (add capability to existing feature)
> → Focused brainstorming: reads previous design for context, produces a small focused design doc for JUST the addition.
>
> **(c) Major redesign** (rethink how the feature works)
> → Full brainstorming pipeline from scratch.

**If no existing docs found:** Skip triage, go straight to full brainstorming.

### How Each Scenario Produces Files

| Scenario | Design Doc | Plan Doc |
|----------|-----------|----------|
| New feature ("Build CRM") | `2026-03-03-crm-system-design.md` | `2026-03-03-crm-system-plan.md` |
| Sub-feature ("Add bulk export") | `2026-04-15-order-bulk-export-design.md` (references previous) | `2026-04-15-order-bulk-export-plan.md` |
| V2 pickup ("Do deferred invoice") | `2026-05-01-invoice-system-design.md` | `2026-05-01-invoice-system-plan.md` |
| Small change ("CSV to Excel") | No design doc needed | Edit existing plan or direct execution |
| Mid-execution adjustment | Depends on size: (a) edit plan directly or (b) small focused design doc | Edit plan or new plan |

**Key principle:** Each brainstorming session produces its own dated doc pair. Previous docs are never modified. Git shows the evolution. No versioning needed — the date prefix tells you which is newer.

---

## Part 7: Deferral Tracking Across Versions

### How Deferrals Flow

```
V1 Plan Coverage:
  F1-F3: Planned
  F4: Deferred — API unavailable
  F5: Deferred — out of MVP scope

V2 Brainstorming starts:
  → AI reads V1 plan's Coverage section (~10 lines)
  → "2 items deferred last time: F4 (Invoice), F5 (Dashboard). In scope now?"
  → User decides per item: pick up / keep deferred / drop
  → New design doc produced with its own Feature Summary

V2 Plan Coverage:
  F4: Planned → Task 1-4 (picked up from V1)
  F6: Planned → Task 5-8 (new feature)
  F5: Deferred — still out of scope (carried from V1)
  F7: Deferred — depends on F6 going live
```

### Rules

- Each plan's Coverage section is **self-contained** — includes current features + carried-forward deferrals
- No chaining through multiple plans — the latest plan always has the full picture
- User actively decides per deferred item: pick up, keep deferred, or drop
- "Dropped" items are recorded once with reason, then not carried forward

---

## Part 8: Context Efficiency Rule

> **Rule:** All `.md` files under `docs/` must be read via Explore subagent, which returns only relevant sections to the main agent. Source code and everything else: native Claude Code behavior, no interference.

**Why this rule exists:** Design docs can be 1000+ lines. Reading them directly into the main agent's context leaves less room for actual work. The Explore subagent reads the full doc in its own disposable context and returns only what's needed (Feature Summary, Coverage, specific sections).

**What this means in practice:**
- AI needs `src/orders/handler.ts` → reads it directly (native behavior)
- AI needs `docs/plans/2026-03-03-crm-design.md` → Explore subagent reads it, returns compact summary
- AI needs `investigations/2026-03-15-discover.md` → reads it directly (not under `docs/`, already compact)

**Where this rule lives:** In the brainstorming and writing-plans skill files. Can also be added to CLAUDE.md for global enforcement.

---

## Part 9: PLAN_REVIEW Changes

RRI-T PLAN_REVIEW personas now receive the Feature Summary table alongside the plan as **context for better domain review** — NOT as a coverage check task.

**What changes:**
- `{ARTIFACT_CONTENT}` includes the plan AND the Feature Summary table
- Personas use the Feature Summary to understand which features exist (improves their domain recommendations)
- Personas do NOT do coverage counting — that's the coverage agent's job

**Example:** The Security persona sees "F4: Invoice System" in the Feature Summary and checks "are invoice API endpoints auth-protected?" This is better domain review because the persona knows what features to look for.

**What does NOT change:**
- Persona prompts still focus on their domain (UX, business rules, QA, DevOps, security)
- No new "coverage check" instruction added to personas
- Findings format stays the same

---

## Part 10: How This Resolves The 13 Discovery Findings

| # | Finding | Resolution |
|---|---------|-----------|
| A1 | No lightweight amendment path | Triage step routes small changes away from full brainstorming |
| A2 | Version semantics undefined | Each session = new dated doc pair. No versioning needed. |
| A3 | Design-manifest drift | Feature Summary is IN the design doc. Cannot drift. |
| A4 | Lifecycle states incomplete | Coverage section in plan. States can be extended later. |
| B1 | Multi-feature doesn't fit per-feature | Flat files. One design doc can have 10 features. |
| B2 | Cross-feature deps untracked | Noted in design doc prose. Not structural (YAGNI). |
| C1 | Self-verifying trust chain | Independent coverage agent (different agent verifies). |
| C2 | Deferred pickup is best-effort | Previous plan's Coverage + count presentation. |
| C3 | Write-access boundaries | Each stage writes its own file. No shared mutable state. |
| D1 | Feature identity fragile | F# in Feature Summary. No slugs, no folders. |
| D2 | Interrupted pipeline corrupt state | Coverage agent catches gaps. Loop-back plans missing features. |
| D3 | Registry manual burden | No registry. Glob finds everything. |
| D4 | Git commit noise | 2 commits per session (design + plan). |

---

## Part 11: What Gets Modified

| File | Change |
|------|--------|
| `skills/brainstorming/SKILL.md` | Add step 1b (triage), step 6b (Feature Summary extraction), context efficiency rule |
| `skills/writing-plans/SKILL.md` | Read Feature Summary first, write Coverage section, spawn coverage agent after PLAN_REVIEW, read previous plan for deferrals, context efficiency rule |
| `skills/rri-t/SKILL.md` | Consolidated investigation files, new file paths (`investigations/`), pass Feature Summary as context to PLAN_REVIEW personas |
| `skills/rri-t/persona-prompts/*.md` | Update PLAN_REVIEW artifact content to include Feature Summary (as context, not coverage check duty) |

### What Gets Created

| Artifact | Location | Purpose |
|----------|----------|---------|
| Investigation files | `investigations/{date}-{topic}-{phase}-{uid}.md` | Per-task RRI-T evidence (consolidated, 1 file not 6) |

### What Gets Removed

| Artifact | Reason |
|----------|--------|
| `.claude/rri-t/` directory | Replaced by `investigations/` |
| Per-feature folders proposal | Superseded by this design |
| Manifest.md proposal | Superseded by Feature Summary section |
| Registry.md proposal | Unnecessary |

---

## Summary

**Before (original proposal):** 4-5 new files per feature, complex lifecycle rules, AI self-verification, no iteration support.

**After (this design):** Zero new file types. Feature Summary = section in design doc. Coverage = section in plan doc. Independent coverage agent = quality gate. Existing file structure unchanged.

**Context cost:** ~20-30 lines of additional content in design + plan docs. No extra files to read.
