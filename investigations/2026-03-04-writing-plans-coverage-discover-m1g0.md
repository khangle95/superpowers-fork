# Investigation: Writing Plans Coverage Fix

> Date: 2026-03-02 | Topic: writing-plans-coverage | Phase: discover | UID: m1g0

## End User Findings

- [MISSING] No design-doc feature inventory step before plan drafting -- SKILL.md:6-18
  - Resolution: pending

- [MISSING] No coverage checklist or traceability artifact is produced -- SKILL.md:6-135
  - Resolution: pending

- [MISSING] No explicit deferral mechanism exists -- SKILL.md:6-135
  - Resolution: pending

- [MISSING] RRI-T PLAN_REVIEW phase has no feature-coverage check instruction -- SKILL.md:97-108
  - Resolution: pending

- [MISSING] No input validation on design doc before planning begins -- SKILL.md:14-18
  - Resolution: pending

- [PAINFUL] Plan header has no field linking back to source design doc -- SKILL.md:33-45
  - Resolution: pending

- [PAINFUL] Phase gate "Clear Context and Start Execution" clears context before coverage verification -- SKILL.md:114-126
  - Resolution: pending

- [PAINFUL] No summary count of tasks vs design sections surfaced before phase gate -- SKILL.md:110-119
  - Resolution: pending

- [PAINFUL] PLAN_REVIEW subagent receives design doc as {ARTIFACT_CONTENT} which is likely truncated for long docs -- rri-t/SKILL.md:136-137
  - Resolution: pending

## BA Findings

- [MISSING] No feature enumeration step before drafting -- SKILL.md:8-10
  - Resolution: pending

- [MISSING] No coverage checkpoint after drafting -- SKILL.md:97-101
  - Resolution: pending

- [MISSING] RRI-T PLAN_REVIEW receives the plan, not the design doc paired with the plan -- rri-t/SKILL.md:136
  - Resolution: pending

- [MISSING] No deferral tracking mechanism -- SKILL.md:29-45
  - Resolution: pending

- [MISSING] No design doc cross-reference in plan header -- SKILL.md:33-45
  - Resolution: pending

- [PAINFUL] Overview instructs "give them the whole plan" but no structural rule for deriving tasks from design -- SKILL.md:10
  - Resolution: pending

- [PAINFUL] RRI-T BA persona PLAN_REVIEW checks business rules but has no design feature list to compare against -- rri-t/SKILL.md:136; persona-prompts/ba.md:68-70
  - Resolution: pending

- [MISSING] No rule requiring plan to account for all top-level sections of design doc -- SKILL.md:91-95
  - Resolution: pending

## QA Destroyer Findings

- [MISSING] No feature enumeration checkpoint before planning begins -- no ground-truth list to check against
  - Resolution: pending

- [MISSING] No instruction to materialize explicit feature list as intermediate artifact -- jumps from "read design" to "write tasks"
  - Resolution: pending

- [MISSING] No "features planned vs features in design" count during or after planning pass
  - Resolution: pending

- [PAINFUL] Design doc passed as {ARTIFACT_CONTENT} in PLAN_REVIEW -- 1800-line design + 400-line plan exceeds comfortable context for subagents
  - Resolution: pending

- [MISSING] No persona in PLAN_REVIEW assigned coverage completeness verification
  - Resolution: pending

- [MISSING] PLAN_REVIEW artifact is the plan, not the design -- dropped features invisible to reviewers
  - Resolution: pending

- [MISSING] No persona's PLAN_REVIEW focus axis includes coverage completeness
  - Resolution: pending

- [PAINFUL] PLAN_REVIEW lead aggregation produces no coverage score -- MISSING counts refer to deficiencies within tasks, not absent features
  - Resolution: pending

- [MISSING] No deferral construct in plan format
  - Resolution: pending

- [MISSING] No distinction between "intentionally deferred," "out of scope," and "accidentally omitted"
  - Resolution: pending

- [MISSING] Phase gate does not ask user to confirm feature coverage before execution
  - Resolution: pending

- [MISSING] Brainstorming-to-writing-plans handoff passes only prose -- no structured feature list
  - Resolution: pending

- [PAINFUL] "Clear Context" path discards verbal clarifications never written into design doc
  - Resolution: pending

- [PAINFUL] 5 parallel PLAN_REVIEW subagents do not increase coverage detection -- gap falls between all five
  - Resolution: pending

- [MISSING] No recovery path for discovering dropped features post-execution
  - Resolution: pending

- [MISSING] No handling for non-standard design doc formats (tables, JSON, diagrams)
  - Resolution: pending

- [MISSING] No handling for cross-referenced or dependent features across large docs
  - Resolution: pending

## DevOps Findings

- [MISSING] No context window capacity check before plan generation
  - Resolution: pending

- [MISSING] No feature inventory / manifest step
  - Resolution: pending

- [MISSING] No completion assertion at plan end -- no "N of M features planned" marker
  - Resolution: pending

- [MISSING] RRI-T PLAN_REVIEW has no coverage gate -- no persona audits plan-vs-design completeness
  - Resolution: pending

- [MISSING] No deferral tracking mechanism
  - Resolution: pending

- [PAINFUL] Plan generation is single uninterrupted write pass -- no checkpoints for large docs
  - Resolution: pending

- [PAINFUL] Context window pressure cumulative across pipeline -- design doc alone may saturate context
  - Resolution: pending

- [PAINFUL] "Clear Context and Start Planning" addresses wrong resource consumer -- design doc itself is large
  - Resolution: pending

- [MISSING] No size classification or routing for large design docs -- treats 50-line and 2000-line identically
  - Resolution: pending

- [PAINFUL] Recovery after dropped features requires manual diff -- no audit or gap-fill mode
  - Resolution: pending

- [MISSING] No handoff integrity check between writing-plans and executing-plans
  - Resolution: pending

## Security Findings

- [MISSING] No feature enumeration before plan authoring -- no checksum or manifest of design contents
  - Resolution: pending

- [MISSING] No design-vs-plan coverage verification step -- PLAN_REVIEW reviews plan against itself, not against design
  - Resolution: pending

- [MISSING] No deferral mechanism -- intentional and accidental omissions indistinguishable
  - Resolution: pending

- [MISSING] Execution phase has no path back to design doc -- gap is permanent once execution begins
  - Resolution: pending

- [MISSING] No integrity signal to user before phase gate -- user approves execution without coverage confirmation
  - Resolution: pending

- [PAINFUL] Context window saturation predictable but undetected -- no early warning for large docs
  - Resolution: pending

- [PAINFUL] Plan header has no coverage manifest -- no "features planned" or "features deferred" field
  - Resolution: pending

- [PAINFUL] RRI-T PLAN_REVIEW subagents not instructed to retrieve design doc -- review inherits same information loss
  - Resolution: pending

## Consolidated Decisions

### Critical (MISSING) -- Structural gaps

1. **No feature manifest step** -- The skill jumps from "read design doc" to "write tasks" with no step to extract a numbered feature list first. Without a ground-truth list, there's nothing to verify completeness against. (ALL 5 personas)

2. **No coverage verification after planning** -- After the plan is drafted, no step checks "design features vs plan tasks." The plan is considered complete when the agent stops writing. (ALL 5 personas)

3. **PLAN_REVIEW receives only the plan, not the design doc** -- RRI-T review personas get {ARTIFACT_CONTENT} = the plan. They can't catch dropped features because they don't have the original design to compare against. The review gate validates internal consistency, not design fidelity. (4 of 5 personas)

4. **No deferral mechanism** -- No way to record "Feature X intentionally deferred because Y." Intentional omissions and accidental drops look identical. (4 of 5 personas)

5. **No coverage persona in PLAN_REVIEW** -- None of the 5 persona focus areas include "does this plan cover all design features?" Coverage completeness falls between all five review lenses. (3 of 5 personas)

6. **Phase gate proceeds without coverage confirmation** -- User approves execution without ever seeing "10 features in design, 8 in plan." (3 of 5 personas)

7. **No size-aware routing for large docs** -- Skill treats 50-line and 2000-line designs identically. No threshold triggers a different strategy. (2 of 5 personas)

### Painful -- Works but unreliable

8. **Single-pass plan generation** -- Large designs processed in one uninterrupted pass with no checkpoints. Context exhaustion truncates silently. (DevOps)

9. **Plan header missing design doc link** -- No provenance: which design, what version, which features covered. (End User, BA, Security)

10. **No recovery path** -- After discovering dropped features, recovery is manual and ad-hoc. No "gap fill" mode. (QA, DevOps)

11. **Brainstorming verbal clarifications lost on context clear** -- "Clear Context" path discards undocumented verbal specs. (QA)
