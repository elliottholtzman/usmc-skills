# USMC Claude Code Skills

Claude Code skills that automate admin tasks for Marine leaders. Each skill is a self-contained folder you drop into your Claude Code skills directory.

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop app, or web)
- [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/) MCP extension (for browser-based skills)
- CAC reader + inserted CAC (for .mil system access)

## Available Skills

| Skill | Description |
|-------|-------------|
| [pull-receipts](pull-receipts/) | Pull IIF/CIF gear receipts from ELMS for individual Marines or an entire roster. Handles login, search, download, and file naming. |

## Installation

1. Copy the skill folder you want into your Claude Code skills directory:

```bash
# Example: install the pull-receipts skill
cp -r pull-receipts/ ~/.claude/skills/pull-receipts/
```

2. Restart Claude Code (or start a new session).
3. Use the skill by typing `/pull-receipts` in Claude Code, or just describe what you need — e.g., "pull gear receipts for my platoon."

## Adding More Skills

Each folder contains a `SKILL.md` that defines what the skill does and how it works. To install any new skill added to this repo, just copy its folder into `~/.claude/skills/`.

## License

MIT
