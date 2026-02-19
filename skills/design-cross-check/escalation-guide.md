# Escalation Guide: Translating Technical Disagreements to Product Language

This is a reference document for the Team Lead when translating unresolved Author/Reviewer disagreements into questions the user (a product designer, not a coder) can answer.

## Core Principle

The user cares about what their product does for people, not how it's built. Every technical disagreement maps to a product tradeoff. Your job is to find that mapping.

## Translation Examples

| Technical Disagreement | Product Language |
|------------------------|-----------------|
| "Should we use a single API or separate endpoints?" | "Should users do everything in one place, or have separate screens for each action?" |
| "This design is over-engineered" | "We could build a simpler version that covers 80% of cases now, or a more complete version that takes longer." |
| "Missing error handling for edge case X" | "What should happen if a user does [X]? Right now the design doesn't say." |
| "Task ordering has a dependency problem" | "We need to build [A] before [B] can work. The current order has [B] first." |
| "This should use caching for performance" | "This feature might feel slow for users with lots of data. Should we invest extra time to make it fast, or ship the simpler version first?" |
| "The data model doesn't support future requirements" | "The way we're building this works for what you need now, but would need to be rebuilt if you later want [feature]. Build for now, or invest more upfront?" |
| "We need input validation on this form" | "What should happen if someone enters something unexpected here — like a really long name, or special characters?" |
| "This task is too large to implement atomically" | "This piece of work is big enough that it might be safer to build it in two smaller steps instead of one big one." |

## Words to Avoid

Never use these in user-facing escalations:

API, endpoint, database, schema, module, component, refactor, architecture, interface, abstraction, file path, function name, class, method, middleware, service layer, ORM, migration, deploy, CI/CD, linter, type system, generic, polymorphism, dependency injection

## Words to Use Instead

- "screen" or "page" instead of "component" or "view"
- "saves/stores" instead of "persists to database"
- "checks" instead of "validates"
- "breaks" or "stops working" instead of "throws an error"
- "the system" or "your product" instead of "the service" or "the backend"
- "users will see" instead of "the response returns"
- "takes longer to build" instead of "increases complexity"

## Escalation Template

```
The team has a question for you:

**What they agree on:** [summary of common ground — what IS clear]

**Where they see it differently:**
- One view: [framed as what users experience or what happens to the product]
- Other view: [framed as what users experience or what happens to the product]

**What this means for your product:** [plain language tradeoff — time, scope, or user experience]

**Your call:** [A simple question with 2-3 clear options the user can pick from]
```

## Batching Escalations

If 2+ concerns escalate, batch them into a single user interaction. Don't ask the user to context-switch multiple times.

Format:

```
The team has a few questions for you:

**1. [First concern label]**

What they agree on: [common ground]
Where they differ: [one view] vs. [other view]
Your call: [question]

**2. [Second concern label]**

What they agree on: [common ground]
Where they differ: [one view] vs. [other view]
Your call: [question]
```

Present all escalated concerns at once using `AskUserQuestion` with one question per concern, so the user can answer them all in one pass.

## Merging Related Escalations

Before presenting escalations to the user, check if any map to the same product decision. If two concerns are really the same question from different angles, merge them into one escalation.

**Example of mergeable concerns:**
- Concern A: "Task 3 doesn't handle users with no data"
- Concern B: "Task 5 assumes data already exists on first run"

Both map to: "What should happen the very first time someone uses this feature, when they have no existing data?"

**Rule:** If answering one concern would automatically answer another, merge them. The user shouldn't have to answer the same question twice.
