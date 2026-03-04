# Claude-et-Codex: Cross-AI Plan Review

A [Claude Code](https://claude.ai/claude-code) skill that calls [OpenAI Codex CLI](https://github.com/openai/codex) with **multi-agent review** to catch blind spots in your implementation plans before you execute them. Two AIs, parallel sub-agents, one plan, fewer blind spots.

## What It Does

Before you start coding, this skill sends your implementation plan to Codex for a second opinion. **v2.0** uses Codex's `multi_agent` feature to spawn parallel sub-agents — an **explorer** that reads your actual code files, and an **architecture reviewer** that evaluates design and risk — then synthesizes both into a structured verdict. Claude Code filters out the noise (Codex loves to over-engineer) and presents only what matters.

```
You write a plan
        |
        v
Claude Code assesses complexity (LOW / MEDIUM / HIGH)
        |
        v
Selects model + reasoning level + thread count
        |
        v
Codex spawns parallel sub-agents:
  +---> Explorer: reads actual files, validates feasibility
  +---> Architecture: evaluates design, sequencing, risk
        |
        v
Codex parent agent merges findings
        |
        v
Claude Code filters noise + deduplicates sub-agent echo
        |
        v
You get: APPROVE / APPROVE_WITH_CHANGES / BLOCK verdict
         with categorized findings and file:line evidence
```

## Quick Start

### Prerequisites

- [Claude Code](https://claude.ai/claude-code) (Anthropic CLI)
- [Codex CLI](https://github.com/openai/codex) >= 0.104.0
- OpenAI API key configured (`codex login` or `OPENAI_API_KEY`)
- macOS or Linux (Windows via WSL)
- `multi_agent` feature enabled in Codex (recommended — see [Configuration](#configuration))

### Install

```bash
git clone https://github.com/Pricing-Logic/Claude-et-Codex.git
mkdir -p ~/.claude/skills/codex-plan-review
cp Claude-et-Codex/skill/SKILL.md ~/.claude/skills/codex-plan-review/SKILL.md
```

Or just copy `skill/SKILL.md` to `~/.claude/skills/codex-plan-review/SKILL.md`.

Restart Claude Code after installing.

See [INSTALL.md](INSTALL.md) for detailed setup, verification, and troubleshooting.

### Use

Say any of these to Claude Code:

- "review my plan"
- "second opinion on this"
- "blind spot check"
- "check with codex"
- `/codex-plan-review`

The skill activates automatically when you've written an implementation plan or are about to execute a multi-file change.

## How It Works

### The Two-Layer Filter

Codex has a well-documented tendency to over-engineer: adding auth checks to internal functions, suggesting factory patterns for one-off operations, and recommending security hardening beyond any reasonable threat model.

This skill handles it with two filter layers:

1. **Prompt filter**: The instructions to Codex explicitly say "DO NOT suggest minor style improvements, extra validation, security hardening beyond context..." — this pre-filters ~60% of the noise.
2. **Safety net filter**: Claude Code reviews Codex's output and applies a Keep/Discard table to catch the remaining noise that Codex didn't self-filter.

### Automatic Model & Concurrency Selection

The skill reads your plan and auto-selects the right Codex model, reasoning effort, and multi-agent thread count:

| Plan Complexity | Model | Reasoning | Threads | When |
|-----------------|-------|-----------|---------|------|
| LOW | Spark | Medium | 2 | UI changes, config, styling (1-3 files) |
| MEDIUM | Codex-5.3 | High | 3 | New endpoints, logic changes (4-8 files) |
| HIGH | Codex-5.3 | XHigh | 4 | Data models, auth, migrations, cross-system (9+ files) |

The skill always explicitly selects a model for deterministic review quality — it never defers to user config defaults. If a LOW-complexity plan touches auth, migrations, concurrency, or other sensitive areas, it auto-escalates to MEDIUM routing.

You never need to pick a model — but you can override with "use o3", "use spark", or specify reasoning effort like "xhigh thinking". You can also request single-shot mode with "skip multi-agent" or "no sub-agents".

See [docs/model-selection.md](docs/model-selection.md) for the full guide on Spark vs Non-Spark, reasoning levels, and concurrency.

## Example Output

```
Codex Plan Review — Multi-Agent Blind Spot Report

Mode: Multi-agent (explorer + architecture reviewer)
Model used: gpt-5.3-codex (high reasoning)
Complexity assessed: MEDIUM — 5 files, new API endpoint with DB queries
Plan reviewed: Add user preferences API with PostgreSQL storage

Verdict: APPROVE_WITH_CHANGES

Critical Findings (integrate these before executing):
1. Missing index on `user_id` in preferences table — queries will
   full-scan as the table grows. Evidence: schema.sql:47
2. The PATCH endpoint doesn't validate that preferences keys exist
   in the enum — arbitrary keys will be silently stored.
   Evidence: routes/preferences.ts:23

Medium Risks (awareness, may need action):
- No rollback migration provided for the new table.

Missing Steps (add to plan before executing):
- Add DOWN migration for preferences table drop.

Noted but Non-Critical (filtered out):
- Codex suggested rate limiting on the GET endpoint — filtered as
  premature for an internal API.
```

See [examples/](examples/) for a full sample plan and output.

## What's In Here

```
Claude-et-Codex/
  README.md              You are here
  INSTALL.md             Detailed setup, verification, troubleshooting
  LICENSE                Unlicense (public domain)
  skill/
    SKILL.md             The skill file (copy to ~/.claude/skills/)
  docs/
    model-selection.md   Spark vs Non-Spark, reasoning levels, decision matrix
    how-it-was-built.md  Development story: 4 versions, 3 Codex reviews
    full-reference.md    Complete reference with changelog
  examples/
    sample-plan.md       Example plan in the required template format
    sample-output.md     What Codex review output looks like
```

## Troubleshooting

**Skill doesn't trigger:**
1. Check file exists: `ls ~/.claude/skills/codex-plan-review/SKILL.md`
2. Restart Claude Code (skills are discovered at startup)
3. Verify frontmatter is intact (the `---` delimiters and `name:`/`description:` fields)

**Codex auth error:**
- Run `codex login` or set `OPENAI_API_KEY` environment variable

**Codex takes too long:**
- The skill sets a 10-minute timeout (multi-agent reviews need more time than single-shot). If it times out, it reports and lets you proceed
- For faster reviews, say "use spark" to force the faster model, or "skip multi-agent" for single-shot mode

**No output / empty review:**
- Verify Codex version: `codex -V` (need >= 0.104.0)
- Check that you're in a git repo or valid project directory

**Multi-agent not working:**
- Verify feature is enabled: `codex features list | grep multi_agent`
- Add `multi_agent = true` under `[features]` in `~/.codex/config.toml`
- If you have `[collab]` in your config, rename it to `[features]` with `multi_agent = true` (legacy name deprecated)
- The skill falls back to single-shot mode automatically if multi-agent is unavailable

More in [INSTALL.md](INSTALL.md#troubleshooting).

## Configuration

### Enabling Multi-Agent

Add to `~/.codex/config.toml`:

```toml
[features]
multi_agent = true
```

The skill also passes `--enable multi_agent` per-call, but having it in config avoids deprecation warnings.

**Note:** If your config has `[collab]`, rename it. The `collab` feature name was deprecated in favor of `multi_agent`.

## How It Was Built

This skill went through 4 versions in a single session, with Codex reviewing each iteration of itself. Key bugs caught by Codex during development:

- **v1.1.0**: Codex caught that `timeout` doesn't exist on macOS
- **v1.2.0**: Codex caught a critical stdin bug — piping a file AND using a heredoc on the same command silently drops the plan. Reviews were returning "clean" because Codex never saw the plan.
- **v1.3.0**: Codex caught that macOS `mktemp` with a `.md` suffix doesn't randomize the filename, and that Claude Code's separate shell invocations break `trap` cleanup and variable persistence.
- **v1.5.0**: Fixed zsh glob bug with nvm path detection, added explicit PATH setup blocks, upgraded model routing (always explicit, never defers to config defaults), and added auto-escalation triggers for sensitive plan areas.
- **v2.0.0**: Multi-agent review mode — Codex now spawns parallel sub-agents (explorer + architecture reviewer) for deeper reviews. Structured verdicts (APPROVE / APPROVE_WITH_CHANGES / BLOCK), concurrency controls, read-only sandbox, automatic single-shot fallback, and legacy `collab` → `multi_agent` migration guidance. Codex itself reviewed the v2.0 upgrade and contributed to the multi-agent prompt design.

Full story in [docs/how-it-was-built.md](docs/how-it-was-built.md).

## License

[Unlicense](LICENSE) — public domain. Do whatever you want with it.
