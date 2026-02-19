# Pruning Mechanics

How OpenClaw's session pruning works, why it often doesn't fire, and what it can fix.

## What Pruning Is

Session pruning trims **old tool results** from the in-memory context before each LLM call. It does NOT modify the session history on disk — it's a transient, per-request optimisation.

Pruning is separate from compaction:
- **Pruning:** Trims tool outputs, transient, per-request
- **Compaction:** Summarises conversation, persistent, rewrites history

## How It Works

### cache-ttl Mode (our current config)

```json5
{
  contextPruning: {
    mode: "cache-ttl",
    ttl: "5m",
    keepLastAssistants: 2
  }
}
```

1. **TTL gate:** Pruning only runs if the last Anthropic API call was more than `ttl` ago (default 5 minutes). This is designed to work with Anthropic's prompt caching — don't prune while the cache is warm.

2. **Recent protection:** Tool results after the last `keepLastAssistants` assistant messages are never pruned. With `keepLastAssistants: 2`, the tool results from the last 2 exchanges are safe.

3. **Two-stage trim:**
   - **Soft-trim:** Large tool results (>50k chars) are truncated — keep head + tail, insert `...`
   - **Hard-clear:** Older tool results are replaced with `"[Old tool result content cleared]"`

### Why It Often Doesn't Fire

**During active conversation, the TTL never expires.** If you're exchanging messages every 1-3 minutes, and the TTL is 5 minutes, pruning never triggers. Tool results accumulate indefinitely during an active session.

This is by design — Anthropic's prompt cache has a TTL, and pruning within that window would invalidate the cache, causing a more expensive re-cache on the next call. Pruning is meant for idle→active transitions, not continuous use.

**But this means:** During the exact scenario where context pressure is highest (active, heavy tool use), pruning does nothing.

## Current Configuration (Our Setup)

```json5
{
  contextPruning: {
    mode: "cache-ttl",
    ttl: "5m",
    keepLastAssistants: 2,
    // Using defaults for the rest:
    softTrimRatio: 0.3,
    hardClearRatio: 0.5,
    minPrunableToolChars: 50000,
    softTrim: { maxChars: 4000, headChars: 1500, tailChars: 1500 },
    hardClear: { enabled: true, placeholder: "[Old tool result content cleared]" }
  }
}
```

### What Each Setting Does

| Setting | Default | Effect |
|---------|---------|--------|
| `mode` | `"off"` | `"cache-ttl"` enables pruning |
| `ttl` | `"5m"` | How long since last API call before pruning fires |
| `keepLastAssistants` | `3` (ours: `2`) | Protect tool results from this many recent exchanges |
| `softTrimRatio` | `0.3` | Soft-trim when tool result > 30% of prunable space |
| `hardClearRatio` | `0.5` | Hard-clear when tool result > 50% of prunable space |
| `minPrunableToolChars` | `50000` | Only soft-trim results larger than this |
| `softTrim.maxChars` | `4000` | Max chars in soft-trimmed output |
| `softTrim.headChars` | `1500` | Keep this many chars from the start |
| `softTrim.tailChars` | `1500` | Keep this many chars from the end |
| `hardClear.enabled` | `true` | Whether to hard-clear old results |
| `hardClear.placeholder` | `"[Old tool result content cleared]"` | Replacement text |
| `tools.deny` | `[]` | Never prune these tools' results |

### What's NOT Pruned

- User messages — never touched
- Assistant messages — never touched
- Image blocks in tool results — skipped
- Tool results from the last `keepLastAssistants` exchanges

## Could More Aggressive Pruning Help?

### Option 1: Lower TTL
Set `ttl: "1m"` — prune after just 1 minute of inactivity. This would catch brief pauses in conversation. But during continuous rapid exchange, it still wouldn't fire.

### Option 2: Lower keepLastAssistants
Set `keepLastAssistants: 1` — only protect the very last exchange. Everything older gets pruned when TTL fires. More aggressive but risks losing context the agent just referenced.

### Option 3: Smaller softTrim thresholds
Set `minPrunableToolChars: 10000` — soft-trim results above 10k chars instead of 50k. Catches more medium-sized tool outputs.

### Option 4: Tool-specific deny lists
Some tool results are worth keeping (file reads you'll reference again). Others are disposable (test output, git status). Configure `tools.allow` to only prune exec results.

## The Fundamental Limitation

Pruning is a **reactive** measure. It can't prevent context growth during active use — it only cleans up during idle periods. For continuous working sessions (which is exactly when context pressure is highest), pruning is essentially disabled.

The real solution for active sessions is **prevention**: move tool-heavy work to sub-agents so the tool results never enter the main session's context in the first place.

## Pruning + Compaction Interaction

They're complementary but independent:

1. **Pruning fires first** (if TTL allows) — trims tool outputs
2. If still over limit, **compaction fires** — summarises the whole conversation
3. Pruning is transient (per-request), compaction is persistent (rewrites history)

In practice, if pruning hasn't been firing (active session), compaction handles everything. The compaction then summarises the full, un-pruned tool outputs — which is why compaction summaries are so large.
