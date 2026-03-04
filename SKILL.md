---
name: coding-assistant
description: "Orchestrates Gemini, Codex, and OpenCode as coding assistants. Use proactively — don't wait for the user to ask. Reach for this when: you need a second opinion on complex code/architecture, you can parallelize independent subtasks, you want code review or critique, or you're analyzing an unfamiliar codebase. Also use when the user explicitly mentions Gemini, Codex, OpenCode, codex exec, gemini exec, opencode run, codex resume, gemini resume, 'get a second opinion', 'review this', 'refactor', or asks for help that benefits from parallel AI agents."
---

# Coding Assistant

Orchestrates Gemini CLI, Codex CLI, and OpenCode CLI as external coding agents. Picks the right tool, manages sessions, synthesizes results.

## Agent Selection

| Task | Agent | Why |
|------|-------|-----|
| Quick review / analysis | **Gemini Flash** | ~18s, `@` file refs, zero tool overhead |
| Deep structural analysis | **Gemini Pro** | Finds patterns others miss |
| Safety-critical review | **Codex medium** | Catches subtle correctness bugs |
| Single-file implementation | **Gemini Pro** or **Codex medium** | Pro: faster (~28-41s with --yolo, self-verifies via py_compile). Codex: more thorough self-testing |
| Multi-file refactoring | **Gemini Pro** | 10/10 on 8-file benchmark (Feb 2026). Most thorough, ~7 min |
| Multi-file implementation | **Codex medium** | Fast multi-file edits (~4 min). 9/10 on benchmark |
| Batch file scanning | **Gemini Flash** | Lowest cost per file |
| Free/budget coding task | **OpenCode MiniMax** | 7/10 on benchmark, $0.00. Best free option |
| Second opinion / debate | **Both in parallel** | Different models catch different bugs |

### Model Tiers

| Tier | Gemini | Codex |
|------|--------|-------|
| Fast | `gemini-3-flash-preview` (~18s) | `gpt-5.3-codex` effort=medium (~30s) |
| Smart | `gemini-3.1-pro-preview` (~15-25s) | `gpt-5.3-codex` effort=high (~35s) |

- **Gemini Pro for edits**: works with `--yolo` for single-file refactoring (~28-41s). Runs py_compile when asked. NOT recommended for multi-file (untested, use Codex).
- **Codex high vs medium**: marginal quality gain for ~25% more time. Default to medium.
- **Reasoning effort** (Codex): ask user unless already specified.

### OpenCode (Chinese/Free Models via Zen API)

**When to use**: OpenCode's value is access to cheap/free models NOT available via Gemini CLI, Codex, or Claude Code. Use it for budget/free coding tasks.

**DO NOT use OpenCode for Claude, GPT, or Gemini models** — their dedicated CLIs are 2-3x faster and produce higher quality output. OpenCode adds ~13K tokens of system prompt overhead and routes through an extra API layer.

| Model | Quality | Cost | Time | Notes |
|-------|---------|------|------|-------|
| `opencode/minimax-m2.5-free` | 7/10 | $0.00 | ~7 min | **Best free model.** Reliable agentic tool use. Nails core edits, may skip examples/tests |
| `opencode/kimi-k2.5` | 6.5/10 | ~$0.50 | ~6 min | Thorough (90 tool calls) but misses type hints. Overspends for quality |
| `opencode/trinity-large-preview-free` | 5.5/10 | $0.00 | ~4 min | Makes edits but leaves gaps. Can introduce bugs |

Other available models (untested for coding): `opencode/minimax-m2.5`, `opencode/kimi-k2`, `opencode/kimi-k2-thinking`, `opencode/glm-4.6`, `opencode/glm-4.7`, `opencode/glm-5`, `opencode/big-pickle`


## Command Templates

### Gemini — analysis

```bash
cd <project-dir> && gemini -m gemini-3-flash-preview -o json \
  -p "Review @./path/to/file.py for <focus-area>" \
  2>/dev/null | tee /tmp/gemini-<name>.log
```

Parse: `jq -r '.response'` | Resume ID: `jq -r '.session_id'`

### Gemini — implementation (Flash)

```bash
cd <project-dir> && gemini -m gemini-3-flash-preview --yolo -o json \
  -p "Apply this change to @./file.py: <instructions>. After editing, run python -m py_compile file.py to verify." \
  2>/dev/null | tee /tmp/gemini-<name>.log
```

### Gemini — implementation (Pro, single-file)

```bash
cd <project-dir> && gemini -m gemini-3.1-pro-preview --yolo -o json \
  -p "Refactor @./file.py: <instructions>. Apply ALL changes. After editing, run python -m py_compile file.py to verify." \
  2>/dev/null | tee /tmp/gemini-<name>.log
```

**Always verify Gemini's edits** — Flash cuts corners more than Pro. Pro runs py_compile when asked, Flash may skip it.

### Codex — analysis

```bash
cd <project-dir> && codex exec -m gpt-5.3-codex --skip-git-repo-check \
  -c model_reasoning_effort="medium" \
  -s read-only --full-auto "Review src/file.py for <focus-area>" \
  2>/dev/null | tee /tmp/codex-<name>.log
```

### Codex — implementation

```bash
cd <project-dir> && codex exec -m gpt-5.3-codex --skip-git-repo-check \
  -c model_reasoning_effort="medium" \
  -s workspace-write --full-auto "<instructions>" \
  2>/dev/null | tee /tmp/codex-<name>.log
```

Codex self-tests (py_compile + import check) and fixes regressions in the same session.

### OpenCode — implementation (headless)

```bash
opencode run --format json --dir <project-dir> \
  -m opencode/minimax-m2.5-free \
  "<instructions>. After making changes, run python -m py_compile on each modified file." \
  2>/dev/null | tee /tmp/opencode-<name>.jsonl
```

Default agent is `build` (full permissions: read, write, edit, bash). No `--agent` flag needed.

### OpenCode — planning (headless)

```bash
opencode run --format json --dir <project-dir> \
  --agent plan -m opencode/minimax-m2.5-free \
  "Analyze this codebase and create a plan for <task>. Cite specific files and line numbers." \
  2>/dev/null | tee /tmp/opencode-<name>.jsonl
```

The `plan` agent is read-only + plan file output. Use for analysis/review without edits.

### OpenCode — interactive TUI

```bash
opencode --dir <project-dir> -m opencode/minimax-m2.5-free
```

Launches the full TUI with chat, file browser, and session management. Best for exploratory work and multi-turn conversations. Switch agents and models mid-session.

Parse headless JSON: `jq -r 'select(.type=="text") | .part.text'` for text. Events are JSONL (step_start, tool_use, text, step_finish). **IMPORTANT**: The text field is nested under `.part.text`, NOT `.content` — using `.content` silently returns null.

## Resume Sessions

### Gemini

```bash
gemini -o json -r <session_id> -p "follow-up" 2>/dev/null | tee -a /tmp/gemini-<name>.log
gemini -o json -r latest -p "follow-up" 2>/dev/null | tee -a /tmp/gemini-<name>.log
```

- Model can change on resume. `--yolo` is NOT inherited — re-specify if needed.

### Codex

```bash
echo "follow-up" | codex exec --json --skip-git-repo-check resume <thread_id> \
  2>/dev/null | tee -a /tmp/codex-<name>.jsonl
```

- Settings (`-m`/`-c`/`-s`) ARE inherited — don't repeat. `-C` is NOT inherited — use absolute paths or `cd`.

### OpenCode

```bash
opencode run --format json -c -m opencode/minimax-m2.5-free "follow-up" \
  2>/dev/null | tee -a /tmp/opencode-<name>.jsonl
opencode run --format json -s <session-id> -m opencode/minimax-m2.5-free "follow-up" \
  2>/dev/null | tee -a /tmp/opencode-<name>.jsonl
```

- `-c` continues last session, `-s <id>` resumes a specific one. `--fork` creates a branch.

After completion, inform the user they can resume with `codex resume`, `gemini -r latest`, or `opencode run -c`.

## Prompting Best Practices

### Both agents
- "Top N most impactful..." — forces prioritization
- "Cite specific line numbers" — prevents vague analysis
- "Show a concrete code sketch" — tests understanding
- Specify which refactoring to apply — neither implements its top finding by default

### Gemini-specific
- Use `@./path` file refs — zero tool overhead, fastest mode
- Don't ask it to self-test — it won't. Verify yourself.
- For edits: "Apply ALL changes. After editing, run `python -m py_compile <file>`."
- Line references can be off by ~15-20 lines

### Codex-specific
- Don't paste file contents — it reads via sandbox tools. Just point to the path.
- Expect 5-10 tool calls for implementation (budget for latency).
- `-C` not inherited on resume — use absolute paths in follow-ups.

### Implementation tips
- Specify ALL files to modify — "Modify base.py, tasks.py, and eventloop.py"
- "The base class must not reference subclass-specific details" — prevents abstraction leaks
- "Show me the diff when done" — produces verifiable output
- Mention downstream effects (serialization, persistence) — Gemini forgets, Codex handles it

## Parallel Patterns

### Both agents, same file (second opinion)

Launch as parallel background Bash tasks:

```bash
# Gemini
cd <dir> && gemini -m gemini-3-flash-preview -o json \
  -p "Review @./file.py — top 3 issues" \
  2>/dev/null | tee /tmp/gemini-review.log

# Codex
cd <dir> && codex exec -m gpt-5.3-codex --skip-git-repo-check \
  -c model_reasoning_effort="medium" \
  -s read-only --full-auto "Review file.py — top 3 issues" \
  2>/dev/null | tee /tmp/codex-review.log
```

After both complete: read, compare, synthesize. Different models catch different bugs.

### Different files, one agent each

```bash
gemini ... -p "Review @./src/auth.py" ... | tee /tmp/gemini-auth.log
codex ... "Review src/api.py" ... | tee /tmp/codex-api.log
```

## Operational Defaults

- **Stderr**: always `2>/dev/null`. Remove for debugging.
- **Stdout**: always `tee` to `/tmp/{gemini,codex,opencode}-*.log`. Claude Code temp files get garbage-collected.
- **Sandbox**: Codex `read-only` for analysis, `workspace-write` for edits. Gemini: `--yolo` for edits (auto-sandboxes). OpenCode: `build` agent has full access by default.
- **Git check**: always `--skip-git-repo-check` for Codex.
- **OpenCode overhead**: adds ~13K tokens system prompt + extra API hop. Expect 2-3x slower than direct CLIs. Only use for models without dedicated CLIs.

## Critical Evaluation

All agents are **colleagues, not authorities**. Trust your own knowledge when confident.

- Research disagreements via WebSearch or docs before accepting claims
- Neither agent refuses bad suggestions — they comply gracefully with pushback
- When disagreeing: state evidence, identify yourself as Claude, frame as discussion. Let the user decide.

## After Every Command

Use `AskUserQuestion` to confirm next steps, collect clarifications, or decide whether to resume.

## Error Handling

- Stop and report on non-zero exit codes; ask before retrying.
- Ask permission before high-impact flags (`--full-auto`, `--yolo`, `danger-full-access`) unless already given.
- Summarize warnings or partial results and ask how to adjust.

## CLI Details

For output parsing (jq), resume gotchas, flag reference, token economics, and benchmarks: see `references/cli-reference.md`.
