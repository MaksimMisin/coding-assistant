# CLI Reference

## Gemini CLI

### Flags

| Flag | Purpose |
|------|---------|
| `-m <model>` | Model selection |
| `-p "prompt"` | One-shot prompt |
| `-o json` | JSON output: `{session_id, response, stats}` |
| `-o stream-json` | Streaming JSONL events |
| `-r <id>` / `-r latest` | Resume session |
| `--yolo` | Auto-approve all tools (sandbox auto-enabled) |
| `-s` | Manual sandbox |
| `--approval-mode plan` | Experimental (requires `experimental.plan` in settings) |

No `-C` flag — must `cd` to target directory before running.

### File References

Use `@./path/to/file` in prompts — reads from project context with zero tool overhead:

```bash
gemini -m gemini-3-flash-preview -o json \
  -p "Review @./src/main.ts and @./src/utils.ts for bugs" \
  2>/dev/null | tee /tmp/gemini-review.log
```

Stdin piping also works: `cat file.py | gemini -p "Explain this code" ...`

### Output Parsing

```bash
jq -r '.response'           # response text
jq -r '.session_id'          # session ID for resume
jq '.stats.models | to_entries[0].value.tokens'  # token usage
```

### Resume Gotchas

- Model flag can be changed on resume
- Context carries over (full conversation history)
- Working directory: uses cwd at invocation time
- `--yolo` is NOT inherited — re-specify if needed

## Codex CLI

### Flags

| Flag | Purpose |
|------|---------|
| `-m <model>` | Model selection |
| `exec "prompt"` | One-shot execution |
| `--json` | JSONL output |
| `resume <id>` / `resume --last` | Resume session |
| `-c model_reasoning_effort="<level>"` | `medium` or `high` |
| `-s <mode>` | `read-only`, `workspace-write`, `danger-full-access` |
| `--full-auto` | Auto-approve all tool calls |
| `-C <dir>` | Working directory (NOT inherited on resume) |
| `--skip-git-repo-check` | Always use this flag |

### JSONL Output Parsing

```bash
jq -r '
  if .type == "thread.started" then "Session: " + .thread_id
  elif .type == "item.completed" and .item.type == "agent_message" then .item.text
  elif .type == "item.completed" and .item.type == "command_execution" then "$ " + .item.command + "\n" + .item.aggregated_output
  elif .type == "item.completed" and .item.type == "reasoning" then empty
  elif .type == "turn.completed" then "---\nTokens: " + (.usage.input_tokens|tostring) + " in / " + (.usage.output_tokens|tostring) + " out (cached: " + (.usage.cached_input_tokens|tostring) + ")"
  else empty end'
```

### Resume Gotchas

- `-C` is NOT inherited — sessions fall back to invocation directory
- **Workaround**: use absolute paths in follow-ups, or `cd` to target directory
- Sandbox mode IS inherited (read-only stays read-only)
- Settings (`-m`/`-c`) are inherited — don't repeat unless changing
- Sandbox may restrict `/tmp` access on resume

## Comparison Table

| | Gemini CLI | Codex CLI |
|---|---|---|
| One-shot | `-p "prompt"` | `exec "prompt"` |
| Model flag | `-m model` | `-m model` |
| JSON output | `-o json` (single object) | `--json` (JSONL stream) |
| Resume | `-r <id>` / `-r latest` | `resume <id>` / `resume --last` |
| Working dir | Uses cwd (no `-C`) | `-C <dir>` (not inherited on resume) |
| Sandbox | `-s` / `--yolo` auto-sandboxes | `-s <mode>` |
| Full auto | `--yolo` | `--full-auto` |
| File refs | `@./path` in prompts | Reads from sandbox |
| Session ID | JSON `.session_id` | JSONL `thread.started` event |

## Token Economics

| Scenario | Input | Output | Notes |
|----------|-------|--------|-------|
| Gemini Flash analysis | ~13k | ~1k | Very cheap |
| Gemini Pro analysis | ~27k | ~1.6k | 2x Flash cost |
| Codex medium analysis | ~22k | ~2k | Moderate |
| Codex implementation | ~120k | ~6k | Expensive (self-verification re-reads) |
| Codex follow-up | ~150-365k | ~6-8k | Context grows fast, ~80-90% cached |

## Implementation Quality Benchmark

Tested on multi-file Python refactoring (3 files):

| Model | Compiles | Design | Self-tested | Files |
|-------|----------|--------|-------------|-------|
| Gemini Flash | Yes | B- (hardcoded subclass identity in base) | No | 3/3 |
| Gemini Pro | N/A | N/A (timed out, 0 output) | N/A | 0/3 |
| Codex medium | Yes | B+ (clean callbacks, keyword-only args) | Yes | 3/3 |
| Codex high | Yes | A- (lazy exception factories, better cancel) | Yes | 3/3 |
