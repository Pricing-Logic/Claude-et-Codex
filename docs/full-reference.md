# Codex Plan Review — Claude Code Skill

> **Version:** 1.3.0
> **Purpose:** Cross-check implementation plans against OpenAI's Codex CLI to catch blind spots before coding
> **Platform:** Claude Code (Anthropic CLI) orchestrating Codex CLI (OpenAI)
> **Requirements:** Codex CLI >= 0.100.0, Claude Code, macOS or Linux

---

## Table of Contents

1. [Overview](#overview)
2. [The Skill (SKILL.md)](#the-skill)
3. [Model Selection Guide](#model-selection-guide)
4. [Installation](#installation)
5. [Changelog](#changelog)

---

## Overview

This skill lets Claude Code call OpenAI's Codex CLI as a second reviewer before executing implementation plans. The two AIs think differently — Claude tends toward pragmatism, Codex tends toward thoroughness — so cross-checking catches blind spots neither would find alone.

The key insight: Codex over-engineers security and adds unnecessary complexity. The skill handles this by applying a two-layer filter — the prompt pre-filters Codex's output, then Claude Code applies a safety net filter before presenting findings to the user.

### What It Catches

- Runtime errors and crashes the plan would introduce
- Breaking changes to existing functionality
- Data loss or corruption risks
- Missing integration points (imports, routes, configs)
- Wrong assumptions about APIs, schemas, or data flow
- Race conditions and state management bugs
- Forgotten migrations, env variables, or deployment steps

### What It Filters Out

- "Consider adding" suggestions
- Extra error handling for impossible scenarios
- Security hardening beyond the threat model
- Architectural preferences and pattern suggestions
- Performance optimizations without evidence
- Additional logging and monitoring

---

## The Skill

Copy the content below into `~/.claude/skills/codex-plan-review/SKILL.md`. After creating or updating the file, **restart Claude Code** so it discovers the new skill.

<!-- IMPORTANT: This section must be kept in sync with the actual SKILL.md file.
     Source of truth: ~/.claude/skills/codex-plan-review/SKILL.md -->

~~~markdown
---
name: codex-plan-review
version: 1.3.0
description: "Cross-check implementation plans against Codex CLI to catch blind spots before coding. Use this skill BEFORE executing any implementation plan — when you've drafted a plan for a new feature, a refactor, a bug fix, or any multi-file change. Also use when the user says things like 'plan review', 'check with codex', 'second opinion', 'blind spot check', 'review my plan', or 'cross-check this'. The whole point is to get a second AI perspective on architectural decisions and catch things Claude Code might miss, then filter the feedback to only what actually matters."
---

# Codex Plan Review — Blind Spot Detection

You have a plan (or are about to finalize one). Before executing it, you're going to get a second opinion from OpenAI's Codex CLI to catch blind spots. Codex thinks differently than you do — it'll spot things you miss. But it also tends to over-engineer security and add unnecessary complexity. Your job is to be a smart filter: extract the critical insights, discard the noise.

**Requirements:** Codex CLI >= 0.100.0 (`codex -V` to check). Install with `npm install -g @openai/codex` if missing.

**Privacy note:** This skill gives Codex read access to project files in the working directory via `-C`. For sensitive or proprietary repos, confirm with the user before proceeding.

**Platform:** macOS and Linux. Temp files use `/tmp/`. Windows users need WSL or equivalent.

## When This Triggers

- You've just written or finalized an implementation plan
- The user asks for a "plan review", "second opinion", or "blind spot check"
- Before executing a multi-step plan that touches 3+ files
- The user explicitly invokes `/codex-plan-review`

## The Workflow

### Step 0: Preflight Check

Before anything else, verify Codex is installed and meets the minimum version:

```bash
CODEX_RAW=$(codex -V 2>/dev/null) || { echo "ERROR: Codex CLI not installed. Install with: npm install -g @openai/codex"; exit 1; }
CODEX_VERSION=$(echo "$CODEX_RAW" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')
if [ -z "$CODEX_VERSION" ]; then echo "ERROR: Could not parse Codex version from: $CODEX_RAW"; exit 1; fi
echo "Found: codex-cli $CODEX_VERSION"
MIN_VERSION="0.100.0"
if [ "$(printf '%s\n' "$MIN_VERSION" "$CODEX_VERSION" | sort -V | head -n1)" != "$MIN_VERSION" ]; then
  echo "WARNING: Codex CLI $CODEX_VERSION is below minimum $MIN_VERSION — some flags may not work."
fi
```

If `codex exec` later fails with an authentication error, tell the user to run `codex login` or set the `OPENAI_API_KEY` environment variable.

### Step 1: Assess Complexity & Select Model

Before calling Codex, assess the plan's complexity to determine which model and reasoning effort to use. This should happen automatically — don't ask the user unless they've expressed a preference.

**Complexity signals:**

| Signal | LOW | MEDIUM | HIGH |
|--------|-----|--------|------|
| Files changed | 1-3 | 4-8 | 9+ |
| Scope | UI/styling, copy, config | New endpoints, logic changes, refactors | Data model, auth, migrations, cross-system |
| Risk | Reversible, no data impact | Could break features | Data loss, security, breaking changes |
| Dependencies | Self-contained | Touches shared code | Cross-package, external APIs |

**Model selection based on complexity:**

| Complexity | Model | Reasoning | Flags to add to `codex exec` |
|------------|-------|-----------|------------------------------|
| LOW | spark (fast, cheap) | low | `-m gpt-5.3-codex-spark -c model_reasoning_effort="low"` |
| MEDIUM | config default | config default | *(no override — uses ~/.codex/config.toml)* |
| HIGH | non-spark (deeper) | high | `-m gpt-5.3-codex -c model_reasoning_effort="high"` |

Rationale: Spark models are optimized for speed and code tasks — perfect for most reviews. Non-spark models reason more carefully about architecture and edge cases, which matters for high-risk plans. The skill defaults to whatever the user has configured in `~/.codex/config.toml` and only overrides when the complexity warrants it.

**Note:** Model names evolve. If `gpt-5.3-codex-spark` or `gpt-5.3-codex` are no longer available, check `codex` docs for current model IDs and substitute accordingly.

If the user explicitly requests a model (e.g., "use o3", "use spark"), always respect that over the auto-selection.

### Step 2: Prepare the Plan Document

Assemble a clear plan document. If you haven't already written one, draft one now using this minimum template:

```markdown
## Objective
[What are we building/changing and why]

## Key Files to Examine
[3-5 most important files Codex should read for context — these are the files
that contain the code most affected by or relevant to the plan]
- `path/to/file1.ts` — [why it matters]
- `path/to/file2.ts` — [why it matters]

## Files to Modify
[Every file path with a 1-2 line summary of changes]

## Files to Create
[Any new files with their purpose — omit if none]

## Architecture Decisions
[Key design choices and why]

## Constraints / Non-Goals
[What this plan explicitly does NOT do — helps Codex avoid suggesting out-of-scope work]

## Sequence
[What order to make changes]
```

The "Key Files to Examine" and "Constraints / Non-Goals" sections are important — they focus Codex on what matters and prevent it from wandering into irrelevant files or suggesting out-of-scope additions.

If a `writing-plans` skill is available, use it to produce the plan.

### Steps 3-5: Build Request, Call Codex, Read Output

These steps MUST run as a single bash command because Claude Code runs each command in a separate shell — variables like `$PLAN_FILE` would be lost between steps. Run the entire block below as one command.

Choose the model flags based on your Step 1 assessment:
- **MEDIUM**: no extra flags (uses config default)
- **LOW**: add `-m gpt-5.3-codex-spark -c model_reasoning_effort="low"` to the `codex exec` line
- **HIGH**: add `-m gpt-5.3-codex -c model_reasoning_effort="high"` to the `codex exec` line

```bash
# --- Create unique temp files (mktemp without extension, then rename for .md) ---
PLAN_FILE=$(mktemp /tmp/codex-plan-XXXXXX) && mv "$PLAN_FILE" "${PLAN_FILE}.md" && PLAN_FILE="${PLAN_FILE}.md"
REQUEST_FILE=$(mktemp /tmp/codex-request-XXXXXX) && mv "$REQUEST_FILE" "${REQUEST_FILE}.md" && REQUEST_FILE="${REQUEST_FILE}.md"
OUTPUT_FILE=$(mktemp /tmp/codex-output-XXXXXX) && mv "$OUTPUT_FILE" "${OUTPUT_FILE}.md" && OUTPUT_FILE="${OUTPUT_FILE}.md"

# --- Write the plan ---
cat > "$PLAN_FILE" << 'PLAN_EOF'
[Your assembled plan goes here]
PLAN_EOF

# --- Build combined request (instructions + plan in one file) ---
cat > "$REQUEST_FILE" << 'INSTRUCTIONS_EOF'
You are reviewing an implementation plan created by another AI assistant (Claude Code). Your job is to find CRITICAL blind spots — things that will cause bugs, data loss, race conditions, breaking changes, or architectural problems.

You have access to the project files in this directory. Prioritize reading files listed under "Key Files to Examine" in the plan, then check other referenced files as needed.

DO NOT suggest:
- Minor style improvements
- Additional logging or monitoring
- Extra validation that isn't strictly necessary
- Security hardening beyond what the context requires
- Performance optimizations unless there's a clear bottleneck
- Additional abstractions or design patterns
- Type annotations or docstrings
- Error handling for impossible scenarios
- Anything listed under "Constraints / Non-Goals" in the plan

ONLY flag issues that would:
1. Cause runtime errors or crashes
2. Break existing functionality (check the actual code to verify)
3. Create data inconsistency or loss
4. Miss a required integration point (check imports, routes, configs)
5. Have incorrect assumptions about APIs, schemas, or data flow
6. Create race conditions or deadlocks
7. Violate constraints the plan doesn't account for
8. Miss a migration, env variable, or deployment step

Limit your response to the TOP 5 most critical findings. For each finding, cite the specific file and line number as evidence.

The plan to review follows below.

---

INSTRUCTIONS_EOF
cat "$PLAN_FILE" >> "$REQUEST_FILE"

# --- Call Codex (MEDIUM complexity shown — add model flags for LOW/HIGH) ---
cat "$REQUEST_FILE" | codex exec \
  --full-auto \
  --ephemeral \
  -C "$(pwd)" \
  -o "$OUTPUT_FILE" \
  -

# --- Read and display the output ---
echo "=== CODEX REVIEW OUTPUT ==="
cat "$OUTPUT_FILE"
echo "=== END OUTPUT ==="

# --- Cleanup ---
rm -f "$PLAN_FILE" "$REQUEST_FILE" "$OUTPUT_FILE"
```

Set the Bash tool timeout to `300000` (5 minutes) when running this command. If it times out, clean up orphaned files with: `rm -f /tmp/codex-plan-*.md /tmp/codex-request-*.md /tmp/codex-output-*.md`

**Important notes on the Codex call:**
- `--full-auto` prevents interactive prompts blocking the terminal
- `--ephemeral` avoids polluting session history with review artifacts
- `-C "$(pwd)"` gives Codex access to the project files so it can verify assumptions against actual code
- `-o` (`--output-last-message`) captures Codex's final response to a file for reliable reading
- The combined request is piped as a single stdin stream via `-` — instructions and plan in one document, avoiding the broken dual-stdin pattern of mixing pipes with heredocs
- The prompt explicitly tells Codex to filter itself — this is the first layer of noise reduction. Step 6 is the safety net because Codex doesn't always follow instructions perfectly
- The skill uses whatever model is configured in `~/.codex/config.toml` for MEDIUM, only overriding for LOW or HIGH complexity plans

**If Codex fails with an auth error:** Tell the user to run `codex login` or set `OPENAI_API_KEY`, then retry.

### Step 6: Filter — The Safety Net

Codex has a well-known tendency to:

- **Over-securitize**: Adding auth checks, input validation, and error handling everywhere, even for internal functions that receive trusted data. If the plan is for an internal tool or a prototype, most security suggestions are noise.
- **Over-abstract**: Suggesting interfaces, factories, and extra layers for things that don't need them yet. One concrete implementation beats a premature abstraction.
- **Scope creep**: Suggesting adjacent features, "while you're at it" additions, or "best practice" extras that expand the scope beyond what was asked.
- **Framework orthodoxy**: Insisting on patterns that a framework supports but the project doesn't use. If the codebase has its own conventions, those win.

**Your filter criteria — only keep findings that are:**

| Keep | Discard |
|------|---------|
| Will cause a runtime error | "Consider adding" suggestions |
| Breaks existing functionality | Additional error handling for unlikely cases |
| Data loss or corruption risk | Extra validation on trusted internal data |
| Missing required step (migration, env var, etc.) | Security hardening beyond the threat model |
| Wrong assumption about an API or schema | Architectural preferences/patterns |
| Race condition or state management bug | Performance suggestions without evidence |
| Forgotten dependency or import | Additional logging/monitoring |

### Step 7: Present Findings to the User

Present the filtered results in this format:

---

**Codex Plan Review — Blind Spot Report**

**Model used:** [model name + reasoning effort]
**Complexity assessed:** [LOW / MEDIUM / HIGH — with brief rationale]
**Plan reviewed:** [brief description]

**Critical Findings** (integrate these before executing):
1. [Finding] — [Why it matters and what to change] — *Evidence: `file:line`*
2. [Finding] — [Why it matters and what to change] — *Evidence: `file:line`*

**Noted but Non-Critical** (awareness only, no action needed):
- [Brief note if anything was borderline worth knowing]

**Verdict:** [Plan is solid / Plan needs adjustments before executing]

---

If Codex found nothing critical, say so clearly: "Codex review came back clean — no critical blind spots detected. Plan is good to execute."

### Step 8: Integrate and Proceed

If there were critical findings:
1. Update the plan to address each critical finding
2. Show the user what changed and why
3. Ask if they want to re-review the updated plan or proceed

If the plan was clean:
1. Proceed directly to execution
2. Note in the plan that it was cross-checked

## Edge Cases

**Codex is unavailable or errors out:**
- Preflight check (Step 0) catches installation and version issues early
- If `codex exec` fails with an auth error, tell the user to run `codex login` or set `OPENAI_API_KEY`
- If it errors for other reasons, show the error and ask if the user wants to proceed without review

**Codex takes too long:**
- The Bash tool timeout (300000ms / 5 minutes) will kill the process
- If it times out, report it and clean up: `rm -f /tmp/codex-plan-*.md /tmp/codex-request-*.md /tmp/codex-output-*.md`

**Codex returns only non-critical noise:**
- This is fine and expected. Report: "Codex review complete — no critical issues found. All suggestions were non-critical (security hardening, additional validation) and filtered out."

**User wants to use a different model:**
- Always respect explicit user model requests: `codex exec -m [model] ...`
- Common options: `o3`, `o4-mini`, `gpt-5.3-codex-spark`, `gpt-5.3-codex`
~~~

---

## Model Selection Guide

### The Two Axes

When deciding which Codex model to use, there are two independent choices:

**Axis 1: Spark vs Non-Spark**

| | Spark (`gpt-5.3-codex-spark`) | Non-Spark (`gpt-5.3-codex`) |
|---|---|---|
| **Optimized for** | Code generation, speed | Reasoning, analysis, architecture |
| **Latency** | Fast (seconds) | Slower (10-60s depending on reasoning) |
| **Cost** | Lower | Higher |
| **Best at** | "Read this code and find the bug" | "Think about whether this architecture will hold up" |
| **Weakness** | Can miss subtle cross-system issues | Overkill for simple tasks |

Analogy: **Spark = senior developer** (fast, pattern-matches, good instincts). **Non-spark = architect** (slower, thinks deeply, considers second-order effects).

**Axis 2: Reasoning Effort (low / medium / high)**

| Level | What it does | Use when |
|-------|-------------|----------|
| **low** | Minimal chain-of-thought. Quick pattern match. | Simple, low-risk changes. "Does this look right?" |
| **medium** | Moderate reasoning. Checks assumptions. | Default for most work. Good balance of speed and depth. |
| **high** | Deep reasoning. Explores edge cases, second-order effects. | High-risk changes: data models, auth, migrations, multi-system. |

### The Decision Matrix

```
                low reasoning    medium reasoning    high reasoning
              +----------------+------------------+------------------+
  Spark       | Quick sanity   | DEFAULT for most | Deep code review |
              | check on a     | plan reviews     | within a single  |
              | simple change  |                  | system           |
              +----------------+------------------+------------------+
  Non-Spark   | (rarely used)  | Cross-system     | Architecture     |
              |                | integration      | review, data     |
              |                | checks           | model changes,   |
              |                |                  | security-critical |
              +----------------+------------------+------------------+
```

### Practical Examples

| Task | Model | Reasoning | Why |
|------|-------|-----------|-----|
| Renaming a component and updating imports | Spark | Low | Mechanical change, just checking nothing was missed |
| Adding a new API endpoint | Spark | Medium | Needs to verify routes, types, integration points |
| Refactoring auth middleware | Non-Spark | High | Security-critical, cross-cutting, easy to break |
| Database migration + schema change | Non-Spark | High | Data loss risk, needs deep analysis of downstream impact |
| New React component with tests | Spark | Medium | Standard code task, pattern-matchable |
| Multi-service feature spanning 3 packages | Non-Spark | High | Cross-system coordination, integration risks |
| Updating CSS/Tailwind styles | Spark | Low | Visual only, no logic risk |
| Changing payment processing flow | Non-Spark | High | Financial data, compliance, cannot afford bugs |

### Auto-Selection: Let the AI Decide

The skill (v1.3.0) includes automatic model selection based on plan complexity. The orchestrating AI (Claude Code) reads the plan, assesses risk using the complexity signals table, and selects the appropriate model without asking the user.

**The principle:** The model doing the work shouldn't pick itself — the orchestrator that understands the full context should pick. Claude Code reads the plan, assesses complexity, and selects the model before handing off to Codex. Codex just does what it's told.

**Override:** Users can always force a specific model by saying "use o3", "use spark", etc. The skill respects explicit requests over auto-selection.

**Future:** If OpenAI adds a routing model (like a native `-m auto` flag), that would handle this at the CLI level. Until then, the skill is the router.

### Setting Your Default

In `~/.codex/config.toml`:

```toml
# Good default for most users — fast, capable, affordable
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
```

The skill overrides this only when the plan's complexity demands it. Most reviews will use your configured default.

---

## Installation

### Prerequisites

- **Claude Code** (Anthropic CLI)
- **Codex CLI** >= 0.100.0 (`npm install -g @openai/codex`)
- An **OpenAI API key** configured for Codex (`codex login` or set `OPENAI_API_KEY`)
- **macOS or Linux** (uses `/tmp/` for temp files; Windows users need WSL)

### Setup

```bash
# Create the skill directory
mkdir -p ~/.claude/skills/codex-plan-review

# Copy SKILL.md content from "The Skill" section above into:
# ~/.claude/skills/codex-plan-review/SKILL.md
```

### Verify

**Restart Claude Code** after adding the skill file (skills are discovered at startup). Then:

- Say "review my plan" or "second opinion" — the skill should activate automatically
- Or invoke directly with `/codex-plan-review`

If the skill doesn't trigger, check:
1. The file exists at `~/.claude/skills/codex-plan-review/SKILL.md`
2. Claude Code has been restarted since the file was created
3. The frontmatter `name:` and `description:` fields are intact

---

## Changelog

### v1.3.0 (2025-02-25)

**Fixed (critical):**
- **macOS `mktemp` with `.md` suffix creates literal filenames** — `mktemp /tmp/foo-XXXXXX.md` produces `/tmp/foo-XXXXXX.md` (the Xs aren't randomized when a suffix is present on macOS). Fixed by creating without extension then renaming: `mktemp /tmp/foo-XXXXXX && mv ...`
- **Shell state lost between steps** — Claude Code runs each bash command in a separate shell, so `$PLAN_FILE` etc. were empty in later steps. Fixed by merging Steps 3-5 into a single bash block that creates, uses, and cleans up temp files in one invocation.
- **Skill content drift between SKILL.md and shareable document** — embedded skill in the document could diverge from the actual installed file. Added sync comment and used tilde fences to cleanly embed without escaping issues.

**Fixed (important):**
- **LOW model routing** — table said "use spark" but the command didn't include `-m gpt-5.3-codex-spark`. Now explicit.
- **Version parse could silently succeed with empty string** — added hard fail when `grep` returns no semver match.
- **"Verify can authenticate" claim was misleading** — preflight only checks install/version. Softened to accurate description; auth errors are caught when `codex exec` runs.
- **`trap` cleanup didn't fire** — removed `trap` approach entirely; cleanup is explicit in the single bash block with a pattern-based fallback for timeouts.

**Added:**
- HTML comment in shareable doc marking the skill section as needing sync with SKILL.md
- Troubleshooting checklist in Installation section

### v1.2.0 (2025-02-25)

**Fixed (critical):**
- **Broken stdin piping** — v1.1.0 used `cat "$PLAN_FILE" | codex exec ... - << 'PROMPT_EOF'` which mixed pipe and heredoc on the same stdin. In bash the heredoc wins and the plan is silently dropped; in zsh behavior is unpredictable. This could produce false "clean" verdicts where Codex never saw the plan. Fixed by building a single combined request file.

**Fixed (important):**
- Preflight version check now does real semver comparison via `sort -V`
- Added auth error recovery guidance
- Removed macOS-incompatible `timeout` command
- All three complexity variants show complete commands (no `[same prompt as above]` placeholders)

**Added:**
- Platform note (macOS/Linux, Windows needs WSL)
- Model name evolution note
- Restart requirement in installation instructions

### v1.1.0 (2025-02-25)

**Added:**
- Automatic model selection based on plan complexity
- Preflight version check
- Privacy note about project file access
- Plan template with "Key Files to Examine" and "Constraints / Non-Goals"
- Two-layer filter explanation
- Top 5 findings cap with file:line evidence requirement
- Version field in frontmatter

**Fixed:**
- Replaced fixed `/tmp/` paths with `mktemp`
- Removed hardcoded model references

### v1.0.0 (2025-02-24)

- Initial release
