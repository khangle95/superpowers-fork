# RRI-T Discovery: Per-Feature Doc Structure Design Review

> **Date:** 2026-03-03
> **What was reviewed:** The proposed per-feature doc structure from `2026-03-03-system-overview-and-changes.md` and `2026-03-02-writing-plans-coverage-fix-design.md`
> **Focus:** Real-world iteration scenarios — what happens when you EDIT, ADJUST, or ADD things to an existing feature?
> **Raw findings:** 47 across 5 personas → consolidated to 13 unique issues

---

## Root Problem A: The structure doesn't support iteration

The design solves Day 1 creation but has no answer for Day 30+ when features need edits, adjustments, or additions.

### A1 [MISSING] — No lightweight amendment path

A small change (CSV→Excel) triggers the full brainstorming pipeline (5 RRI-T personas, 2-3 approaches, design sections, etc.) because the HARD-GATE has no carve-out for small edits. No "amend design.md" path exists.

- Flagged by: End User, QA

### A2 [MISSING] — Version semantics are undefined

"design-v2.md or dated" is ambiguous. plan.md has NO versioning rule at all. After V1 is built, the AI agent has no algorithm to determine which file is "current." The plan header always points to `design.md` even when `design-v2.md` exists.

- Flagged by: End User, BA, QA

### A3 [MISSING] — Design-manifest drift after edits

If design.md is edited post-approval (e.g., "drop feature Y"), the manifest still has Y as "Planned." No sync mechanism exists. Worse: execution builds Y anyway because manifest says "Planned" and nobody checks design.md again.

- Flagged by: QA, BA

### A4 [MISSING] — Lifecycle states are incomplete

Manifest only tracks: blank → Planned → Deferred. No "In Progress," "Complete," "Blocked." No skill writes back to manifest during execution. The registry example shows "Complete" but no mechanism produces it.

- Flagged by: BA

---

## Root Problem B: The 1:1 assumption is wrong

The design assumes: one brainstorming session = one feature = one folder. Reality is 1:many (one session designs many features) and many:1 (many sessions iterate one feature).

### B1 [MISSING] — Multi-feature brainstorming doesn't fit per-feature folders

"Build me a CRM system" produces ONE design covering 10 features. The structure wants 10 separate `design.md` files. No split procedure is defined. Design sections describing interrelated features lose coherence when separated.

- Flagged by: End User, BA, QA

### B2 [MISSING] — Cross-feature dependencies untracked

Feature A (Invoice Export) depends on Feature B (Order data model). No entity captures this. No skill checks dependency ordering before execution. Parallel execution can start Feature A before Feature B is ready.

- Flagged by: BA

---

## Root Problem C: The AI verifies its own work

The manifest was designed to prevent features from being dropped. But the same AI that drops features also maintains the manifest.

### C1 [MISSING] — Self-verifying trust chain

AI creates the manifest, AI updates Status during planning, AI runs the coverage check. The coverage check is structural (is Status filled?) not semantic (do the tasks actually implement the feature?). F7 marked as "Planned → Task 15-18" passes the check even if Tasks 15-18 implement F6.

- Flagged by: Security, QA

### C2 [MISSING] — Deferred feature pickup is best-effort

V2 brainstorming is supposed to present all deferred items. But if the AI presents 3 out of 5 deferred items, the user can't know 2 were omitted. Same context-exhaustion risk that caused the original problem.

- Flagged by: Security

### C3 [PAINFUL] — Write-access boundaries undefined

Multiple pipeline stages write to manifest.md (brainstorming creates it, writing-plans updates Status, gap-fill appends). No ownership rules. A V2 brainstorming session can overwrite V1 Status values that writing-plans already filled.

- Flagged by: Security

---

## Root Problem D: Operational fragility

The file management overhead may exceed the tracking value.

### D1 [MISSING] — Feature identity is fragile

The folder slug is the primary key. Renaming a feature breaks all manifest links, investigation file references, and evidence links. No collision detection: "Order Management" and "Order Management V2" both canonicalize to the same slug.

- Flagged by: BA, DevOps, Security

### D2 [MISSING] — Interrupted pipeline leaves corrupt state

If writing-plans crashes mid-chunk, manifest is half-updated. Gap-fill resumes but can create duplicate tasks with no deduplication check. No way to distinguish "planning in progress" from "planning failed."

- Flagged by: QA, BA, Security, DevOps

### D3 [PAINFUL] — Registry is a manual maintenance burden

Must be updated on every change, no enforcement, no auto-generation. If AI forgets, registry drifts silently. A derived artifact that could be auto-generated from manifest files but is treated as manually maintained.

- Flagged by: DevOps, Security, QA

### D4 [PAINFUL] — Git commit noise

Conservative estimate: 10-14 workflow commits per feature BEFORE any code is written. 5 features = 50-70 overhead commits mixed with code commits. No convention to distinguish workflow commits from code commits.

- Flagged by: DevOps

---

## Key Concerns (one per persona)

| Persona | Key Concern |
|---------|-------------|
| **End User** | Every real-world iteration scenario after V1 is either undocumented or contradicted by competing design choices |
| **BA** | No stable feature identity, no post-planning lifecycle states, no cross-feature dependency model |
| **QA Destroyer** | System can be corrupted through normal intended use (design edits, V2 planning) — not just error conditions |
| **DevOps** | Migration from current state is entirely unspecified — first real use starts with orphaned context |
| **Security** | Silent misattribution of task coverage to wrong features passes every defined gate undetected |

---

## User Decisions

*(To be filled after user review)*

| # | Finding | Decision | Notes |
|---|---------|----------|-------|
| A1 | No lightweight amendment path | | |
| A2 | Version semantics undefined | | |
| A3 | Design-manifest drift | | |
| A4 | Lifecycle states incomplete | | |
| B1 | Multi-feature doesn't fit per-feature folders | | |
| B2 | Cross-feature dependencies untracked | | |
| C1 | Self-verifying trust chain | | |
| C2 | Deferred pickup is best-effort | | |
| C3 | Write-access boundaries undefined | | |
| D1 | Feature identity is fragile | | |
| D2 | Interrupted pipeline corrupt state | | |
| D3 | Registry is manual burden | | |
| D4 | Git commit noise | | |
