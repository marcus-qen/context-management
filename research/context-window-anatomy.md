# Context Window Anatomy

What actually fills a 200k token context window in a long-running OpenClaw session.

## The Layers

A context window isn't one thing — it's five layers stacked together, and only one of them is your actual conversation.

```
┌─────────────────────────────────────────────────────────┐
│ LAYER 1: SYSTEM PROMPT                                  │
│ OpenClaw instructions, safety rules, tool policies,     │
│ runtime info, reply tags, messaging rules               │
│ ~3,000-5,000 tokens (FIXED, every turn)                 │
├─────────────────────────────────────────────────────────┤
│ LAYER 2: WORKSPACE FILES (auto-loaded)                  │
│ AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md,    │
│ HEARTBEAT.md, MEMORY.md — loaded into system prompt     │
│ ~2,000-4,000 tokens (FIXED, depends on file sizes)      │
├─────────────────────────────────────────────────────────┤
│ LAYER 3: SKILL & TOOL DEFINITIONS                       │
│ Descriptions of all available skills (~64 in our setup) │
│ + JSON schemas for all available tools (~20+)           │
│ ~4,000-7,000 tokens (FIXED)                             │
├─────────────────────────────────────────────────────────┤
│ LAYER 4: CONVERSATION HISTORY                           │
│ Your messages + agent replies + compaction summaries     │
│ ~20,000-80,000 tokens (GROWS with conversation)         │
├─────────────────────────────────────────────────────────┤
│ LAYER 5: TOOL CALL RESULTS                              │
│ Every exec output, file read, web fetch, search result  │
│ ~30,000-120,000 tokens (GROWS, often the biggest layer) │
└─────────────────────────────────────────────────────────┘
```

## Real Measurements (Our Workspace)

Measured 2026-02-19 from a production OpenClaw deployment running Claude Opus (200k window):

### Layer 1: System Prompt
OpenClaw's own instructions — tool policies, reply formatting, session management rules, heartbeat handling, group chat context, etc.

**Estimated: ~4,000-5,000 tokens**

Cannot be reduced by users. This is OpenClaw's operating system.

### Layer 2: Workspace Files

| File | Characters | Est. Tokens |
|------|-----------|-------------|
| AGENTS.md | 3,550 | ~887 |
| SOUL.md | 1,356 | ~339 |
| USER.md | 1,753 | ~438 |
| TOOLS.md | 860 | ~215 |
| IDENTITY.md | 636 | ~159 |
| HEARTBEAT.md | 168 | ~42 |
| MEMORY.md | 2,996 | ~749 |
| **Total** | **11,319** | **~2,829** |

**Key insight:** MEMORY.md is the only file here with significant variance. When it was 224 lines (pre-workspace-standard), it cost ~1,800 tokens. At 73 lines, it costs ~749. That's a 1,000 token saving per session. Small, but compound it across hundreds of sessions.

### Layer 3: Skill & Tool Definitions

- 12 local skills (in workspace `skills/` directory)
- 52 built-in skills (in OpenClaw installation)
- Each skill's `description` field: ~50-150 tokens
- ~20+ tool definitions (JSON schemas): ~100-200 tokens each

**Estimated: ~5,000-7,000 tokens**

This is a hidden cost. 64 skills × ~80 tokens average = ~5,000 tokens just for skill descriptions. Users with fewer skills pay less here.

### Layer 4: Conversation History

This is what most people think of as "the context." It's usually not the biggest layer.

In a 2-hour working session:
- User messages: ~5,000-10,000 tokens (you type less than you think)
- Agent replies: ~30,000-50,000 tokens (the agent is verbose)
- Compaction summaries (if any): ~10,000-20,000 tokens each

**Typical range: 40,000-80,000 tokens**

### Layer 5: Tool Call Results

This is the silent killer. Every `exec` command, every file read, every web search — the full output lives in context.

Examples from our 2026-02-19 session:
- 10 test suite runs: ~25,000-30,000 tokens
- 4 competitor skill reads: ~10,000-15,000 tokens
- Full file walkthroughs: ~10,000 tokens
- Various exec commands: ~20,000-30,000 tokens

**Typical range: 60,000-120,000 tokens in a heavy working session**

**This layer is almost always the largest.** Tool results accumulate because:
1. They're full outputs — a test run that prints 200 lines is 200 lines in context
2. They're rarely pruned during active conversation (pruning TTL doesn't expire)
3. They're kept on compaction if they're "recent"

## The Fixed Floor

Before you say a single word, the context window is already:

```
System prompt:           ~5,000 tokens
Workspace files:         ~3,000 tokens
Skill descriptions:      ~5,000 tokens
Tool definitions:        ~2,000 tokens
─────────────────────────────────────
FLOOR:                   ~15,000 tokens (7.5% of 200k)
```

This floor is **irreducible without removing skills or shrinking files.** It's the cost of having a capable agent.

After compaction, the floor rises:

```
Fixed floor:             ~15,000 tokens
Compaction summary:      ~15,000 tokens
Recent messages:         ~10,000 tokens
─────────────────────────────────────
POST-COMPACTION FLOOR:   ~40,000 tokens (20% of 200k)
```

After second compaction:

```
Fixed floor:             ~15,000 tokens
Cumulative summaries:    ~25,000 tokens  ← grew
Recent messages:         ~10,000 tokens
─────────────────────────────────────
FLOOR:                   ~50,000 tokens (25% of 200k)
```

Each compaction cycle raises the floor. This is the death spiral.

## Implications

1. **A 200k window gives you ~185k of usable space** (after fixed overhead). That sounds like a lot, but heavy tool use can burn it in 30-60 minutes.

2. **Tool outputs are the #1 optimisation target.** Reducing tool output size or moving tool-heavy work to sub-agents has the biggest impact.

3. **MEMORY.md budget matters but isn't transformative.** Saving 1,000 tokens on MEMORY.md is worthwhile but won't prevent compaction death spirals.

4. **Skill count has a real cost.** 64 skills × ~80 tokens = ~5,000 tokens. If you're not using most of them, that's wasted baseline.

5. **The agent's own verbosity is a factor.** Longer replies mean more context consumed. Terse, information-dense replies cost less. But users generally want thoughtful responses, not telegrams.
