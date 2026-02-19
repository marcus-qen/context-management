# Plan: Context Management Skill

A ClawHub skill that gives users visibility and control over context window consumption.

## Status: Draft

## Problem

Users have no visibility into what's consuming their context window. Compaction is opaque and janky. Sub-agent spawning is ad-hoc. The result is a frustrating cycle of filling up, losing thread, and destroying workflow productivity.

## Goals

1. **Visibility:** Users can ask "what's eating my context?" and get a breakdown
2. **Control:** Defined rules for when to spawn sub-agents vs work in main
3. **Proactive management:** Agent warns and acts before hitting compaction, not after
4. **Graceful transitions:** When compaction or `/new` is needed, make it smooth — structured handoff, not lossy summarisation

## What We Can Build Today (skill-level, no core changes)

### 1. Context Awareness Prompts

Teach the agent to respond to questions like:
- "What's eating my context?" → Estimate breakdown by category
- "How much runway do I have?" → Estimate remaining work capacity
- "Should we compact?" → Recommend based on current usage + planned work

### 2. Sub-Agent Spawn Policy

Define rules the agent follows for deciding main vs sub-agent:

```yaml
spawn_policy:
  always_spawn:
    - test_suites
    - multi_file_audits
    - build_pipelines
    - research_tasks
    - bulk_operations
  context_thresholds:
    - above: 50%
      spawn_if: "task involves >5 tool calls"
    - above: 70%
      spawn_if: "task involves >2 tool calls"
    - above: 85%
      action: "suggest compact or /new"
  never_spawn:
    - single_commands
    - conversations
    - quick_edits
```

### 3. Proactive Context Warnings

Agent checks context after every tool-heavy operation and warns:

```
⚠️ Context: 68% (136k/200k). This session has room for ~30 more 
tool calls before compaction. The current task (testing) is 
tool-heavy — recommend spawning a sub-agent for the remaining tests.
```

### 4. Pre-Compaction Checkpoint

Before compaction (or on `/compact`), write a structured state file:

```markdown
# .context-checkpoint.md
## Active Task
Building workspace-standard skill — currently testing

## Key State  
- SKILL.md: complete (233 lines)
- Scripts: complete and tested
- README: complete with safety/prompt sections
- Repo: pushed to marcus-qen/workspace-standard
- ClawHub: not yet published (need auth)

## What Was Just Discussed
- Keith asked about context token management
- Exploring whether to build a context-management skill

## Decisions Made This Session
1. workspace-standard: clean break (no symlinks), runbooks/ not docs/
2. Level 2 config (numbers + directory names configurable, roles opinionated)
3. Dedicated public repo under marcus-qen org

## Next Steps
1. Publish to ClawHub (need account)
2. Install from ClawHub as a test
3. Design context-management skill
```

This file survives compaction (it's on disk). After compaction, the agent reads it to restore context. Much better than relying on the lossy summary alone.

### 5. Graceful `/new` Handoff

Instead of a hard `/new` (blank slate), a pattern where:
1. Agent writes the checkpoint file
2. User runs `/new`
3. New session reads the checkpoint file on first turn
4. Resumes with structured context, not lossy summary

## What Needs Core Changes (upstream to OpenClaw)

### 1. Token Breakdown in `/status`
Show per-category token consumption:
```
📊 Context: 164k/200k (82%)
  System:       15k (7%)
  Conversation: 49k (25%)
  Tool outputs:  95k (48%)
  Summaries:      5k (2%)
```

### 2. Non-Cumulative Compaction
Replace the previous summary instead of stacking. Each compaction produces ONE summary that replaces the old one, not a new summary layered on top.

### 3. Selective Compaction
Compact tool results but keep conversation messages. Tool outputs are the biggest consumer and the most disposable.

### 4. Pre-Execution Token Estimates
Before running a tool call, estimate the output size and warn if it'll consume >5% of remaining context.

### 5. `/handoff` Command
Structured session transition: write state → start new session → load state. Built into OpenClaw rather than hacked with file writes.

## Skill Structure

```
context-management/
├── SKILL.md                    # Main skill: prompts, policies, procedures
├── scripts/
│   └── context-checkpoint.sh   # Write/read checkpoint files
├── references/
│   ├── context-anatomy.md      # What consumes context (for agent reference)
│   ├── token-costs.md          # Per-operation cost estimates
│   └── config-guide.md         # How to tune OpenClaw config
└── .context-policy.yml.example # Example spawn policy config
```

## Open Questions

1. **Can we estimate context percentage from within a skill?** `session_status` gives us the number but not the breakdown. We'd be estimating based on conversation length and tool call count.

2. **Should the checkpoint file be automatic (on every compaction) or manual?** Automatic is safer but adds a file write every compaction. Manual means the user/agent must remember.

3. **How do we handle the spawn policy?** The agent reads it from a config file? It's baked into the skill? The user defines it in their workspace?

4. **Is this one skill or two?** "Context visibility" and "sub-agent policy" are related but distinct. Could be separate skills.

## Timeline

- **Phase 1:** Capture research and measurements (THIS REPO — done)
- **Phase 2:** Build the skill with checkpoint + spawn policy + awareness prompts
- **Phase 3:** Test in production (install on our workspace, run for a week)
- **Phase 4:** Publish to ClawHub
- **Phase 5:** Submit upstream issues for core improvements
