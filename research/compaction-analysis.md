# Compaction Analysis

Why compaction barely moves the dial and how the death spiral works.

## What Compaction Does

When a session approaches the context window limit, OpenClaw:

1. Runs a **memory flush** — asks the agent to save durable state to disk files
2. **Summarises** the conversation history into a compact block
3. **Keeps recent messages** (last few exchanges) intact
4. **Replaces** everything else with the summary
5. The session continues with: summary + recent messages + new exchanges

## The Promise vs Reality

**What users expect:**
```
Before:  ████████████████████░░░░  85% full
After:   ████░░░░░░░░░░░░░░░░░░░  20% full — loads of room!
```

**What actually happens:**
```
Before:  ████████████████████░░░░  85% full
After:   █████████████░░░░░░░░░░░  65% full — bought maybe 20 minutes
```

## Why? The Numbers

Take a session at 170k/200k (85%):

| Component | Before compaction | After compaction |
|-----------|------------------|-----------------|
| System prompt + files + skills + tools | 15,000 | 15,000 (unchanged) |
| Old conversation | 100,000 | 0 (replaced by summary) |
| Compaction summary | 0 | 15,000-25,000 (new) |
| Recent messages | 55,000 | 10,000-20,000 (trimmed to recent) |
| Memory flush exchange | 0 | 3,000-5,000 (new) |
| **Total** | **170,000** | **43,000-65,000** |

So you drop from 170k to maybe 50-65k. That's 105-127k freed. Sounds good?

**The problem is what happens next.** Your "recent messages" might include the last big tool output (10-20k tokens). The summary itself is 15-25k. The memory flush added another 3-5k. You've freed space, but 25-35% of the window is already consumed by compaction artefacts.

## The Death Spiral

Each compaction compounds:

### Cycle 1
```
Session start:  15k baseline
Work 2 hours:   170k total
Compact:         50k (summary: 15k)
Resume work:     50k → 85k → 120k → 170k
```

### Cycle 2
```
Compact again:   55k (summaries: 25k — old summary + new summary)
Resume work:     55k → 90k → 130k → 170k
```

### Cycle 3
```
Compact again:   65k (summaries: 40k — three layers of summaries)
Resume work:     65k → 100k → 140k → 170k ← fills up faster
```

Each cycle:
- The summary grows (cumulative)
- The usable window shrinks
- Time between compactions decreases
- Each summary is a lossy compression of the previous summary — detail degrades

By cycle 4-5, summaries can consume 50-60k tokens. The usable window is down to maybe 100k, and with the fixed baseline, you really have 85k for actual work. That fills in 20-30 minutes of heavy tool use.

## What Gets Lost

Compaction summaries are **lossy by nature**. What survives:

- ✅ Major decisions and outcomes
- ✅ File paths and URLs mentioned
- ✅ Task completion status
- ✅ Key facts and numbers

What gets lost:

- ❌ Reasoning chains ("I tried X because Y, which showed Z, so I pivoted to W")
- ❌ Debugging context ("the error was on line 47, caused by a race condition in...")
- ❌ Conversational nuance ("you seemed hesitant about this approach")
- ❌ Half-finished investigations ("I was about to check whether...")
- ❌ The mental model of the problem space
- ❌ Working hypotheses that hadn't been confirmed yet

The agent after compaction is like someone who read the meeting notes but wasn't in the meeting. They know *what* was decided but not *why*, not the alternatives considered, not the subtleties.

## OpenClaw's Compaction Modes

### `default`
Standard summarisation. Single pass. Works for short-medium sessions.

### `safeguard`
Chunked summarisation for long histories. Breaks the history into chunks, summarises each, then combines. Better for very long sessions (our current setting).

### Configuration

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard",
        reserveTokensFloor: 20000,   // keep at least this many tokens free
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 10000  // trigger flush when this close to limit
        }
      }
    }
  }
}
```

`reserveTokensFloor: 20000` means compaction triggers when only 20k tokens remain free. On a 200k window, that's at 180k (90%). This is **very late**. By the time compaction fires, the summary has to compress a massive conversation.

## What Would Actually Help

### Immediate (config changes)
- Increase `reserveTokensFloor` to 50-60k — compact earlier when there's less to summarise
- Use `/compact` manually at 50% with targeted instructions
- Enable more aggressive pruning of tool results

### Structural (discipline changes)
- Move all heavy tool work to sub-agents
- Cap main session tool calls — if you need >5 tool calls for a task, spawn it
- Proactive context awareness — agent warns at 50%, suggests action

### Fundamental (needs core changes)
- Compaction that produces structured output (not narrative summary)
- `/new` with handoff document (fresh session, structured state transfer)
- Non-cumulative compaction — replace the old summary instead of stacking
- Token-aware tool calls — show estimated cost before executing
- Selective compaction — compact tool results but keep conversation
