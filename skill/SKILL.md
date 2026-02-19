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

1. **Fixed baseline**: ~8% of context consumed before any conversation — system prompt, workspace files, skill descriptions, tool definitions. Actual size depends on workspace and skill count.
2. **60/40 rule**: ~60% of consumed context is tool outputs, ~40% conversation. Tool outputs are the primary target for savings.
3. **Compaction is lossy**: Summaries stack cumulatively. Each cycle raises the floor. After 3+ compactions, summaries alone can consume 30%+ of context.
4. **Sub-agents are disposable context**: A sub-agent can burn most of its context investigating something; only the summary (~500 tokens) enters main context.

Note: All percentages are relative to the model's context window (e.g. 200k for Claude Opus, 128k for GPT-4, etc.). Check `session_status` for the actual window size.

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
Fixed baseline:         ~8% (system prompt + workspace files + skills + tools)
Per user message:       ~100-500 tokens each
Per assistant response: ~200-1000 tokens each
Per tool call result:   ~500-5000 tokens each (exec/read heavy, search light)
Compaction summaries:   ~2000-5000 tokens each (cumulative!)
```

Count messages and tool calls in recent history, multiply by midpoint estimates. Report as ranges, not false precision.

### Spawn Policy

If `.context-policy.yml` exists in workspace root, use it as guidance for spawn thresholds and task categories. Otherwise use these defaults:

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

Before compaction or `/new`, write `.context-checkpoint.md` in the **workspace root** (the agent needs to read this post-compaction — it must be where the agent can find it):

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

**Coordination with OpenClaw memoryFlush:** OpenClaw may fire its own pre-compaction flush (writing to daily log). The checkpoint is complementary — the flush saves to the daily log, the checkpoint saves structured resume state. Both should exist. If the memoryFlush fires first, the checkpoint may not get written (compaction already in progress). For critical sessions, write checkpoints proactively at 75%, don't wait for 85%.

The `scripts/context-checkpoint.sh` script handles basic write/read/clear operations. For the full 5-section checkpoint, write the file directly — the script only covers task, state, and next steps.

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

## Session Profiling & Config Advice

After significant work (or on request), profile the current session and recommend config changes.

### Step 1: Classify the Session Pattern

Run `session_status`. Count approximate tool calls and message exchanges. Classify:

| Pattern | Signature | Example |
|---------|-----------|---------|
| **Tool-heavy** | Most context from tool results, many exec/read/web calls | Audits, migrations, test suites, debugging |
| **Conversational** | Most context from messages, few tool calls | Planning, discussion, decisions |
| **Mixed** | Roughly even split | Feature builds (discuss → code → test → discuss) |
| **Bursty** | Long quiet periods with intense tool bursts | Monitoring + incident response |

### Step 2: Recommend Config

| Pattern | reserveTokensFloor | pruning TTL | keepLastAssistants | Rationale |
|---------|-------------------|-------------|-------------------|-----------|
| Tool-heavy | `60000` | `1m` | `1` | Compact early, prune aggressively — tool output is disposable |
| Conversational | `30000` | `5m` | `3` | Protect conversation history, less pruning needed |
| Mixed | `50000` | `2m` | `2` | Balance: reasonable headroom + moderate pruning |
| Bursty | `50000` | `2m` | `1` | Prune burst results fast, keep headroom for next burst |

Also recommend:
- **Tool-heavy sessions**: Lower `minPrunableToolChars` to `10000` to catch medium outputs
- **Sessions with browser/canvas work**: Ensure those tools are in `tools.deny` list
- **Long-running sessions (>2h)**: Higher floor (`60000`) to survive multiple compactions

### Step 3: Report

Format recommendation as:

```
📊 Session Profile: {pattern}
  Context: {pct}% ({used}k/{total}k)
  Tool calls: ~{n} | Messages: ~{m} | Compactions: {c}

  Recommended config for this work style:
    reserveTokensFloor: {value} (current: {current})
    pruning TTL: {value} (current: {current})
    keepLastAssistants: {value} (current: {current})

  {specific_advice}
```

If the user agrees, apply changes following the procedure below.

### Step 4: Learn Over Time

After giving advice, note the session pattern and outcome in the daily log. Over multiple sessions, patterns emerge — the user's _typical_ work style becomes clear and default config can be permanently tuned.

## Applying Config Changes — Mandatory Procedure

When recommending config changes, follow this exact sequence. No shortcuts.

### 1. Backup First
```bash
cp /home/openclaw/.openclaw/openclaw.json \
   /home/openclaw/.openclaw/openclaw.json.backup-$(date +%Y%m%d-%H%M%S)
```

### 2. Write a Rollback Document
Create `/tmp/rollback-context-config.md` — NOT the workspace (user may not have access to the agent's workspace). `/tmp/` is universally accessible. The file survives long enough for rollback; if the machine reboots, the backup config file next to `openclaw.json` is the real safety net. Include:

```markdown
# Context Config Rollback — {date}

## What Changed
| Setting | Before | After | File |
|---------|--------|-------|------|
| ... | ... | ... | /home/openclaw/.openclaw/openclaw.json |

## Backup Location
/home/openclaw/.openclaw/openclaw.json.backup-{timestamp}

## How to Rollback
cp {backup_path} /home/openclaw/.openclaw/openclaw.json

## How to Restart the Gateway
Depends on local setup — check which applies:
- systemd: sudo systemctl restart openclaw-gateway
- CLI: openclaw gateway restart
- Manual process: pkill -f "openclaw gateway" && openclaw gateway start

## How to Check Health
curl -sf http://localhost:{port}/health && echo "OK" || echo "DOWN"
pgrep -af openclaw

## What to Do If Gateway Won't Start
1. Restore backup (cp command above)
2. Restart gateway
3. Check logs: journalctl -u openclaw-gateway --since "5 min ago"
```

### 3. Explain to the User BEFORE Applying
Tell them:
- **Which file** is being modified (full path)
- **What values** change (before → after table)
- **What "restart" means** — the OpenClaw gateway process restarts (not the machine, not Kubernetes, not SSH). Brief 2-3 second pause, then the session reconnects automatically.
- **Where the backup is** (full path)
- **Where the rollback doc is** (full path)
- **How to check** if something goes wrong

### 4. Apply with gateway config.patch
Use the `gateway` tool with `action: config.patch`. Include a clear `note` parameter — this message is delivered to the user after the gateway restarts.

### 5. Post-Restart Confirmation (MANDATORY)
After the gateway restarts and the session reconnects, **immediately confirm to the user**:

```
✅ Gateway is back. Config changes applied successfully.

What changed:
- Compaction trigger: {old} → {new} (compacts earlier, smaller summaries)
- Pruning TTL: {old} → {new} (clears stale tool output sooner)
- [etc.]

Rollback doc: /tmp/rollback-context-config.md
Backup: /home/openclaw/.openclaw/openclaw.json.backup-{timestamp}

Everything is working normally. Ready to continue.
```

**Never stay silent after a restart.** The user needs to know:
1. We're back
2. The changes landed
3. Where to find the rollback doc
4. That we're ready to continue

## Reference Docs

For detailed config options and profiles: `references/config-guide.md`
For per-operation cost estimates: `references/operation-costs.md`
