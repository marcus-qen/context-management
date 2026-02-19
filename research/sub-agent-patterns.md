# Sub-Agent Patterns

When to spawn, what happens to their context, and the trade-offs.

## What Sub-Agents Are

A sub-agent (`sessions_spawn`) creates an **isolated session** with its own context window. It runs a task to completion, then announces the result back to the main session. The result is a summary — the full context (tool outputs, reasoning, intermediate steps) stays in the sub-agent's session.

**This is their superpower:** disposable context. A sub-agent can burn 150k tokens of tool output investigating something, and only 500 tokens of summary come back to the main session.

## What Happens to Sub-Agent Context

1. **During execution:** The sub-agent has its own 200k context window. It loads the same workspace files (AGENTS.md, MEMORY.md, etc.) and has access to the same tools.

2. **On completion:** The sub-agent writes a summary/announcement. This summary is delivered to the main session as a message. The sub-agent's session persists on disk (JSONL history) but its context window is released.

3. **After completion:** The sub-agent's session can be read with `sessions_history` if you need details. But by default, only the summary enters the main session's context. The 150k of intermediate work is gone from active context.

4. **Cleanup:** Sessions can be configured to delete after completion (`cleanup: "delete"`) or kept (`cleanup: "keep"`). Kept sessions are useful for debugging but accumulate disk space.

## When to Spawn

### Definitely Spawn

| Task | Why |
|------|-----|
| Test suites (>3 tests) | 10+ test runs × 2-3k tokens each = 25-30k tokens. Not worth polluting main context. |
| Multi-file audits | Reading 10+ files to check something. Each read is 1-3k tokens. |
| Build/deploy pipelines | Verbose output, multiple steps, lots of exec calls. |
| Research tasks | Web searches + fetches + analysis. Multiple heavy tool calls. |
| Code review | Reading diffs, checking files, running tests — lots of reads. |
| Competitor analysis | Downloading and reading multiple external resources. |
| Bulk file operations | Adding front-matter to 50 files, for example. |

### Probably Keep in Main

| Task | Why |
|------|-----|
| Quick single command | One `exec` = 100-500 tokens. Spawning a sub-agent has overhead too. |
| Conversation/discussion | The whole point is interactive — you want to see the thinking. |
| File edits (1-3 files) | A few reads + edits is ~2-5k tokens. Manageable. |
| Status checks | `session_status`, `git status`, quick lookups. Nearly free. |
| Decisions requiring user input | Sub-agents can't ask questions mid-task. |

### The Grey Zone

| Task | Consideration |
|------|--------------|
| 5 tool calls | Borderline. If context is below 50%, keep in main. If above 50%, spawn. |
| File creation (like this repo) | Writing files is cheap, but if it involves research + testing + iteration, spawn. |
| Debugging | Needs interactive back-and-forth but generates lots of tool output. Consider spawning the investigation and keeping the discussion in main. |

## The "Always Spawn" Pattern

**What if every task ran in a sub-agent?**

The main session becomes purely conversational. You talk, the agent plans, it spawns work, results come back as summaries. The main session never fills with tool output.

### Pros
- Main context stays lean indefinitely
- No compaction death spiral
- Clean separation of concerns
- Can run multiple tasks in parallel

### Cons
- **No interactivity during work.** You can't say "wait, try X instead" while a sub-agent is running. You'd have to kill it and spawn a new one, or steer it (limited).
- **Latency.** Sub-agent startup has overhead. Quick tasks feel slow.
- **Lossy handoff.** The summary might miss details you'd have caught watching the work happen.
- **Overhead.** Each sub-agent loads the full system prompt (~15k tokens). For a 500-token task, that's 30× overhead.
- **No shared learning.** Sub-agent A discovers something useful. Sub-agent B doesn't know about it unless it's in the workspace files.

### Verdict
"Always spawn" is too extreme. But **"spawn by default for tasks with >5 tool calls"** might be the right heuristic. Quick questions and simple edits stay in main. Everything else gets spawned.

## Spawn Policy Proposal

A context-management skill could enforce rules like:

```yaml
# .context-policy.yml
spawn_policy:
  # Always spawn these task types
  always_spawn:
    - test_suites
    - multi_file_audits
    - build_pipelines
    - research_tasks

  # Spawn based on context pressure
  context_thresholds:
    - above: 50%      # When context is above 50%
      spawn_if: ">3 tool calls estimated"
    - above: 70%      # When context is above 70%
      spawn_if: ">1 tool call estimated"
    - above: 85%      # When context is above 85%
      action: "compact or /new"

  # Never spawn (keep interactive)
  never_spawn:
    - single_commands
    - conversations
    - quick_edits
```

## Communication Patterns

### Main → Sub-Agent
The task description is the only input. Make it detailed:
- What to do
- What files to read first
- What success looks like
- Where to write results
- What NOT to do

### Sub-Agent → Main
The announcement/summary. Should include:
- What was done
- What was found
- What files were changed
- Whether it succeeded or failed
- Key decisions made

### During Execution
Limited options:
- `subagents steer` — send a message to a running sub-agent
- `subagents kill` — terminate it
- `sessions_history` — peek at what it's doing

Steering is useful but the sub-agent might not see the message immediately (it processes between turns).

## Sub-Agent Anti-Patterns

1. **Spawning for a 2-second task.** The overhead of spawning + summarising exceeds the cost of just doing it in main.

2. **Spawning without detailed instructions.** The sub-agent has no conversation context — it only knows what you tell it in the task description. Vague tasks get vague results.

3. **Spawning and polling.** Don't loop checking if the sub-agent is done. The result is push-based — it announces when complete.

4. **Not checking the result.** The summary might say "done" but the work might have errors. Verify in main session before moving on.

5. **Running 10 sub-agents simultaneously.** They share the same machine resources and tools. Concurrent git operations, file writes, or infrastructure changes can conflict.
