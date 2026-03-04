# Simplified Coverage Tracking — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use super-bear:executing-plans to implement this plan task-by-task.

**Goal:** Implement the simplified coverage tracking system: Feature Summary in design docs, Coverage section in plans, independent coverage agent, consolidated investigation files, and context efficiency rule.

**Design:** `docs/plans/2026-03-04-simplified-coverage-tracking-design.md`

**Architecture:** All changes are to skill markdown files (SKILL.md) and templates. No code changes. The skills are instruction files that AI agents follow — edits change agent behavior, not runtime code.

**Tech Stack:** Markdown (skill files), Claude Code native tools (Agent/Explore subagent)

---

### Task 1: Update brainstorming/SKILL.md — Context Efficiency Rule

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

**Step 1: Add context efficiency rule to Key Principles section**

At the end of the `## Key Principles` section (after "Be flexible"), add:

```markdown
- **Context efficiency** - All `.md` files under `docs/` must be read via Explore subagent, which returns only relevant sections. Source code and everything else: native Claude Code behavior, no interference.
```

**Step 2: Verify the edit**

Read `skills/brainstorming/SKILL.md` and confirm the new principle is in the Key Principles list.

**Step 3: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): add context efficiency rule"
```

---

### Task 2: Update brainstorming/SKILL.md — Triage Step

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

**Step 1: Update the checklist to add triage**

In the `## Checklist` section, modify items 1 and 2. Replace the existing step 1:

```markdown
1. **Explore project context** — check files, docs, recent commits
```

With:

```markdown
1. **Explore project context** — use Explore subagent to scan `docs/plans/` for related docs (returns Feature Summary and Coverage sections only, not full docs). Read relevant source code directly (native behavior).
1b. **Triage (if existing docs found)** — present the user with: (a) Small implementation change → skip brainstorming, edit plan or go to execution; (b) New sub-feature addition → focused brainstorming with previous context; (c) Major redesign → full brainstorming. If no existing docs found, skip triage.
```

**Step 2: Add triage details after "The Process" section**

After the `## The Process` section's "Understanding the idea" subsection, add a new subsection:

```markdown
**Triage (step 1b):**
- Only triggers when Explore subagent finds related docs in `docs/plans/`
- Present as AskUserQuestion with 3 options:
  - **(a) Small implementation change** (format, library, parameter) → exit brainstorming, suggest plan edit or direct execution
  - **(b) New sub-feature addition** (new capability on existing feature) → focused brainstorming reading previous design for context, produces new dated design doc for just the addition
  - **(c) Major redesign** (rethink the feature) → full brainstorming from scratch
- If no existing docs: skip triage entirely, proceed to step 2
```

**Step 3: Update the process flow diagram**

In the `digraph brainstorming` section, add triage nodes after "Explore project context":

```dot
    "Explore project context" -> "Existing docs found?" ;
    "Existing docs found?" -> "Triage (AskUserQuestion)" [label="yes"];
    "Existing docs found?" -> "Ask clarifying questions" [label="no"];
    "Triage (AskUserQuestion)" -> "Exit brainstorming" [label="small change"];
    "Triage (AskUserQuestion)" -> "Ask clarifying questions" [label="sub-feature or redesign"];
```

Remove the direct edge from "Explore project context" to "Ask clarifying questions".

**Step 4: Verify the edit**

Read the updated file. Confirm the checklist has step 1b, the triage subsection exists, and the flow diagram includes triage.

**Step 5: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): add triage step for iteration scenarios"
```

---

### Task 3: Update brainstorming/SKILL.md — Feature Summary Extraction + HARD-GATE

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

**Step 1: Add HARD-GATE for Feature Summary**

After the existing `<HARD-GATE>` block (about no implementation before design approval), add a second HARD-GATE:

```markdown
<HARD-GATE>
The design doc MUST contain a ## Feature Summary section before it can be committed. If the design doc does not have this section, extract it now and get user confirmation before proceeding.
</HARD-GATE>
```

**Step 2: Add step 5b to the checklist**

In the `## Checklist` section, after step 5 ("Present design"), insert:

```markdown
5b. **Extract Feature Summary** — after all design sections are approved, extract a numbered feature table. Present to user: "I found N features. Is this complete?" User confirms or adds missing features. This becomes the ## Feature Summary section in the design doc.
```

Renumber existing step 6 (Write design doc) and step 7 (Phase gate) accordingly.

**Step 3: Add Feature Summary format with self-describing instruction**

In the `## After the Design` section, before the documentation subsection, add:

```markdown
**Feature Summary (step 5b):**

After all design sections are approved, extract a compact feature table and present to the user for confirmation. The Feature Summary includes an inline instruction so any agent reading the design doc knows what to do with it:

```markdown
## Feature Summary
> **For planning:** Use this table as your checklist. Every item must appear in the plan's ## Coverage section as either Planned or Deferred.

| # | Feature | Section |
|---|---------|---------|
| F1 | [Feature name] | [Section ref] |
| F2 | ... | ... |

**Total: N features**
```

Rules:
- The `> **For planning:**` instruction line makes the artifact self-describing — any agent reading the design doc knows what to do, even without loading the writing-plans skill
- User confirms completeness — can add missing features
- Immutable after commit — snapshot of what was agreed, not a living document
- This section is included in the design doc before commit
- Multi-feature designs: one table with all features, no need to split
```

**Step 3: Update the "After the Design" documentation instruction**

Change the existing line:

```markdown
- Write the validated design to `docs/plans/YYYY-MM-DD-<topic>-design.md`
```

To:

```markdown
- Write the validated design (including Feature Summary section) to `docs/plans/YYYY-MM-DD-<topic>-design.md`
```

**Step 4: Add deferral pickup for V2 brainstorming**

In the `## The Process` section, under "Understanding the idea", add:

```markdown
**Deferral pickup (V2+):**
- If triage (step 1b) identified this as a sub-feature or redesign of an existing feature, read the previous plan's ## Coverage section
- Present all deferred items with count: "N items deferred last time: [list]. In scope now?"
- User decides per item: pick up (include in new design), keep deferred (carry forward), or drop
- Picked-up items become part of the new design; carried/dropped items are noted
```

**Step 5: Update the flow diagram**

Add Feature Summary node after "User approves design?":

```dot
    "User approves design?" -> "Extract Feature Summary" [label="yes"];
    "Extract Feature Summary" -> "User confirms Feature Summary?";
    "User confirms Feature Summary?" -> "Extract Feature Summary" [label="adjust"];
    "User confirms Feature Summary?" -> "Write design doc" [label="confirmed"];
```

Remove the direct edge from "User approves design?" to "Write design doc".

**Step 6: Verify the edit**

Read the updated file. Confirm step 5b exists in checklist, Feature Summary format is documented, deferral pickup is described, and flow diagram is updated.

**Step 7: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): add Feature Summary extraction and deferral pickup"
```

---

### Task 4: Update brainstorming/SKILL.md — Update RRI-T References

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

**Step 1: Update the RRI-T findings reference**

Replace:

```markdown
**Note:** Findings persist in `.claude/rri-t/` files. The PLAN_REVIEW phase (invoked by writing-plans) spawns fresh subagents that read these files to resume context.
```

With:

```markdown
**Note:** Findings are written to `investigations/{date}-{topic}-discover-{uid}.md` as a single consolidated file (all 5 personas in one file). The lead writes this file from subagent output — subagents no longer need file write access. The PLAN_REVIEW phase reads this file for context continuity.
```

Also update the "Update findings files with user decisions" reference:

Replace:

```markdown
- Update findings files with user decisions
```

With:

```markdown
- Update the consolidated investigation file with user decisions
```

**Step 2: Verify and commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): update RRI-T references to consolidated investigation files"
```

---

### Task 5: Update writing-plans/SKILL.md — Feature Summary Read + Coverage Section + Context Rule + HARD-GATE

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Step 1: Add HARD-GATE for Coverage section**

Near the top of the file, after the overview section, add:

```markdown
<HARD-GATE>
The plan MUST have a ## Coverage section with 0 unaccounted items before proceeding to the phase gate. The independent coverage agent must return PASS. Do NOT present the phase gate until coverage is verified.
</HARD-GATE>
```

**Step 2: Add context efficiency rule**

In the `## Remember` section, add:

```markdown
- **Context efficiency** — All `.md` files under `docs/` must be read via Explore subagent (returns only relevant sections). Source code: native Claude Code behavior.
```

**Step 3: Add Feature Summary instruction**

After the `## Plan Document Header` section, add a new section:

```markdown
## Feature Summary — Read First

Before writing any tasks, read the design doc's **## Feature Summary** section. This is a compact table (10-20 lines) listing all features the design covers. Use it as your planning checklist.

**Process:**
1. Read the Feature Summary table from the design doc (use Explore subagent for the doc, but the Feature Summary is small enough to keep in main context)
2. Plan tasks for each feature in the summary
3. After all tasks are written, write the Coverage section (see below)

For large designs (500+ lines): load one design section at a time while keeping the Feature Summary table in context throughout. This prevents context exhaustion while ensuring no features are missed.
```

**Step 4: Add Coverage section format**

After the new Feature Summary section, add:

```markdown
## Coverage Section — Write Last

After all tasks are written, add a **## Coverage** section at the end of the plan. The Coverage section includes a self-describing instruction so any agent reading the plan knows what it is:

```markdown
## Coverage
> **Verification:** An independent coverage agent checks this table against the Feature Summary. Every item must be Planned or Deferred. 0 unaccounted required.

(Source: [design doc filename], Feature Summary)

| F# | Feature | Status | Tasks |
|----|---------|--------|-------|
| F1 | [name] | Planned | Task 1-3 |
| F2 | [name] | Deferred — [reason] | — |

**N planned, N deferred, 0 unaccounted**
```

**Rules:**
- The `> **Verification:**` instruction line makes the artifact self-describing — any agent reading the plan knows the Coverage section is verified by an independent agent
- Every Feature Summary item MUST have an entry (Planned or Deferred)
- Zero unaccounted items required
- Deferred items must have a reason
- If this is a V2+ plan, include carried-forward deferrals from the previous plan's Coverage section
```

**Step 5: Add deferral carry-forward instruction**

After the Coverage section, add:

```markdown
## Deferral Carry-Forward (V2+)

If the design doc references a previous design or was produced by a sub-feature/redesign triage:
1. Read the previous plan's ## Coverage section (use Explore subagent)
2. Find deferred items that are NOT in the current design's Feature Summary
3. Include them in the Coverage section as: `Deferred — [original reason] (carried from [previous plan])` or `Dropped — [reason]`
4. The Coverage section must be self-contained — the latest plan always has the full picture
```

**Step 6: Update plan header format**

In the `## Plan Document Header` section, update the header template to include design doc link and coverage summary:

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

**Step 7: Verify the edit**

Read the updated file. Confirm: HARD-GATE exists, context rule in Remember, Feature Summary section exists, Coverage section format exists with self-describing instruction, deferral carry-forward exists, updated plan header.

**Step 8: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): add Feature Summary read, Coverage section, deferral carry-forward"
```

---

### Task 6: Update writing-plans/SKILL.md — Independent Coverage Agent

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Step 1: Add coverage agent section**

After the `## RRI-T Plan Review` section, add a new section:

```markdown
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
```

**Step 2: Update the flow in RRI-T Plan Review section**

In the existing `## RRI-T Plan Review` section, change:

```markdown
**Only proceed to Phase Gate after plan review completes and user has decided on all FAIL items.**
```

To:

```markdown
**After plan review completes and user has decided on all FAIL items, run the Independent Coverage Agent (see below). Only proceed to Phase Gate after coverage agent returns PASS.**
```

**Step 3: Verify and commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): add independent coverage agent as quality gate"
```

---

### Task 7: Update rri-t/SKILL.md — Consolidated Investigation Files

**Files:**
- Modify: `skills/rri-t/SKILL.md`

**Step 1: Replace the Module Detection and Findings Directory section**

Replace the entire `## Module Detection and Findings Directory` section (including the directory structure) with:

```markdown
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
```

**Step 2: Remove the old templates reference**

Remove or replace the line:

```markdown
Use templates from this skill's directory:
- `findings-template.md` — for persona findings files
- `lead-template.md` — for lead.md summary files
```

With:

```markdown
Use the consolidated investigation file format above. The old per-persona templates (`findings-template.md`, `lead-template.md`) are no longer used.
```

**Step 3: Verify and commit**

```bash
git add skills/rri-t/SKILL.md
git commit -m "feat(rri-t): replace per-persona files with consolidated investigation files"
```

---

### Task 8: Update rri-t/SKILL.md — Update All Three Phases

**Files:**
- Modify: `skills/rri-t/SKILL.md`

**Step 1: Update DISCOVER phase**

In `## Phase: DISCOVER`, replace step 2's spawn instructions. Change the `{FINDINGS_DIR}` placeholder references to indicate subagents return output instead of writing files. Update step 3 (wait for subagents) and step 4 (aggregate) to reflect the new model:

- Step 2: Subagent prompt should say "Return your findings as your final output text. Do NOT write to any files."
- Step 3: "Each returns findings as output text" (not "writes findings to its file")
- Step 4: Replace "Read each persona's findings file / Write to `.claude/rri-t/{module}/lead.md`" with "Read each persona's returned output / Write ONE consolidated investigation file to `investigations/{date}-{topic}-discover-{uid}.md`"
- Step 6: Replace "Update findings files with user decisions" with "Update the consolidated investigation file's ## Consolidated Decisions section"

**Step 2: Update PLAN_REVIEW phase**

In `## Phase: PLAN_REVIEW`:
- Step 1: Change "Include instruction to read their existing findings file" to "Include instruction to read the previous DISCOVER investigation file if it exists: `Glob investigations/*-{topic}-discover-{uid}.md`"
- Add: "The `{ARTIFACT_CONTENT}` placeholder now includes BOTH the plan AND the Feature Summary table from the design doc. Personas use the Feature Summary as context for better domain review — they do NOT do coverage counting."
- Step 2: "Each returns findings as output text" (not "updates their findings file")
- Step 3: Replace "Aggregate into lead.md" with "Write ONE consolidated investigation file to `investigations/{date}-{topic}-plan-review-{uid}.md`"

**Step 3: Update POST_CODE_VERIFY phase**

Same pattern:
- Subagents return output, don't write files
- Lead writes one consolidated file to `investigations/{date}-{topic}-post-verify-{uid}.md`
- Include instruction to read previous investigation files for context

**Step 4: Update Context Resilience section**

Replace references to "findings files" with "investigation files in `investigations/`". Update the table:

```markdown
| Situation | What happens |
|-----------|-------------|
| Session normal | Subagents spawn, read code, return results, lead writes investigation file |
| Auto-compact | No impact — subagents are short-lived |
| Clear context / new session | Spawn fresh subagents, they read existing investigation files via Glob |
| Different project | Plugin carries skill prompts, user-profile.md provides context |
```

**Step 5: Verify and commit**

```bash
git add skills/rri-t/SKILL.md
git commit -m "feat(rri-t): update all phases for consolidated output model"
```

---

### Task 9: Update Persona Prompts — Return Output Instead of Writing Files

**Files:**
- Modify: `skills/rri-t/persona-prompts/end-user.md`
- Modify: `skills/rri-t/persona-prompts/ba.md`
- Modify: `skills/rri-t/persona-prompts/qa-destroyer.md`
- Modify: `skills/rri-t/persona-prompts/devops.md`
- Modify: `skills/rri-t/persona-prompts/security-auditor.md`

**Step 1: Update all 5 persona prompts with the same changes**

In each persona prompt file, make these edits:

a) **Remove the `{FINDINGS_DIR}` placeholder from the assignment block.** Change:
```markdown
- **Your findings file:** {FINDINGS_DIR}/[persona].md
```
To:
```markdown
- **Output:** Return findings as your final output text (do not write to files)
```

b) **Replace "Writing Findings" section.** Change each persona's `### Writing Findings` section from:
```markdown
### Writing Findings

Write to `{FINDINGS_DIR}/[persona].md` using the findings template. Each finding must have:
...
```
To:
```markdown
### Writing Findings

Return your findings as your final output text. Each finding must have:
- A severity tag: [PASS], [FAIL], [PAINFUL], or [MISSING]
- A specific description (not generic)
- A code reference if applicable (file:line)

Format each finding as a bullet point with severity tag first.
```

c) **Update PLAN_REVIEW phase instructions** to mention Feature Summary context. In the `### Phase Instructions` section, under "If PLAN_REVIEW:", add after the existing instructions:

```markdown
- **Context:** You will receive the plan AND a Feature Summary table listing all designed features. Use this to understand what features exist — it helps you give more targeted domain recommendations. You do NOT need to count features or check coverage — a separate agent handles that.
```

d) **Update the Rules section.** Change:
```markdown
- **NEVER use Bash tool** — you are a code reader, not a code runner. Use only Read, Glob, Grep, and Write (for your findings file). Never run cargo, npm, node, or any build/compile/test commands.
```
To:
```markdown
- **NEVER use Bash tool** — you are a code reader, not a code runner. Use only Read, Glob, and Grep. Never run cargo, npm, node, or any build/compile/test commands. Never write to files — return all findings as output text.
```

**Step 2: Verify each file**

Read each of the 5 files and confirm: no {FINDINGS_DIR} references, no "Write to" file instructions, PLAN_REVIEW mentions Feature Summary context, Rules section updated.

**Step 3: Commit**

```bash
git add skills/rri-t/persona-prompts/
git commit -m "feat(rri-t): update persona prompts to return output instead of writing files"
```

---

### Task 10: Update Templates — Replace Old With Consolidated Investigation Template

**Files:**
- Modify: `skills/rri-t/findings-template.md` (replace content)
- Modify: `skills/rri-t/lead-template.md` (replace content)

**Step 1: Replace findings-template.md with consolidated investigation template**

Replace the entire content of `skills/rri-t/findings-template.md` with:

```markdown
# Investigation: {TITLE}

> Date: {DATE} | Topic: {TOPIC} | Phase: {PHASE} | UID: {UID}

## End User Findings
<!-- Findings from End User persona -->

## BA Findings
<!-- Findings from BA persona -->

## QA Destroyer Findings
<!-- Findings from QA Destroyer persona -->

## DevOps Findings
<!-- Findings from DevOps persona -->

## Security Findings
<!-- Findings from Security Auditor persona -->

## Consolidated Decisions
<!-- User decisions on findings. Format:
- [Finding reference]: [approved | rejected | deferred] — [reason/notes]
-->
```

**Step 2: Remove lead-template.md**

Delete `skills/rri-t/lead-template.md` — the lead no longer writes a separate aggregation file. The consolidated investigation file replaces both the per-persona files and the lead summary.

```bash
git rm skills/rri-t/lead-template.md
```

**Step 3: Verify and commit**

```bash
git add skills/rri-t/findings-template.md
git commit -m "feat(rri-t): replace templates with consolidated investigation format"
```

---

### Task 11: Migrate Existing .claude/rri-t/ Content

**Files:**
- Read: `.claude/rri-t/writing-plans/` (existing findings)
- Create: `investigations/` directory
- Remove: `.claude/rri-t/` references

**Step 1: Check existing content**

Read the existing files in `.claude/rri-t/writing-plans/` to understand what's there.

**Step 2: Create investigations/ directory**

```bash
mkdir -p investigations
```

**Step 3: Migrate existing findings**

If the existing `.claude/rri-t/writing-plans/` files contain useful findings, consolidate them into a single investigation file:

```bash
# Create consolidated file from existing per-persona files
# Name: {date}-writing-plans-coverage-discover-{uid}.md
# Content: merge all persona findings into consolidated format
```

The exact content depends on what's in the existing files. Read each file, merge into the consolidated format, write to `investigations/`.

**Step 4: Add .gitkeep to investigations/**

```bash
touch investigations/.gitkeep
git add investigations/.gitkeep
```

**Step 5: Commit**

```bash
git commit -m "feat: create investigations/ directory and migrate existing rri-t findings"
```

---

### Task 12: Final Cleanup and Verification

**Files:**
- All modified skill files

**Step 1: Verify no remaining references to old paths**

Search for any remaining references to `.claude/rri-t/`, `manifest.md`, `registry.md`, or `docs/features/` in skill files:

```bash
grep -r "\.claude/rri-t" skills/
grep -r "manifest\.md" skills/
grep -r "registry\.md" skills/
grep -r "docs/features/" skills/
```

Fix any remaining references.

**Step 2: Verify all skill files are internally consistent**

Read each modified file and confirm:
- `skills/brainstorming/SKILL.md` — has triage, Feature Summary, context rule, new RRI-T references
- `skills/writing-plans/SKILL.md` — has Feature Summary read, Coverage section, coverage agent, deferral carry-forward, context rule
- `skills/rri-t/SKILL.md` — has consolidated investigation files, subagent output model, PLAN_REVIEW with Feature Summary context
- All 5 persona prompts — return output, no file writing, PLAN_REVIEW Feature Summary context

**Step 3: Final commit if any cleanup was needed**

```bash
git add -A
git commit -m "chore: cleanup old references to .claude/rri-t/ and manifest files"
```

---

## Coverage
(Source: 2026-03-04-simplified-coverage-tracking-design.md, Feature Summary)

| F# | Feature | Status | Tasks |
|----|---------|--------|-------|
| F1 | Update brainstorming/SKILL.md | Planned | Task 1-4 |
| F2 | Update writing-plans/SKILL.md | Planned | Task 5-6 |
| F3 | Update rri-t/SKILL.md | Planned | Task 7-8 |
| F4 | Update persona prompts | Planned | Task 9 |
| F5 | Update/create templates | Planned | Task 10 |
| F6 | Migrate .claude/rri-t/ + cleanup | Planned | Task 11-12 |

**6 planned, 0 deferred, 0 unaccounted**
