# Super Bear

> Fork of [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra). All original credit goes to him.

Super Bear adds independent design validation and plan review to the Superpowers workflow. The core problem it solves: the same agent that creates a design is the one presenting it for approval — no fresh eyes, no challenge, no debate.

## What's different from Superpowers

### New: Design sanity check (brainstorming phase)

After you approve a design, a `design-reviewer` subagent reads it with fresh eyes and produces:
- A **readback** — restating the design in their own words to catch ambiguity
- **Yes/no questions** about assumptions the design makes that you never confirmed

Output goes **directly to you**, not filtered by the agent that created the design (who may have made the wrong assumptions).

### New: Agent Team debate (planning phase)

After an implementation plan is drafted, a `design-cross-check` skill spawns an Agent Team:
- **Author** defends the plan
- **Reviewer** challenges it (pre-mortem framing, steel-manning)
- They debate via peer DMs (keeps the lead's context lean)
- Max 3 rounds per concern, max 7 concerns
- Unresolved disagreements escalate to you in **plain product language** (no code jargon)
- Author applies accepted changes directly to the plan

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
  → design-reviewer sanity check (readback + assumption questions)
  → phase gate
writing-plans
  → design-cross-check (Agent Team debate)
  → phase gate
execution (subagent-driven-development or executing-plans)
  → subagents check for applicable skills before coding
  → code review
finishing-a-development-branch
```

## New Files

| File | Purpose |
|------|---------|
| `agents/design-reviewer.md` | Lightweight readback + assumption check agent |
| `skills/design-cross-check/SKILL.md` | Agent Team debate orchestration |
| `skills/design-cross-check/author-teammate-prompt.md` | Author spawn template |
| `skills/design-cross-check/reviewer-teammate-prompt.md` | Reviewer spawn template (pre-mortem + steel-manning) |
| `skills/design-cross-check/escalation-guide.md` | Translation guide for user-facing escalations |

## Everything else

All other skills, commands, hooks, and architecture are inherited from Superpowers. See the [original README](https://github.com/obra/superpowers) for full documentation.

## License

MIT License - see LICENSE file for details

## Credits

- **Original project:** [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra)
- If Superpowers has helped you, consider [sponsoring Jesse's work](https://github.com/sponsors/obra)
