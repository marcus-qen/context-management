# OpenClaw Context Configuration Reference

All configuration settings that affect context window management.

## Compaction

```json5
{
  agents: {
    defaults: {
      compaction: {
        // "default" — single-pass summarisation
        // "safeguard" — chunked summarisation for long histories (recommended)
        mode: "safeguard",

        // Trigger compaction when fewer than this many tokens remain free
        // Higher = compact earlier = smaller summaries = more headroom after
        // Default: 24000. We use: 20000. Recommendation: 40000-60000.
        reserveTokensFloor: 20000,

        memoryFlush: {
          // Run a silent agent turn before compaction to save state to disk
          enabled: true,
          // Trigger memory flush when this many tokens from the limit
          softThresholdTokens: 10000,
          // System prompt for the flush turn
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          // User prompt for the flush turn
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

### Key Tuning Points

**`reserveTokensFloor`**: The most impactful single setting. Default 24000 means compaction fires at 88% (176k/200k). Setting it to 50000 means compaction fires at 75% (150k/200k) — earlier, with less history to summarise, producing a smaller summary.

**`mode: "safeguard"`**: Recommended for long sessions. Chunks the history before summarising, which produces better summaries for very long conversations.

**`memoryFlush.enabled`**: Critical for not losing state. The flush turn asks the agent to write important context to disk before compaction destroys it.

## Session Pruning

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        // "off" — no pruning
        // "cache-ttl" — prune when cache TTL expires
        mode: "cache-ttl",

        // How long since last API call before pruning fires
        // Lower = more aggressive. "1m" catches brief pauses.
        // But won't fire during continuous conversation regardless.
        ttl: "5m",

        // Protect tool results from this many recent assistant messages
        // Lower = more aggressive pruning
        keepLastAssistants: 2,

        // Soft-trim: truncate large tool results
        softTrimRatio: 0.3,     // Trim when result > 30% of prunable space
        minPrunableToolChars: 50000,  // Only trim results larger than this
        softTrim: {
          maxChars: 4000,       // Max size of trimmed output
          headChars: 1500,      // Keep this much from the start
          tailChars: 1500       // Keep this much from the end
        },

        // Hard-clear: replace old tool results entirely
        hardClearRatio: 0.5,    // Clear when result > 50% of prunable space
        hardClear: {
          enabled: true,
          placeholder: "[Old tool result content cleared]"
        },

        // Tool-specific filtering
        tools: {
          allow: [],            // Empty = all tools eligible for pruning
          deny: ["browser", "canvas"]  // Never prune these tools' results
        }
      }
    }
  }
}
```

### Key Tuning Points

**`keepLastAssistants`**: Controls how many recent exchanges are protected. Default 3, we use 2. Setting to 1 is most aggressive — only the last exchange is safe.

**`minPrunableToolChars`**: Default 50000 (50k chars ≈ 12.5k tokens). Only targets very large results. Lowering to 10000 catches medium results too.

**`tools.deny`**: Results from denied tools are never pruned. Useful for tools whose output you'll reference later (browser snapshots, images).

## Context Window

```json5
{
  agents: {
    defaults: {
      // Cap the resolved context window (optional)
      // Useful if you want to limit usage below the model's maximum
      contextTokens: 200000,

      models: {
        "anthropic/claude-opus-4-6": {
          params: {
            // Anthropic's extended 1M context (if available)
            context1m: false
          }
        }
      }
    }
  }
}
```

## Recommended Configurations

### Conservative (current: minimal intervention)
```json5
{
  compaction: { mode: "safeguard", reserveTokensFloor: 20000 },
  contextPruning: { mode: "cache-ttl", ttl: "5m", keepLastAssistants: 2 }
}
```
Pros: Least interference. Cons: Late compaction, large summaries, death spiral.

### Balanced (recommended)
```json5
{
  compaction: { mode: "safeguard", reserveTokensFloor: 50000 },
  contextPruning: {
    mode: "cache-ttl",
    ttl: "2m",
    keepLastAssistants: 1,
    minPrunableToolChars: 10000
  }
}
```
Pros: Earlier compaction, more aggressive pruning. Cons: Might lose some recently-referenced tool context.

### Aggressive (heavy tool users)
```json5
{
  compaction: { mode: "safeguard", reserveTokensFloor: 60000 },
  contextPruning: {
    mode: "cache-ttl",
    ttl: "1m",
    keepLastAssistants: 1,
    minPrunableToolChars: 5000,
    softTrim: { maxChars: 2000, headChars: 800, tailChars: 800 }
  }
}
```
Pros: Maximum headroom, smallest summaries. Cons: Aggressive tool result pruning, may need to re-read files.
