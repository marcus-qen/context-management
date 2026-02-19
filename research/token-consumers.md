# Token Consumers

What actually burns tokens during a working session. Real data from production sessions.

## The 60/40 Rule

In a heavy working session (building, testing, debugging), the token split is roughly:

```
Tool outputs:     ~60% of consumed context
Conversation:     ~40% of consumed context
```

This surprised us. The intuition is that conversation dominates — after all, the agent writes long replies. But tool results are larger and more numerous.

## Tool Output Cost by Type

Measured across multiple sessions. These are **per-call** estimates:

| Tool | Typical output size | Tokens (est.) | Notes |
|------|-------------------|---------------|-------|
| `exec` (simple command) | 5-20 lines | 50-200 | `git status`, `ls`, `wc -l` |
| `exec` (test run) | 50-300 lines | 500-3,000 | Test suites, build output |
| `exec` (verbose command) | 100-1000 lines | 1,000-10,000 | `find`, logs, large grep |
| `Read` (small file) | 20-50 lines | 200-500 | Config files, short scripts |
| `Read` (medium file) | 100-300 lines | 1,000-3,000 | SKILL.md files, plans |
| `Read` (large file) | 500+ lines | 5,000+ | Full source files |
| `web_search` | 10 results | 500-1,000 | Titles + snippets |
| `web_fetch` | Variable | 2,000-10,000 | Depending on page content |
| `memory_search` | 5-10 results | 300-800 | Snippets with paths |
| `browser snapshot` | Variable | 1,000-5,000 | DOM snapshot |
| `session_status` | ~20 lines | 100-200 | Cheap |

### The Expensive Operations

These are the budget-busters:

1. **Test suites** — 10 test runs at 2-3k each = 25-30k tokens. That's 12-15% of a 200k window from testing alone.

2. **Full file reads** — Reading a 400-line file costs ~4,000 tokens. Reading 4 competitor skills = 16k tokens.

3. **Verbose exec output** — `bash -x script.sh` or any debug output can easily hit 5-10k per call.

4. **Web fetches** — A single web page can be 5-10k tokens after markdown conversion.

5. **Repeated reads** — Reading the same file multiple times (for editing, reviewing, verifying) multiplies the cost. Each read is a new tool result in context.

### The Cheap Operations

These are nearly free:

1. **`session_status`** — ~100-200 tokens
2. **`memory_search`** — ~300-800 tokens
3. **Simple exec** (`git status`, `wc -l`, `ls`) — ~50-200 tokens
4. **`Edit`** — ~100-300 tokens (just the old/new text)
5. **`Write`** — ~50-100 tokens (confirmation only)

## Conversation Cost

Agent replies vary enormously:

| Reply type | Tokens (est.) |
|-----------|---------------|
| Short answer | 50-200 |
| Explanation paragraph | 200-500 |
| Detailed analysis | 500-2,000 |
| Code with explanation | 1,000-3,000 |
| Long walkthrough (like this doc) | 3,000-8,000 |

User messages are typically 50-300 tokens (even long messages are short by token standards).

## Session Profile: Our 2026-02-19 Session

A real ~4 hour session building and testing the workspace-standard skill:

```
Estimated breakdown:
  System prompt + files:         ~15,000 tokens (9%)
  User messages (~30):           ~5,000 tokens (3%)
  Agent replies (~30):           ~40,000 tokens (24%)
  exec results (~50 calls):      ~60,000 tokens (37%)
  Read results (~15 calls):      ~25,000 tokens (15%)
  Other tools (~20 calls):       ~15,000 tokens (9%)
  Compaction summaries:          ~5,000 tokens (3%)
  ─────────────────────────────────────────────────
  Total:                         ~165,000 tokens (100%)
```

Tool results (exec + Read + other): ~100,000 tokens — **61% of total context**.

## Implications for Context Management

### 1. Reduce tool output, not conversation
The biggest win is making tool calls cheaper, not making the agent more terse. Options:
- Run heavy tools in sub-agents (their context is separate)
- Use `--quiet` flags on verbose commands
- Read specific line ranges instead of full files
- Pipe through `head`/`tail` instead of reading everything

### 2. Avoid redundant reads
Reading the same file 3 times costs 3× the tokens. Read once, work from memory, only re-read if unsure.

### 3. Batch operations
Instead of 10 separate `exec` calls, combine into one script. One result instead of ten.

### 4. The agent should estimate before acting
Before running a test suite that will produce 200 lines of output, the agent should consider: "Is this worth 2-3k tokens? Should this go in a sub-agent?"

### 5. Summary tool outputs
Instead of dumping raw output into context, the agent could pipe through `wc -l` first, then only read the relevant parts. But this requires discipline and adds latency.
