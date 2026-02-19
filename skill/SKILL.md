---
name: context-management
description: >
  Manage AI agent context window consumption, prevent compaction death spirals,
  and enforce sub-agent spawn policies. Use when: (1) context is filling up and
  work quality may degrade, (2) deciding whether to spawn a sub-agent or work in
  main session, (3) preparing for compaction or session handoff, (4) user asks
  "what's eating my context?" or "how much runway left?", (5) after compaction to
  restore working state from checkpoint files. NOT for: general memory/workspace
  management (use memory-keeper or workspace-standard).
---

# Context Management

Prevent context exhaustion, enforce spawn discipline, and make compaction survivable.

## Core Concepts

1. **Fixed baseline**: ~16k tokens (8%) consumed before any conversation — system prompt, workspace files, skill descriptions, tool definitions
2. **60/40 rule**: ~60% of context consumed by tool outputs, ~40% by conversation. Tool outputs are the primary target for savings.
3. **Compaction is lossy**: Summaries stack cumulatively. Each cycle raises the floor. After 3+ compactions, summaries alone can consume 30%+ of context.
4. **Sub-agents are disposable context**: A sub-agent can burn 150k tokens investigating something; only the summary (~500 tokens) enters main context.

## Procedures

### When Context Pressure Rises

After every tool-heavy operation (>5 tool calls), assess:

1. Run `session_status` to check usage
2. If **below 50%**: continue normally
3. If **50-70%**: spawn sub-agents for remaining tool-heavy work (>3 tool calls)
4. If **70-85%**: spawn sub-agents for ANY tool work (>1 tool call). Warn user.
5. If **above 85%**: write checkpoint (see below), suggest `/compact` or `/new`

### "What's Eating My Context?" — Estimation Method

Cannot get exact per-component breakdown. Estimate:

```
Fixed baseline:         ~16k tokens (8%)
Per user message:       ~100-500 tokens each
Per assistant response: ~200-1000 tokens each  
Per tool call result:   ~500-5000 tokens each (exec/read heavy, search light)
Compaction summaries:   ~2000-5000 tokens each (cumulative!)
```

Count messages and tool calls in recent history, multiply by midpoint estimates. Report as ranges, not false precision.

### Spawn Policy

Read `.context-policy.yml` from workspace root if it exists. Otherwise use defaults:

**Always spawn** (regardless of context level):
- Test suites (>3 tests)
- Multi-file audits (>5 files)  
- Build/deploy pipelines
- Research tasks (web search + analysis)
- Bulk file operations

**Never spawn** (keep in main session):
- Single commands
- Conversations / discussions
- Quick edits (1-3 files)
- Status checks
- Tasks requiring user input mid-execution

**Context-dependent** (spawn when context exceeds threshold):
- Above 50%: spawn if task involves >5 tool calls
- Above 70%: spawn if task involves >2 tool calls

When spawning, write detailed task descriptions. Sub-agents have no conversation context — they only know what the task field tells them.

### Pre-Compaction Checkpoint

Before compaction or `/new`, write `.context-checkpoint.md` in workspace root:

```markdown
# Context Checkpoint — {date} {time}

## Active Task
{what you were doing}

## Key State
{bullet list of current state — what's done, what's in progress}

## Decisions Made This Session
{numbered list of decisions with rationale}

## Files Changed
{list of files modified this session}

## Next Steps
{what to do after resuming}
```

This file survives compaction. On session start or post-compaction, check for it and use it to restore context. Delete after consuming.

### Post-Compaction Recovery

After compaction or `/new`:

1. Read `.context-checkpoint.md` if it exists
2. Read `memory/{today}.md` for recent log entries
3. Resume from the checkpoint's "Next Steps"
4. Delete the checkpoint file after restoring context

### Proactive Warning Template

When context exceeds 65%, warn:

```
⚠️ Context: {pct}% ({used}k/{total}k). Estimated runway: ~{remaining_calls} 
tool calls. {recommendation}
```

Recommendations by level:
- 65%: "Spawning sub-agents for remaining tool-heavy work."
- 75%: "Recommend compacting soon. Writing checkpoint."
- 85%: "Context critical. Writing checkpoint now. Suggest `/compact` or `/new`."

## OpenClaw Config Tuning

For users who want to tune their context behaviour, read `references/config-guide.md`.

## Operation Cost Reference

For per-operation cost estimates, read `references/operation-costs.md`.
