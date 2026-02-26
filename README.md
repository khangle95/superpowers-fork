# Super Bear

> Fork of [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra). All original credit goes to him.

Super Bear adds independent design validation and plan review to the Superpowers workflow. The core problem it solves: the same agent that creates a design is the one presenting it for approval — no fresh eyes, no challenge, no debate.

## What's different from Superpowers

### New: RRI-T Discovery (brainstorming phase)

After user Q&A, 5 persona agents (End User, BA, QA Destroyer, DevOps, Security Auditor) read the relevant code areas and identify hidden requirements **before** approaches are proposed:
- **MISSING** items — things users will need that aren't covered
- **PAINFUL** items — things that will frustrate users

Findings go **directly to you** as a checklist. Your decisions inform the design, not retroactively patch it.

### New: RRI-T Plan Review (planning phase)

After an implementation plan is drafted, the same 5 RRI-T personas review through their specialized lenses:
- Each tags findings as **PASS** / **FAIL** / **PAINFUL** / **MISSING**
- FAIL items require your decision before proceeding
- PAINFUL items presented as tradeoffs, MISSING as scope questions
- Personas already have context from discovery — no cold start

Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.

### New: Phase gates with context clearing

Each phase transition (brainstorming → planning → execution) offers a choice:
- **Clear context and start fresh** (recommended) — gives the next phase clean, focused context
- **Continue in this session** — if you want to reference previous discussion

### New: Subagent skill-checking

Subagents dispatched during execution now check for applicable skills before writing code:
- **React/Next.js work** → invokes `vercel-react-best-practices`
- **UI/UX work** → invokes `ui-ux-pro-max` and follows design guidelines in CLAUDE.md

### Renamed prefix

All skill references use `super-bear:` instead of `superpowers:` (e.g., `super-bear:brainstorming`).

---

## Installation

### Claude Code (local dev)

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/super-bear.git

# Register the local dev marketplace
# Add to ~/.claude/settings.json:
# "super-bear@super-bear-dev": true
```

### Environment variable for Agent Teams

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

## The Workflow

Same pipeline as Superpowers, with review gates added:

```
brainstorming
  → RRI-T Discovery (5 personas identify hidden requirements)
  → phase gate
writing-plans
  → RRI-T Plan Review (5 personas review plan)
  → phase gate
execution (subagent-driven-development or executing-plans)
  → subagents check for applicable skills before coding
  → code review + RRI-T Post-Code Verify (optional)
finishing-a-development-branch
```

## New Files

| File | Purpose |
|------|---------|
| `skills/rri-t/SKILL.md` | RRI-T quality review orchestration (3-phase lifecycle) |
| `skills/rri-t/persona-prompts/end-user.md` | End User persona spawn template |
| `skills/rri-t/persona-prompts/ba.md` | BA persona spawn template |
| `skills/rri-t/persona-prompts/qa-destroyer.md` | QA Destroyer persona spawn template |
| `skills/rri-t/persona-prompts/devops.md` | DevOps persona spawn template |
| `skills/rri-t/persona-prompts/security-auditor.md` | Security Auditor persona spawn template |
| `skills/rri-t/findings-template.md` | Template for persona findings files |
| `skills/rri-t/lead-template.md` | Template for lead summary files |

## Everything else

All other skills, commands, hooks, and architecture are inherited from Superpowers. See the [original README](https://github.com/obra/superpowers) for full documentation.

## License

MIT License - see LICENSE file for details

## Credits

- **Original project:** [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra)
- **RRI-T methodology:** Created by [Lam Nguyen](https://www.facebook.com/nclamvn)
- If Superpowers has helped you, consider [sponsoring Jesse's work](https://github.com/sponsors/obra)
