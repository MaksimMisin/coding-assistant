# coding-assistant

A [Claude Code](https://claude.ai/code) skill that teaches Claude how to use Gemini CLI, Codex CLI, and OpenCode as external coding agents — picking the right tool, running them in parallel, and synthesizing results.

## Requirements

- [Gemini CLI](https://github.com/google-gemini/gemini-cli)
- [Codex CLI](https://github.com/openai/codex)
- [OpenCode](https://opencode.ai) — for free/budget models

## Installation

```bash
mkdir -p ~/.claude/skills/coding-assistant/references
cp SKILL.md ~/.claude/skills/coding-assistant/
cp references/cli-reference.md ~/.claude/skills/coding-assistant/references/
```

## Usage

- "gemini review this file for bugs"
- "codex refactor auth.py and api.py"
- "gemini and codex both review this, get a second opinion"
- "opencode minimax implement this"
