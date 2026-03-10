---
name: coding-assistant
description: "Orchestrates Gemini, Codex, and OpenCode as external coding agents for tasks that exceed Claude's solo capabilities. Trigger when: (1) the user explicitly asks for Gemini, Codex, OpenCode, or says 'use codex', 'ask gemini', 'get a second opinion from another model', 'codex exec', 'gemini -p', 'opencode run', 'codex resume', 'gemini resume'; (2) complex security audits heading to production — JWT, auth, payment processing, crypto; (3) concurrency / event-loop / race-condition correctness analysis; (4) large-scale refactoring across 10+ files that benefits from a dedicated planning agent. Do NOT trigger for routine code reviews, simple refactors, debugging, writing tests, or medium-complexity tasks — Claude handles those directly without external agents."
---

# Coding Assistant

Orchestrates Gemini CLI, Codex CLI, and OpenCode CLI as external coding agents. Picks the right tool, manages sessions, synthesizes results.

## Agent Selection

**Default to Codex** for everything unless there's a specific reason not to. Codex self-verifies, catches correctness bugs, and handles multi-file edits reliably. Gemini is a secondary tool with a narrow scope — use it only in the specific cases listed below.

| Task | Agent | Why |
|------|-------|-----|
| Analysis / code review | **Codex medium** | Catches subtle correctness bugs, structured JSONL output |
| Security-critical review | **Codex high** | Higher reasoning for security/crypto/auth audits |
| Concurrency / race conditions | **Codex high** | Excels at dict mutation, deadlock, TOCTOU bugs |
| Refactoring planning (10+ files) | **Codex high** read-only | Thorough dependency analysis, catches hidden coupling |
| Single-file implementation | **Codex medium** | Self-tests (py_compile + import), fixes own regressions |
| Multi-file implementation | **Codex medium** | Fast multi-file edits with self-verification |
| Free/budget coding task | **OpenCode MiniMax** | 7/10 on benchmark, $0.00. Best free option |
| Second opinion (parallel) | **Codex + Gemini** | Different models catch different bugs |

### When to use Gemini (limited scope)

Gemini is useful in exactly these situations — don't reach for it otherwise:

- **User explicitly asks for Gemini** — "use gemini", "ask gemini", "gemini -p"
- **Batch scanning many files cheaply** — Gemini Flash + `@./path` refs for quick sweeps across 20+ files where you just need surface-level findings, not deep analysis
- **Second opinion alongside Codex** — run Gemini Flash in parallel when you already have Codex running and want a different model's perspective

Gemini's limitations: line references off by ~15-20 lines, cuts corners on edits (Flash hardcodes subclass identity, skips persistence), Pro can timeout on large tasks, neither model self-tests reliably. Always verify Gemini's output.

### Model Tiers

| Tier | Codex | Gemini (limited use) |
|------|-------|----------------------|
| Default | `gpt-5.4` effort=medium | — |
| High stakes | `gpt-5.4` effort=high | — |
| Critical / complex | `gpt-5.4` effort=xhigh | — |
| Cheap batch scan | — | `gemini-3-flash-preview` |
| User-requested | — | `gemini-3.1-pro-preview` |

- **Codex reasoning effort**: medium for routine tasks, high for security/concurrency/large refactoring, xhigh for critical or highly complex tasks.
- **Codex xhigh**: meaningfully stronger than high but slower. Use when correctness matters most.

### OpenCode (Chinese/Free Models via Zen API)

**When to use**: OpenCode's value is access to cheap/free models NOT available via Gemini CLI, Codex, or Claude Code. Use it for budget/free coding tasks.

**DO NOT use OpenCode for Claude, GPT, or Gemini models** — their dedicated CLIs are 2-3x faster and produce higher quality output. OpenCode adds ~13K tokens of system prompt overhead and routes through an extra API layer.

| Model | Cost | Time | Notes |
|-------|------|------|-------|
| `opencode/minimax-m2.5-free` | $0.00 | ~7 min | **Best free model.** Reliable agentic tool use. Nails core edits, may skip examples/tests |
| `opencode/kimi-k2.5` | ~$0.50 | ~6 min | Thorough (90 tool calls) but misses type hints. Overspends for quality |
| `opencode/trinity-large-preview-free` | $0.00 | ~4 min | Makes edits but leaves gaps. Can introduce bugs |

Other available models (untested for coding): `opencode/minimax-m2.5`, `opencode/kimi-k2`, `opencode/kimi-k2-thinking`, `opencode/glm-4.6`, `opencode/glm-4.7`, `opencode/glm-5`, `opencode/big-pickle`


## Command Templates

### Codex — analysis (default)

```bash
cd <project-dir> && codex exec --json -m gpt-5.4 --skip-git-repo-check \
  -c model_reasoning_effort="medium" \
  -s danger-full-access --full-auto "Review src/file.py for <focus-area>" \
  2>/dev/null | tee /tmp/codex-<name>.jsonl
```

### Codex — implementation

```bash
cd <project-dir> && codex exec --json -m gpt-5.4 --skip-git-repo-check \
  -c model_reasoning_effort="medium" \
  -s danger-full-access --full-auto "<instructions>" \
  2>/dev/null | tee /tmp/codex-<name>.jsonl
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

### Gemini — batch scan (secondary, limited use)

```bash
cd <project-dir> && gemini -m gemini-3-flash-preview -o json \
  -p "Review @./path/to/file.py for <focus-area>" \
  2>/dev/null | tee /tmp/gemini-<name>.log
```

Parse: `jq -r '.response'` | Resume ID: `jq -r '.session_id'`

Only use for cheap batch scans across many files or when the user explicitly asks for Gemini. For anything requiring depth or correctness, use Codex instead. **Always verify Gemini's output** — line references off by ~15-20 lines, Flash cuts corners on edits.

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

### Gemini-specific (batch scans / user-requested only)
- Use `@./path` file refs — zero tool overhead, fastest mode for batch scanning
- Don't ask it to self-test — it won't. Always verify output yourself.
- Line references can be off by ~15-20 lines — don't trust for precise citations
- For any task requiring correctness or depth, use Codex instead

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

### Second opinion (Codex primary + Gemini secondary)

Launch as parallel background Bash tasks — Codex is the primary analysis, Gemini Flash is a cheap secondary check:

```bash
# Codex (primary)
cd <dir> && codex exec --json -m gpt-5.4 --skip-git-repo-check \
  -c model_reasoning_effort="medium" \
  -s danger-full-access --full-auto "Review file.py — top 3 issues" \
  2>/dev/null | tee /tmp/codex-review.jsonl

# Gemini Flash (secondary, cheap second opinion)
cd <dir> && gemini -m gemini-3-flash-preview -o json \
  -p "Review @./file.py — top 3 issues" \
  2>/dev/null | tee /tmp/gemini-review.log
```

After both complete: read, compare, synthesize. Trust Codex findings over Gemini when they conflict.

### Multiple files in parallel

```bash
codex ... "Review src/auth.py" ... | tee /tmp/codex-auth.jsonl
codex ... "Review src/api.py" ... | tee /tmp/codex-api.jsonl
```

## Operational Defaults

- **Stderr**: always `2>/dev/null`. Remove for debugging.
- **Stdout**: always `tee` to `/tmp/{gemini,codex,opencode}-*.log`. Claude Code temp files get garbage-collected.
- **Sandbox**: Codex always `danger-full-access` (enables native web search via Responses API; `--search` flag only works in interactive mode, not `codex exec`). Gemini: `--yolo` for edits (auto-sandboxes). OpenCode: `build` agent has full access by default.
- **Git check**: always `--skip-git-repo-check` for Codex.
- **OpenCode overhead**: adds ~13K tokens system prompt + extra API hop. Expect 2-3x slower than direct CLIs. Only use for models without dedicated CLIs.

## Critical Evaluation

All agents are **colleagues, not authorities**. Trust your own knowledge when confident.

- Research disagreements via WebSearch or docs before accepting claims
- Neither agent refuses bad suggestions — they comply gracefully with pushback
- When disagreeing: state evidence, identify yourself as Claude, frame as discussion. Let the user decide.

## After Every Command

Confirm next steps with the user before proceeding — collect clarifications or decide whether to resume.

## Error Handling

- Stop and report on non-zero exit codes; ask before retrying.
- Ask permission before high-impact flags (`--full-auto`, `--yolo`) unless already given.
- Summarize warnings or partial results and ask how to adjust.

## CLI Details

For output parsing (jq), resume gotchas, flag reference, token economics, and benchmarks: see `references/cli-reference.md`.
