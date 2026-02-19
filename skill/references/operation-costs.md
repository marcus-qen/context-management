# Operation Cost Reference

Estimates from production measurements (Claude Opus, 200k context). Use chars/4 approximation.

## Fixed Baseline (~16k tokens, 8%)

| Component | Tokens | Notes |
|-----------|--------|-------|
| System prompt | ~4,500 | OpenClaw instructions |
| Workspace files | ~2,800 | AGENTS.md, SOUL.md, USER.md, MEMORY.md, etc. |
| Skill descriptions | ~5,100 | 64 skills × ~80 tokens each |
| Tool definitions | ~3,500 | ~20 tool JSON schemas |

## Per-Operation Costs

| Operation | Tokens | Context Impact |
|-----------|--------|---------------|
| Ask a question, get answer | 500-1,500 | Negligible |
| Read a file (small) | 500-2,000 | Low |
| Read a file (large) | 2,000-10,000 | Moderate-High |
| Edit a file | 500-1,500 | Low |
| exec (simple command) | 200-800 | Negligible |
| exec (verbose output) | 2,000-10,000 | High |
| Web search | 1,000-2,000 | Low |
| Web fetch | 2,000-8,000 | Moderate-High |
| Browser snapshot | 1,000-5,000 | Moderate |
| Run test suite (per test) | 2,000-5,000 | High |
| SSH + remote exec | 500-3,000 | Moderate |

## Session Burn Rates

| Activity | Tokens/minute | Time to 85% |
|----------|--------------|-------------|
| Casual conversation | 500-1,000 | 5-6 hours |
| Planning/discussion | 1,000-2,000 | 2-3 hours |
| Active coding/editing | 2,000-4,000 | 1-1.5 hours |
| Heavy testing/debugging | 3,000-6,000 | 30-60 min |
| Bulk operations | 5,000-10,000 | 20-30 min |

## Spawn Decision Heuristic

A sub-agent costs ~16k tokens of baseline overhead. Only worth spawning if the task would consume >20k tokens in main context. Rule of thumb:

- **<5 tool calls**: Keep in main (~5-15k tokens, not worth spawn overhead)
- **5-10 tool calls**: Spawn if context above 50%
- **>10 tool calls**: Always spawn (~30-100k tokens saved)

## Compaction Summary Costs

Each compaction produces a summary that persists in context:
- First compaction: ~2,000-4,000 tokens
- Second compaction: ~4,000-8,000 tokens (cumulative — includes prior summary)
- Third compaction: ~6,000-12,000 tokens
- After 5+ compactions: summaries alone may consume 20-30% of context
