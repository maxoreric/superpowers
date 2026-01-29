# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Superpowers is a plugin system that provides composable development workflow skills to AI coding agents (Claude Code, OpenCode, Codex). It teaches agents proven software development practices including TDD, systematic debugging, and subagent-driven development.

Current version: 4.1.1

## Commands

### Testing

Integration tests run real Claude Code sessions in headless mode:

```bash
# Main integration test (10-30 minutes)
cd tests/claude-code
./test-subagent-driven-development-integration.sh

# Explicit skill request tests
cd tests/explicit-skill-requests
./run-all.sh

# Skill triggering verification
cd tests/skill-triggering
./run-test.sh
```

**Requirements:**
- Claude Code installed and available as `claude` command
- Must run FROM the superpowers directory (skills only load when running from plugin root)
- Dev marketplace enabled: `"superpowers@superpowers-dev": true` in `~/.claude/settings.json`

### Token Analysis

```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<project-dir>/<session-id>.jsonl
```

Session transcripts are in `~/.claude/projects/` with directory path encoded (e.g., `-Users-jesse-Documents-superpowers`).

### Plugin Update

```bash
/plugin update superpowers
```

## Architecture

### Skill Format

Each skill is a directory in `skills/` containing:

```
skills/skill-name/
  SKILL.md              # Main reference (required)
  supporting-file.*     # Only if needed (heavy reference, tools)
```

**SKILL.md structure:**
- YAML frontmatter: only `name` and `description` fields (max 1024 chars total)
- Description starts with "Use when..." - describes triggering conditions only, NOT workflow
- Markdown body with Overview, When to Use, Core Pattern, Quick Reference, Common Mistakes

### Skill Discovery

`lib/skills-core.js` provides skill discovery functions:
- `extractFrontmatter()` - Parse YAML frontmatter from SKILL.md
- `findSkillsInDir()` - Find all skills recursively (max depth 3)
- `resolveSkillPath()` - Resolve skill name to file path (personal skills shadow superpowers skills)
- `stripFrontmatter()` - Remove frontmatter, return content only

Personal skills override superpowers skills when names match.

### Session Hooks

`hooks/hooks.json` defines SessionStart hook that runs `session-start.sh` on startup/resume/clear/compact. The hook:
1. Reads `skills/using-superpowers/SKILL.md`
2. Injects it as `additionalContext` in the session
3. Warns if legacy `~/.config/superpowers/skills` directory exists

### Multi-Platform Support

| Platform | Plugin Entry Point | Skills Location |
|----------|-------------------|-----------------|
| Claude Code | `.claude-plugin/plugin.json` | Native plugin system |
| OpenCode | `.opencode/plugins/superpowers.js` | Symlinked to `~/.config/opencode/skills/superpowers` |
| Codex | `.codex/superpowers-codex` | Cloned to `~/.codex/superpowers` |

### Core Skills (16 total)

**Workflow:** brainstorming → using-git-worktrees → writing-plans → subagent-driven-development/executing-plans → requesting-code-review → finishing-a-development-branch

**Linear Integration:** daily-planning, picking-a-task

**Supporting:** test-driven-development, systematic-debugging, verification-before-completion, receiving-code-review, dispatching-parallel-agents, using-superpowers, writing-skills

## Writing Integration Tests

Tests parse session transcripts (`.jsonl` files) to verify behavior:

```bash
# Example: verify skill was invoked
grep -q '"name":"Skill".*"skill":"your-skill-name"' "$SESSION_FILE"

# Example: count subagent dispatches
grep -c '"name":"Task"' "$SESSION_FILE"
```

Key flags for headless testing:
- `--permission-mode bypassPermissions` - Avoid permission prompts
- `--add-dir /path/to/temp` - Grant access to test directories
- `--allowed-tools=all` - Enable all tools

## Creating New Skills

Follow TDD: write pressure scenario test → run WITHOUT skill (RED) → write minimal skill (GREEN) → close loopholes (REFACTOR).

See `skills/writing-skills/SKILL.md` for the complete methodology and `skills/writing-skills/testing-skills-with-subagents.md` for testing patterns.

Key rules:
- Description is triggering conditions only, never workflow summary
- Use letters/numbers/hyphens only in names
- One excellent code example beats multiple mediocre ones
- Test skill with subagents before deploying

## Linear Integration

### Setup

1. Set `LINEAR_API_KEY` environment variable:
   ```bash
   export LINEAR_API_KEY="lin_api_xxx"
   ```

2. Enable Linear GitHub Integration in Linear Settings → Integrations → GitHub

### Workflow

1. `/daily-planning` - Plan today's tasks from Linear backlog
2. `/picking-a-task PRJ-123` - Start work on specific task
3. Branch naming: `PRJ-123/description` (automatic via skills)
4. PR creation: Auto-includes `Fixes PRJ-123` (automatic via skills)
