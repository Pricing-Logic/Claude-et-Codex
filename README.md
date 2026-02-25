# Claude-et-Codex: Cross-AI Plan Review

A [Claude Code](https://claude.ai/claude-code) skill that calls [OpenAI Codex CLI](https://github.com/openai/codex) to review your implementation plans before you execute them. Two AIs, one plan, fewer blind spots.

## What It Does

Before you start coding, this skill sends your implementation plan to Codex for a second opinion. Codex reads your actual project files and flags issues that would cause bugs, break things, or miss required steps. Claude Code then filters out the noise (Codex loves to over-engineer) and presents only what matters.

```
You write a plan
        |
        v
Claude Code assesses complexity (LOW / MEDIUM / HIGH)
        |
        v
Selects appropriate Codex model + reasoning level
        |
        v
Codex reviews plan against your actual codebase
        |
        v
Claude Code filters out noise (security theater, over-abstraction)
        |
        v
You get: critical findings only, with file:line evidence
```

## Quick Start

### Prerequisites

- [Claude Code](https://claude.ai/claude-code) (Anthropic CLI)
- [Codex CLI](https://github.com/openai/codex) >= 0.100.0
- OpenAI API key configured (`codex login` or `OPENAI_API_KEY`)
- macOS or Linux (Windows via WSL)

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

### Automatic Model Selection

The skill reads your plan and auto-selects the right Codex model:

| Plan Complexity | Model | Reasoning | When |
|-----------------|-------|-----------|------|
| LOW | Spark | Low | UI changes, config, styling (1-3 files) |
| MEDIUM | Config default | Config default | New endpoints, logic changes (4-8 files) |
| HIGH | Non-Spark | High | Data models, auth, migrations, cross-system (9+ files) |

You never need to pick a model — but you can override with "use o3" or "use spark" if you want.

See [docs/model-selection.md](docs/model-selection.md) for the full guide on Spark vs Non-Spark and reasoning levels.

## Example Output

```
Codex Plan Review — Blind Spot Report

Model used: gpt-5.3-codex-spark (medium reasoning)
Complexity assessed: MEDIUM — 5 files, new API endpoint with DB queries
Plan reviewed: Add user preferences API with PostgreSQL storage

Critical Findings (integrate these before executing):
1. Missing index on `user_id` in preferences table — queries will
   full-scan as the table grows. Evidence: schema.sql:47
2. The PATCH endpoint doesn't validate that preferences keys exist
   in the enum — arbitrary keys will be silently stored.
   Evidence: routes/preferences.ts:23

Noted but Non-Critical (awareness only):
- Codex suggested rate limiting on the GET endpoint — filtered as
  premature for an internal API.

Verdict: Plan needs 2 adjustments before executing.
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
- The skill sets a 5-minute timeout. If it times out, it reports and lets you proceed
- For faster reviews, say "use spark" to force the faster model

**No output / empty review:**
- Verify Codex version: `codex -V` (need >= 0.100.0)
- Check that you're in a git repo or valid project directory

More in [INSTALL.md](INSTALL.md#troubleshooting).

## How It Was Built

This skill went through 4 versions in a single session, with Codex reviewing each iteration of itself. Key bugs caught by Codex during development:

- **v1.1.0**: Codex caught that `timeout` doesn't exist on macOS
- **v1.2.0**: Codex caught a critical stdin bug — piping a file AND using a heredoc on the same command silently drops the plan. Reviews were returning "clean" because Codex never saw the plan.
- **v1.3.0**: Codex caught that macOS `mktemp` with a `.md` suffix doesn't randomize the filename, and that Claude Code's separate shell invocations break `trap` cleanup and variable persistence.

Full story in [docs/how-it-was-built.md](docs/how-it-was-built.md).

## License

[Unlicense](LICENSE) — public domain. Do whatever you want with it.
