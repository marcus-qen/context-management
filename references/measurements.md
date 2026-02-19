# Baseline Measurements

Real measurements from a production OpenClaw workspace. Taken 2026-02-19.

## Environment

- **Model:** Claude Opus (anthropic/claude-opus-4-6)
- **Context window:** 200,000 tokens
- **OpenClaw version:** 2026.2.17
- **Workspace:** 12 local skills, 52 built-in skills, 7 auto-loaded files

## Auto-Loaded Workspace Files

These files are injected into every system prompt, every turn:

| File | Characters | Est. Tokens | Notes |
|------|-----------|-------------|-------|
| AGENTS.md | 3,550 | ~887 | Operating rules |
| SOUL.md | 1,356 | ~339 | Persona definition |
| USER.md | 1,753 | ~438 | User profile |
| TOOLS.md | 860 | ~215 | Tool notes |
| IDENTITY.md | 636 | ~159 | Agent identity |
| HEARTBEAT.md | 168 | ~42 | Heartbeat tasks (empty/comments = minimal) |
| MEMORY.md | 2,996 | ~749 | Current state (73 lines, post workspace-standard) |
| **Total** | **11,319** | **~2,829** | |

### Before vs After workspace-standard

| File | Before (tokens) | After (tokens) | Saved |
|------|----------------|----------------|-------|
| MEMORY.md | ~1,800 (224 lines) | ~749 (73 lines) | ~1,050 |

## Skill Description Overhead

Each skill's `description` field is included in the system prompt for skill matching:

| Category | Count | Est. tokens per skill | Total |
|----------|-------|-----------------------|-------|
| Local skills | 12 | ~80-150 | ~1,200 |
| Built-in skills | 52 | ~50-100 | ~3,900 |
| **Total** | **64** | | **~5,100** |

### Note on skill count

64 skills is high. Many built-in skills may never trigger for a given user. Each unused skill still costs ~80 tokens of system prompt space. Disabling unused skills would reduce baseline.

## Tool Definitions

Each available tool's JSON schema is included in the system prompt:

| Tools | Est. tokens |
|-------|-------------|
| ~20+ tool definitions | ~3,000-4,000 |

## Total Fixed Baseline

```
System prompt (OpenClaw instructions):    ~4,500 tokens
Workspace files:                          ~2,829 tokens
Skill descriptions:                       ~5,100 tokens
Tool definitions:                         ~3,500 tokens
───────────────────────────────────────────────────────
TOTAL BASELINE:                           ~15,929 tokens (8%)
```

This is the cost of having the agent ready to respond, before any conversation happens.

## Session Token Burn Rates

Observed during different types of work:

| Activity | Tokens/minute (est.) | Time to 85% from clean |
|----------|---------------------|----------------------|
| Casual conversation | 500-1,000 | 5-6 hours |
| Planning/discussion | 1,000-2,000 | 2-3 hours |
| Active coding/editing | 2,000-4,000 | 1-1.5 hours |
| Heavy testing/debugging | 3,000-6,000 | 30-60 minutes |
| Bulk operations (audits, migrations) | 5,000-10,000 | 20-30 minutes |

## Cost of Common Operations (in context tokens)

| Operation | Tokens consumed | Context impact |
|-----------|----------------|---------------|
| Ask a question, get an answer | 500-1,500 | Negligible |
| Read + edit a file | 2,000-5,000 | Low |
| Run a test and discuss results | 3,000-8,000 | Moderate |
| Debug a failing test (5 iterations) | 15,000-30,000 | High |
| Run 10-test validation suite | 25,000-40,000 | Very high |
| Full file walkthrough with review | 10,000-20,000 | High |
| Web research (5 searches + 3 fetches) | 10,000-25,000 | High |

## Methodology

Token estimates use chars/4 approximation. Actual tokenization varies by content (code tends to tokenize less efficiently than prose). These are order-of-magnitude estimates, not exact measurements.

A more accurate measurement would require access to the actual token counts from the API response headers, which OpenClaw doesn't currently expose per-component.
