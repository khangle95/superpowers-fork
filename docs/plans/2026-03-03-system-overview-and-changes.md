# Super Bear Workflow System — Overview & Planned Changes

> **For:** Khang (PM/builder)
> **Date:** 2026-03-03
> **Purpose:** Explain how the current system works, what problems we found, and what we agreed to fix.

---

## Part 1: How The Current System Works

Super Bear is a workflow framework that forces AI agents to follow a disciplined process when building software. Instead of letting the AI jump straight to coding, it goes through 3 mandatory phases:

### Phase 1: Brainstorming (Design Before Code)

```
You say: "I need feature X"
         ↓
1. Agent explores the codebase
2. Asks you clarifying questions (one at a time)
3. RRI-T DISCOVER — 5 persona agents investigate hidden requirements
4. You review their findings, approve/reject each one
5. Agent proposes 2-3 approaches with tradeoffs
6. Agent presents design sections — you approve each one
7. Design doc written and committed to git
8. Phase gate: clear context or continue
         ↓
Output: A design document (what to build and why)
```

### Phase 2: Writing Plans (Plan Before Code)

```
Agent reads the design document
         ↓
1. Breaks it into bite-sized tasks (2-5 min each)
2. Each task: write test → run test → implement → verify → commit
3. RRI-T PLAN_REVIEW — 5 persona agents review the plan
4. You review their findings, decide on issues
5. Plan committed to git
6. Phase gate: choose execution method
         ↓
Output: An implementation plan (step-by-step instructions)
```

### Phase 3: Execution (Build With Reviews)

```
Agent reads the implementation plan
         ↓
1. Picks up tasks one by one
2. For each task: write code, run tests, commit
3. Code review between tasks
4. RRI-T POST_CODE_VERIFY — 5 persona agents verify the result
         ↓
Output: Working code, tested and reviewed
```

### What is RRI-T?

RRI-T (Reverse Requirements Interview — Testing) is the quality review system. It uses 5 specialized "persona" agents, each looking at the work from a different angle:

| Persona | What they look for |
|---------|-------------------|
| **End User** | Can users actually complete their daily tasks? UX friction? |
| **BA** | Are all business rules captured? Data consistency? |
| **QA Destroyer** | What breaks? Edge cases? Error handling? |
| **DevOps** | Can this be deployed safely? Does it scale? |
| **Security** | Auth correct? Data protected? Injection risks? |

RRI-T runs at 3 checkpoints:
- **DISCOVER** (during brainstorming) — find hidden requirements before designing
- **PLAN_REVIEW** (during planning) — review the plan before coding
- **POST_CODE_VERIFY** (after coding) — verify the implementation before release

Each persona tags findings with 4 levels:
- **PASS** — looks good
- **FAIL** — must fix before proceeding
- **PAINFUL** — works but will frustrate users
- **MISSING** — not covered at all

---

## Part 2: The Problems We Found

### Problem 1: Writing-Plans Drops Features (The Original Problem)

**What happens:** When a design doc is long (1000+ lines with 10 features), the writing-plans skill only creates plan tasks for 8-9 features. The missing 1-2 features are not flagged, not deferred — they just silently disappear.

**Why it happens:** The AI agent has a limited "attention window" (context window). With a 1000+ line design doc, the agent starts writing tasks and by the time it gets through 80% of the features, it "feels done" and stops. There's no checklist to verify against.

**Why RRI-T didn't catch it:** The PLAN_REVIEW phase receives only the plan — not the original design doc. The 5 persona agents review what's IN the plan, but they can't see what's MISSING from it because they don't have the design to compare against.

**Who caught it:** You did. Manually. Which defeats the purpose of automated quality gates.

### Problem 2: No Feature Tracking Across the Pipeline

**What happens:** There's no numbered list of features that travels from design → plan → execution. The design doc describes features in prose. The plan describes tasks. Nobody cross-references them.

**Like this:**
```
Design doc says:          Plan has tasks for:         Gap:
─────────────────         ──────────────────         ─────
Feature 1 (Auth)          Task 1-3 (Auth)            ✓
Feature 2 (Orders)        Task 4-7 (Orders)          ✓
Feature 3 (Reports)       Task 8-10 (Reports)        ✓
Feature 4 (Export)        ... nothing ...             ✗ SILENTLY DROPPED
Feature 5 (Dashboard)     Task 11-13 (Dashboard)     ✓
```

Nobody knows Feature 4 was dropped until you manually compare the two documents.

### Problem 3: "Intentionally Skipped" Looks the Same as "Accidentally Forgotten"

**What happens:** Sometimes you decide to defer a feature (e.g., "we'll do Export in V2"). Sometimes the agent forgets a feature. Both produce the same result: the feature isn't in the plan. There's no way to tell the difference.

### Problem 4: RRI-T Files Are Hard to Access

**What happens:** RRI-T findings live in `.claude/rri-t/` — a hidden directory inside the Claude config folder. This causes:
- Permission errors when subagents try to write
- Hard to browse visually
- Mixed with Claude system files

### Problem 5: RRI-T Creates Too Many Files

**What happens:** Each RRI-T audit creates 6 files (1 per persona + 1 lead summary). These files accumulate across phases. After a full cycle (DISCOVER + PLAN_REVIEW + POST_CODE_VERIFY), one feature has 6 files with mixed findings from 3 phases. Old findings clutter new investigations.

---

## Part 3: What We Agreed To Fix

### Fix 1: Feature Manifest (Solves Problems 1, 2, 3)

**What:** A numbered checklist of all features, created during brainstorming and used as ground-truth throughout the pipeline.

**How it works:**

```
BRAINSTORMING (new step 6b, after design approved):
         ↓
Agent extracts numbered feature list from approved design
         ↓
    "I found 10 features. Here they are:"
    F1. User Authentication
    F2. Order Management
    F3. Report Export
    ...
         ↓
You confirm: "Yes, that's complete" (can add missing ones)
         ↓
Manifest saved as docs/features/{feature}/manifest.md
```

Then during WRITING-PLANS:

```
Agent reads manifest.md (not re-extracting from design)
         ↓
Plans tasks for each manifest item
         ↓
After planning: updates manifest with status
    F1: Planned → Task 1-3
    F2: Planned → Task 4-7
    F9: Deferred — API not available yet
    F10: Deferred — out of MVP scope
         ↓
Coverage check: "10 features: 8 planned, 2 deferred, 0 unaccounted"
         ↓
You see the coverage BEFORE approving execution
```

**Why this works:**
- Features are enumerated ONCE during brainstorming (richest context, nothing lost)
- Writing-plans reads the list instead of re-extracting (no more silent drops)
- Every feature must be either "Planned" or "Deferred" — nothing can be blank
- You see coverage numbers before approving execution
- "Deferred" is explicitly tracked with a reason — different from "forgotten"

### Fix 2: Chunked Planning for Large Docs (Solves Problem 1)

**What:** Instead of planning all features in one pass, the agent plans in groups.

**How it works:**
- Small designs (under ~500 lines): plan all at once (no change)
- Large designs (500+ lines): group related features, plan one group at a time
- After each group: mark those features as "planned" in the manifest
- After all groups: verify 0 unaccounted features

**Why this works:** Each group only needs the manifest + relevant design sections in context. The agent's attention stays focused instead of spreading thin across 1000+ lines.

### Fix 3: Pass Manifest to PLAN_REVIEW (Solves Problem 1)

**What:** RRI-T PLAN_REVIEW personas now receive the manifest alongside the plan.

**How it works:**
- Before: personas got only the plan → could review quality but not detect missing features
- After: personas get plan + manifest → every persona checks "does each manifest item have a plan task?" before their domain review

**Why this works:** 5 independent coverage checks. If even one persona notices a gap, it surfaces. The manifest is compact (just a table), so it barely costs extra context.

### Fix 4: Consolidated Investigation Files (Solves Problems 4, 5)

**What:** RRI-T investigations move out of `.claude/rri-t/` and use one file per audit instead of 6.

**Before:**
```
.claude/rri-t/order-management/     ← hidden directory, 6 files per audit
├── lead.md
├── end-user.md
├── ba.md
├── qa.md
├── devops.md
└── security.md
```

**After:**
```
investigations/                      ← visible, top-level, 1 file per audit
├── 2026-03-15-crm-order-management-discover-a3f2.md    ← all 5 personas in 1 file
├── 2026-03-15-crm-order-management-plan-review-a3f2.md
└── 2026-03-17-crm-order-management-post-verify-a3f2.md
```

**Key changes:**
- 5 subagents run in parallel, return their findings as output
- Lead writes ONE consolidated file (not 6 separate files)
- Subagents no longer need file write permission (fixes the permission problem)
- Each audit = new file. Always create, never append.
- The `a3f2` uid links phases of the same task together

### Fix 5: Per-Feature Doc Structure (Solves Problem 4)

**What:** Project documentation organized by feature, not by date or type.

```
docs/features/
├── order-management/
│   ├── manifest.md          ← living feature checklist
│   ├── design.md            ← latest design
│   └── plan.md              ← latest plan
├── user-authentication/
│   └── ... same structure
```

**Two levels of artifacts:**

| Level | What | Lifecycle | Location |
|-------|------|-----------|----------|
| **Project** | Manifests, designs, plans | Living, persistent | `docs/features/` |
| **Task** | RRI-T investigations | Per-task, fresh each time | `investigations/` |

Manifest links to investigation files as evidence:
```
| F9 | Invoice Export | Deferred — API unavailable | [why](../../investigations/2026-03-15-crm-order-mgmt-discover-a3f2.md) |
```

### Fix 6: Small But Important Additions

**Context clear safety:** Before clearing context, agent confirms the design doc captures everything discussed verbally. Prevents losing undocumented specs.

**Cross-version manifest:** When brainstorming starts on an existing feature, agent reads the current manifest first. Presents deferred items: "These were deferred last time — in scope now?" Prevents deferred features from silently disappearing across versions.

**Updated plan header:** Plan document now includes links to design doc and manifest, plus coverage summary. Anyone reading the plan knows where it came from and what's covered.

---

## Part 4: What the Updated Pipeline Looks Like

```
BRAINSTORMING
├── 1. Explore codebase
├── 2. Ask questions (one at a time)
├── 3. RRI-T DISCOVER → writes investigations/{date}-{module}-{feature}-discover-{uid}.md
├── 4. Present findings → you approve/reject
├── 5. Propose approaches → you pick
├── 6. Present design → you approve each section
├── 6b. NEW: Extract Feature Manifest → you confirm completeness
├── 6c. NEW: Capture verbal clarifications if clearing context
├── 7. Write design doc + manifest.md → commit
└── 8. Phase gate

WRITING PLANS
├── 1. NEW: Read manifest.md (not re-extracting from design)
├── 2. NEW: If large doc → chunk by feature groups
├── 3. Write tasks per chunk, update manifest Status after each
├── 4. NEW: Coverage verification — 0 unaccounted required
├── 5. RRI-T PLAN_REVIEW (now receives manifest too)
│        → writes investigations/{date}-{module}-{feature}-plan-review-{uid}.md
├── 6. Present findings → you decide
├── 7. Commit plan
└── 8. Phase gate

EXECUTION
├── 1. Execute tasks from plan
├── 2. Code review between tasks
├── 3. RRI-T POST_CODE_VERIFY
│        → writes investigations/{date}-{module}-{feature}-post-verify-{uid}.md
└── 4. Release gate
```

---

## Part 5: What We Deferred (V2+)

These are valid but not needed for the first working version:

| Item | Why deferred |
|------|-------------|
| **Registry file** (`docs/features/registry.md`) | Not needed until 10+ features. Glob finds manifests fine for now. |
| **Gap-fill mode** | Recovery after failure. V1 focuses on prevention. Add after prevention proves itself. |
| **Feature split/merge/abandon rules** | Edge cases. Design when they actually happen. |

---

## Part 6: What Gets Modified

| File | Type of change |
|------|---------------|
| `skills/brainstorming/SKILL.md` | Add manifest extraction step (6b), verbal clarification step, read existing manifest for V2 |
| `skills/writing-plans/SKILL.md` | Read manifest, chunked planning, coverage verification, write-back to manifest, updated plan header, deferred features section |
| `skills/rri-t/SKILL.md` | Consolidated investigation files (1 not 6), new file paths, new naming convention |
| `skills/rri-t/persona-prompts/*.md` | Add coverage check instruction to PLAN_REVIEW for all 5 personas |
| `.claude/rri-t/` | Migrate existing contents to `investigations/`, then remove |
