# Super Bear

> Fork of [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra). All original credit goes to him.

Super Bear adds independent design validation, plan review, and structured task ordering to the Superpowers workflow. The core problem it solves: the same agent that creates a design is the one presenting it for approval — no fresh eyes, no challenge, no debate.

## What's different from Superpowers

### RRI-T Discovery (brainstorming phase)

After user Q&A, 5 persona agents (End User, BA, QA Destroyer, DevOps, Security Auditor) read the relevant code areas and identify hidden requirements **before** approaches are proposed:
- **MISSING** items — things users will need that aren't covered
- **PAINFUL** items — things that will frustrate users

Findings go **directly to you** as a checklist. Your decisions inform the design, not retroactively patch it.

**Structure:** Fire-and-forget subagents with codebase access. Orchestrated by `brainstorming/SKILL.md`.

### RRI-T Plan Review (planning phase)

After an implementation plan is drafted, the same 5 personas review it — but this time as an **agent team** with a 6th member: the **Triage Analyst**.

**Why a team?** Personas review the plan document (no codebase access). They may flag issues the codebase already handles. The Triage Analyst investigates the actual codebase and debates disputed findings directly with the persona who raised them. This resolves false positives before they reach you.

**The flow:**
1. 5 personas review the plan, tag findings as PASS / FAIL / PAINFUL / MISSING
2. Triage Analyst picks up FAIL items, investigates the codebase
3. If Triage disagrees with a finding → debates with the original persona (max 3 rounds)
4. Technical fixes auto-applied to the plan (no approval needed)
5. Requirement/scope decisions escalated to you (never assumes your intent)
6. You only see: resolved technical fixes + items needing your decision

**Structure:** Agent team (5 personas + Triage Analyst). Orchestrated by `writing-plans/SKILL.md`.

Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.

### Severity Calibration

All personas receive project context (user count, deployment model, threat model) to calibrate finding severity. A missing rate limiter on a 20-user internal ERP is not the same as on a public API. Personas must verify before marking FAIL — no theoretical concerns, no "when in doubt, flag it."

### Task Ordering

Plans follow bottom-up ordering rules:
1. **Data & types first** — schema, interfaces, configs
2. **Business logic & auth second** — services, middleware, validation
3. **Wiring third** — API routes, event handlers, skill references
4. **User-facing last** — UI, CLI output, formatting
5. **Tests & docs at the end**

Each task gets a dependency annotation — `(Independent)` or `(Depends on: Task N)` — enabling future parallel execution. Layer gates between groups verify before proceeding.

### Phase gates

Each phase transition (brainstorming → planning → execution) offers execution options. Context clearing is recommended between phases.

### Subagent skill-checking

Subagents dispatched during execution check for applicable skills before writing code.

### Renamed prefix

All skill references use `super-bear:` instead of `superpowers:`.

---

## Installation

In Claude Code, register the marketplace first:

```
/plugin marketplace add khangle95/superpowers-fork
```

Then install the plugin:

```
/plugin install super-bear@super-bear-dev
```

### Environment variable for Agent Teams

RRI-T plan review requires agent teams:

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

## The Workflow

Same pipeline as Superpowers, with review gates and task ordering added:

```
brainstorming
  → RRI-T Discovery (5 fire-and-forget persona subagents)
  → phase gate
writing-plans
  → Task ordering (bottom-up layers + dependency annotations + layer gates)
  → RRI-T Plan Review (agent team: 5 personas + Triage Analyst with debate)
  → phase gate
execution (subagent-driven-development or executing-plans)
  → Layer gate enforcement between task groups
  → subagents check for applicable skills before coding
  → code review + RRI-T Post-Code Verify (optional)
finishing-a-development-branch
```

## Key Files

| File | Purpose |
|------|---------|
| `skills/rri-t/SKILL.md` | RRI-T template library: personas, severity rules, investigation format |
| `skills/rri-t/persona-prompts/end-user.md` | End User persona template |
| `skills/rri-t/persona-prompts/ba.md` | BA persona template |
| `skills/rri-t/persona-prompts/qa-destroyer.md` | QA Destroyer persona template |
| `skills/rri-t/persona-prompts/devops.md` | DevOps persona template |
| `skills/rri-t/persona-prompts/security-auditor.md` | Security Auditor persona template |
| `skills/rri-t/persona-prompts/triage-analyst.md` | Triage Analyst template (PLAN_REVIEW only) |
| `skills/brainstorming/SKILL.md` | Brainstorming + DISCOVER orchestration |
| `skills/writing-plans/SKILL.md` | Plan writing + PLAN_REVIEW orchestration + task ordering |
| `skills/subagent-driven-development/SKILL.md` | Execution with layer gate enforcement |

## Everything else

All other skills, commands, hooks, and architecture are inherited from Superpowers. See the [original README](https://github.com/obra/superpowers) for full documentation.

## License

MIT License - see LICENSE file for details

## Credits

- **Original project:** [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra)
- **RRI-T methodology:** Created by [Lam Nguyen](https://www.facebook.com/nclamvn)
- If Superpowers has helped you, consider [sponsoring Jesse's work](https://github.com/sponsors/obra)
