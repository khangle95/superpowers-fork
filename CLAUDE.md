# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Superpowers is a composable skills system for AI coding agents (Claude Code, Cursor, Codex, OpenCode). It provides workflow skills (TDD, debugging, brainstorming, planning, code review) that agents automatically invoke during development. This is **not** a compiled application — it's a plugin/configuration project distributed via git clone and symlinks.

**Current version:** 4.3.0 (see `.claude-plugin/plugin.json`)

## Running Tests

Tests invoke Claude Code CLI in headless mode and verify behavior through session transcripts. All tests require the `claude` CLI to be installed.

```bash
# Run fast skill tests (~2 min each)
./tests/claude-code/run-skill-tests.sh

# Run a specific test
./tests/claude-code/run-skill-tests.sh --test test-subagent-driven-development.sh

# Run integration tests (slow, 10-30 min, real Claude sessions)
./tests/claude-code/run-skill-tests.sh --integration

# Verbose output
./tests/claude-code/run-skill-tests.sh --verbose

# Custom timeout (seconds)
./tests/claude-code/run-skill-tests.sh --timeout 1800

# Analyze token usage from a session transcript
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<project-dir>/<session-id>.jsonl
```

Tests must be run **from the superpowers directory** (not temp dirs) so skills load correctly. The local dev marketplace must be enabled in `~/.claude/settings.json` (`"super-bear@super-bear-dev": true`).

## Architecture

### Skill System

Each skill lives in `skills/<skill-name>/SKILL.md` with YAML frontmatter:
```yaml
---
name: skill-name
description: Use when [trigger condition] - [what it does]
---
```

Skills are loaded via the `Skill` tool at runtime. The `using-superpowers` skill (injected at session start via hook) teaches agents how to discover and invoke other skills. Personal skills in `~/.claude/skills/` shadow superpowers skills of the same name.

### Hooks

`hooks/hooks.json` defines a synchronous `SessionStart` hook that runs `hooks/session-start.sh`. This script reads the `using-superpowers` skill content, escapes it for JSON, and injects it as additional context so agents know about the skills system from the first turn. The hook outputs JSON compatible with both Claude Code and Cursor formats.

### Core Library

`lib/skills-core.js` provides skill management utilities: frontmatter parsing (`extractFrontmatter`), skill discovery (`findSkillsInDir`), path resolution with personal skill shadowing (`resolveSkillPath`), update checking, and frontmatter stripping. Used by platform adapters (OpenCode, Codex).

### Multi-Platform Support

- **Claude Code**: `.claude-plugin/plugin.json` — plugin manifest, hooks, skills auto-discovered
- **Cursor**: `.cursor-plugin/plugin.json` — plugin manifest with explicit paths to skills/agents/commands/hooks
- **Codex**: `.codex/INSTALL.md` — fetch-and-follow install, native skill discovery via `~/.agents/skills/` symlinks
- **OpenCode**: `.opencode/plugins/superpowers.js` — JS plugin adapter using `skills-core.js`

### Commands and Agents

- `commands/` — Slash commands (`/brainstorm`, `/write-plan`, `/execute-plan`) that invoke skills
- `agents/code-reviewer.md` — Custom agent for plan-aligned code review with severity categorization

### Test Framework

`tests/claude-code/test-helpers.sh` provides: `run_claude` (headless CLI execution), `assert_contains`, `assert_not_contains`, `assert_count`, `assert_order`, `create_test_project`, `create_test_plan`. Tests parse `.jsonl` session transcripts, not user-facing output.

## Skill Workflow Order

The skills form a pipeline: **brainstorming** → **using-git-worktrees** → **writing-plans** → **subagent-driven-development** (or **executing-plans**) → **test-driven-development** (during implementation) → **requesting-code-review** → **finishing-a-development-branch**. Brainstorming is enforced with hard gates before implementation.

## Key Conventions

- Skills use YAML frontmatter with `name` and `description` fields — `description` must start with "Use when" to define trigger conditions
- The `writing-skills` skill defines how to create/test new skills — follow it for any skill changes
- Hook JSON escaping uses bash parameter substitution (not character-by-character loops) for performance
- Session transcripts are JSONL at `~/.claude/projects/<encoded-path>/<session-id>.jsonl`
- Version is tracked in both `.claude-plugin/plugin.json` and `.cursor-plugin/plugin.json`
