# context-management

Research, analysis, and tooling for AI agent context window management.

**The problem:** Long-running AI agent sessions fill their context window, trigger compaction, lose thread, and productivity collapses. Compaction barely moves the dial. Users have no visibility into what's eating their tokens. Sub-agent spawning is ad-hoc. There's no playbook.

**This repo captures:** Everything we've learned running a 24/7 AI agent (OpenClaw on Claude Opus, 200k context window) across weeks of continuous platform engineering work. Real numbers, real pain points, real analysis.

## Contents

| File | What it covers |
|------|---------------|
| [`research/context-window-anatomy.md`](research/context-window-anatomy.md) | Full breakdown of what consumes context: system prompt, loaded files, skill descriptions, tool definitions, conversation, tool outputs. Real measurements. |
| [`research/compaction-analysis.md`](research/compaction-analysis.md) | Why compaction barely moves the dial. The death spiral. Cumulative summaries. Real numbers from production sessions. |
| [`research/token-consumers.md`](research/token-consumers.md) | What actually burns tokens during a session. Tool outputs vs conversation. The 60/40 ratio. Which operations are expensive. |
| [`research/sub-agent-patterns.md`](research/sub-agent-patterns.md) | When to spawn sub-agents, what happens to their context, trade-offs of "always spawn" vs selective spawning, handoff friction. |
| [`research/pruning-mechanics.md`](research/pruning-mechanics.md) | How OpenClaw's session pruning works, why it often doesn't fire, configuration options, and what it can/can't fix. |
| [`references/openclaw-config.md`](references/openclaw-config.md) | All context-related OpenClaw configuration: compaction modes, pruning settings, cache TTL, with explanations of what each does. |
| [`references/measurements.md`](references/measurements.md) | Baseline measurements from a real workspace: system prompt cost, loaded file costs, skill description overhead, tool definition overhead. |
| [`plans/context-management-skill.md`](plans/context-management-skill.md) | Plan for a ClawHub skill that gives users visibility and control over context consumption. |

## Key Findings

1. **Fixed baseline is ~15k tokens (7.5%)** before you say a word — system prompt, loaded workspace files, skill descriptions, tool definitions.

2. **Tool outputs consume ~60% of context** in a working session. Conversation is ~40%. The tools are the real killer.

3. **Compaction drops ~20%, not ~60%.** From 85% you land at 65-75%, not 20%. The summary + fixed overhead + kept recent messages eat most of what was freed.

4. **Compaction summaries accumulate.** Each compaction stacks on the previous summary. After 3 compactions, summaries alone can be 40-50k tokens. The floor keeps rising.

5. **Pruning often doesn't fire** during active sessions because the cache-TTL hasn't expired. Tool results accumulate indefinitely during rapid conversation.

6. **Sub-agents are disposable context** — this is their superpower. Run heavy work, extract the result, throw away 150k of tool output.

7. **Users have zero visibility** into what's consuming their context. One number (`164k/200k`) tells you nothing.

## Who This Is For

- OpenClaw users hitting context limits and compaction loops
- AI agent developers building memory/context management systems
- Anyone running long-duration agent sessions (hours/days)

## Related

- [workspace-standard](https://github.com/marcus-qen/workspace-standard) — Workspace file organisation (reduces MEMORY.md baseline cost)
- [OpenClaw docs: Compaction](https://docs.openclaw.ai/concepts/compaction)
- [OpenClaw docs: Session Pruning](https://docs.openclaw.ai/concepts/session-pruning)

## License

MIT
