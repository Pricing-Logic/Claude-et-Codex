# Codex Plan Review — Claude Code Skill

> **Version:** 2.0.0
> **Purpose:** Cross-check implementation plans against OpenAI's Codex CLI using multi-agent review to catch blind spots before coding
> **Platform:** Claude Code (Anthropic CLI) orchestrating Codex CLI (OpenAI) with parallel sub-agents
> **Requirements:** Codex CLI >= 0.104.0, Claude Code, macOS/Linux/Windows (Git Bash)

---

## Table of Contents

1. [Overview](#overview)
2. [Multi-Agent Architecture](#multi-agent-architecture)
3. [The Skill (SKILL.md)](#the-skill)
4. [Model Selection Guide](#model-selection-guide)
5. [Installation](#installation)
6. [Configuration](#configuration)
7. [Changelog](#changelog)

---

## Overview

This skill lets Claude Code call OpenAI's Codex CLI as a second reviewer before executing implementation plans. The two AIs think differently — Claude tends toward pragmatism, Codex tends toward thoroughness — so cross-checking catches blind spots neither would find alone.

**v2.0** upgrades from single-shot review to **multi-agent review** using Codex's `multi_agent` feature. Instead of one Codex agent doing everything, the skill spawns two parallel sub-agents — an explorer that validates code reality, and an architecture reviewer that evaluates design and risk — then synthesizes both into a single verdict with structured categories.

The key insight: Codex over-engineers security and adds unnecessary complexity. The skill handles this by applying a two-layer filter — the prompt pre-filters Codex's output, then Claude Code applies a safety net filter before presenting findings to the user. With multi-agent mode, there's also a deduplication step to catch when both sub-agents flag the same issue.

### What It Catches

- Runtime errors and crashes the plan would introduce
- Breaking changes to existing functionality
- Data loss or corruption risks
- Missing integration points (imports, routes, configs)
- Wrong assumptions about APIs, schemas, or data flow
- Race conditions and state management bugs
- Forgotten migrations, env variables, or deployment steps
- Mismatches between the plan and actual code (explorer sub-agent)
- Architectural risks, sequencing problems, rollback gaps (architecture sub-agent)

### What It Filters Out

- "Consider adding" suggestions
- Extra error handling for impossible scenarios
- Security hardening beyond the threat model
- Architectural preferences and pattern suggestions
- Performance optimizations without evidence
- Additional logging and monitoring
- Duplicate findings from both sub-agents

---

## Multi-Agent Architecture

### How It Works

```
Claude Code (orchestrator)
    |
    v
Codex Parent Agent (coordinator)
    |
    +---> Explorer Sub-Agent (parallel)
    |     - Reads Key Files to Examine
    |     - Reads Files to Modify
    |     - Validates imports, deps, APIs
    |     - Output: code-level mismatches
    |
    +---> Architecture Sub-Agent (parallel)
    |     - Evaluates design boundaries
    |     - Checks sequencing, migration, rollback
    |     - Verifies data consistency, race conditions
    |     - Output: architectural risks
    |
    v
Codex Parent Agent merges findings
    |
    v
Claude Code filters noise + deduplicates
    |
    v
User gets: structured verdict with evidence
```

### Why Multi-Agent

Single-shot reviews have a fundamental limitation: one agent must context-switch between reading code and reasoning about architecture. Multi-agent mode eliminates this by letting each sub-agent focus on what it does best:

- **Explorer** is fast, file-focused, and concrete — it reads actual code and reports mismatches
- **Architecture reviewer** is slower, analytical, and abstract — it reasons about design, sequencing, and risk
- **Parent agent** reconciles conflicting findings and produces a coherent verdict

This produces significantly deeper reviews, especially for MEDIUM and HIGH complexity plans.

### Concurrency Controls

| Complexity | Max Threads | Max Depth | Per-Worker Timeout |
|------------|-------------|-----------|-------------------|
| LOW | 2 | 1 | 600s (10 min) |
| MEDIUM | 3 | 1 | 900s (15 min) |
| HIGH | 4 | 1 | 1200s (20 min) |

- `max_depth=1` prevents sub-agents from spawning their own sub-agents (safety guard)
- Thread count scales with complexity — higher-complexity plans benefit from more parallel exploration
- Per-worker timeout prevents runaway agents while giving enough time for thorough file reading

### Fallback

If multi-agent mode fails (older Codex version, feature disabled, runtime error), the skill automatically retries in single-shot mode with the same prompt. Codex processes the instructions linearly instead of spawning sub-agents. The user is informed which mode was used.

### Sandbox Policy

The skill uses `--sandbox read-only` — Codex can read all project files but cannot write, execute builds, or run tests. This is appropriate for plan review. **Important:** Do NOT use `--full-auto` with `--sandbox read-only` — `--full-auto` silently overrides the sandbox to `workspace-write`. In exec mode, approval defaults to `never`, so `--full-auto` is unnecessary. If you need Codex to run build/test commands during review, switch to `--sandbox workspace-write`.

---

## The Skill

Copy the content below into `~/.claude/skills/codex-plan-review/SKILL.md`. After creating or updating the file, **restart Claude Code** so it discovers the new skill.

<!-- IMPORTANT: This section must be kept in sync with skill/SKILL.md.
     Source of truth: skill/SKILL.md in this repository -->

The full skill is in [`skill/SKILL.md`](../skill/SKILL.md). Copy it directly:

```bash
mkdir -p ~/.claude/skills/codex-plan-review
cp skill/SKILL.md ~/.claude/skills/codex-plan-review/SKILL.md
```

---

## Model Selection Guide

### The Three Axes (v2.0)

v2.0 adds a third axis to model selection: multi-agent concurrency.

**Axis 1: Spark vs Non-Spark**

| | Spark (`gpt-5.3-codex-spark`) | Non-Spark (`gpt-5.3-codex`) |
|---|---|---|
| **Optimized for** | Code generation, speed | Reasoning, analysis, architecture |
| **Latency** | Fast (seconds) | Slower (10-60s depending on reasoning) |
| **Cost** | Lower | Higher |
| **Best at** | "Read this code and find the bug" | "Think about whether this architecture will hold up" |
| **Weakness** | Can miss subtle cross-system issues | Overkill for simple tasks |

Analogy: **Spark = senior developer** (fast, pattern-matches, good instincts). **Non-spark = architect** (slower, thinks deeply, considers second-order effects).

**Axis 2: Reasoning Effort (low / medium / high / xhigh)**

| Level | What it does | Use when |
|-------|-------------|----------|
| **low** | Minimal chain-of-thought. Quick pattern match. | Simple, low-risk changes. "Does this look right?" |
| **medium** | Moderate reasoning. Checks assumptions. | Default for most work. Good balance of speed and depth. |
| **high** | Deep reasoning. Explores edge cases, second-order effects. | High-risk changes: data models, auth, migrations, multi-system. |
| **xhigh** | Maximum reasoning depth. Full exploration of alternatives. | Critical changes: data model redesigns, security-critical systems, cross-service migrations. |

**Axis 3: Multi-Agent Concurrency**

| Threads | What it does | Use when |
|---------|-------------|----------|
| **2** | Explorer + Architecture reviewer | LOW complexity — sufficient for simple plans |
| **3** | 2 + headroom for additional focused agent | MEDIUM complexity — standard multi-file changes |
| **4** | Maximum parallel exploration | HIGH complexity — large-scale or cross-system changes |

### The Decision Matrix (v2.0)

| Complexity | Model | Reasoning | Threads | Flags |
|------------|-------|-----------|---------|-------|
| LOW | Spark | Medium | 2 | `-m gpt-5.3-codex-spark -c model_reasoning_effort="medium" -c agents.max_threads=2` |
| MEDIUM | Codex-5.3 | High | 3 | `-m gpt-5.3-codex -c model_reasoning_effort="high" -c agents.max_threads=3` |
| HIGH | Codex-5.3 | XHigh | 4 | `-m gpt-5.3-codex -c model_reasoning_effort="xhigh" -c agents.max_threads=4` |

All tiers now use explicit model selection — the skill never defers to user config defaults for deterministic review quality.

### Practical Examples

| Task | Model | Reasoning | Threads | Why |
|------|-------|-----------|---------|-----|
| Renaming a component and updating imports | Spark | Medium | 2 | Mechanical change, just checking nothing was missed |
| Adding a new API endpoint | Codex-5.3 | High | 3 | Needs to verify routes, types, integration points |
| Refactoring auth middleware | Codex-5.3 | XHigh | 4 | Security-critical, cross-cutting, easy to break |
| Database migration + schema change | Codex-5.3 | XHigh | 4 | Data loss risk, needs deep analysis of downstream impact |
| New React component with tests | Spark | Medium | 2 | Standard code task, pattern-matchable |
| Multi-service feature spanning 3 packages | Codex-5.3 | XHigh | 4 | Cross-system coordination, integration risks |
| Updating CSS/Tailwind styles | Spark | Medium | 2 | Visual only, no logic risk |
| Changing payment processing flow | Codex-5.3 | XHigh | 4 | Financial data, compliance, cannot afford bugs |

### Auto-Selection

The skill (v2.0.0) includes automatic model and concurrency selection based on plan complexity. Claude Code reads the plan, assesses risk using the complexity signals table, and selects the appropriate model, reasoning effort, and thread count without asking the user.

**The principle:** The model doing the work shouldn't pick itself — the orchestrator that understands the full context should pick. Claude Code reads the plan, assesses complexity, selects the configuration, then hands off to Codex. Codex just does what it's told.

**Override:** Users can always force a specific model by saying "use o3", "use spark", etc. The skill respects explicit requests over auto-selection. Users can also request single-shot mode with "skip multi-agent" or "no sub-agents".

### A Note on Model Names

Model names change as OpenAI releases new versions. The current names (`gpt-5.3-codex-spark`, `gpt-5.3-codex`) are accurate as of March 2026. If they stop working, check the [Codex CLI docs](https://github.com/openai/codex) for current model IDs.

---

## Installation

### Prerequisites

- **Claude Code** (Anthropic CLI)
- **Codex CLI** >= 0.104.0 (`npm install -g @openai/codex`)
- An **OpenAI API key** configured for Codex (`codex login` or set `OPENAI_API_KEY`)
- **macOS, Linux, or Windows** (Git Bash on Windows 11; temp files use project-relative dirs, no WSL needed)

### Setup

```bash
# Clone the repo
git clone https://github.com/Pricing-Logic/Claude-et-Codex.git

# Create the skill directory and copy
mkdir -p ~/.claude/skills/codex-plan-review
cp Claude-et-Codex/skill/SKILL.md ~/.claude/skills/codex-plan-review/SKILL.md
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

## Configuration

### Enabling Multi-Agent in Codex

The `multi_agent` feature must be enabled in Codex. Add to `~/.codex/config.toml`:

```toml
[features]
multi_agent = true
```

Or enable per-call with `--enable multi_agent` (the skill does this automatically).

**Note:** If your config still has `[collab]`, rename it to `[features]` with `multi_agent = true`. The `collab` name is deprecated.

### Recommended Codex Config

```toml
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "high"

[features]
multi_agent = true
```

The skill overrides the model and reasoning effort per-review based on plan complexity, but these are good defaults for other Codex usage.

---

## Changelog

### v2.0.1 (2026-03-04)

**Fixed:**
- **Windows /tmp path mismatch** — On Windows (Git Bash), `/tmp` resolves to different physical locations depending on the tool (Write tool, Node.js, bash). Replaced all `/tmp/` temp file usage with a project-relative directory (`$(pwd)/.codex-tmp-$$`), which all tools resolve consistently. Native Git Bash on Windows 11 is now fully supported — no WSL required.
- **mkdir fail-fast** — Added explicit error check after `mkdir -p "$WORK_DIR"` so failures in read-only or locked directories fail immediately with a clear error rather than silently continuing with missing artifacts.
- **Stale timeout cleanup hint** — The inline note after the bash block still referenced `/tmp/codex-*` cleanup; updated to `rm -rf "$(pwd)"/.codex-tmp-*`.

### v2.0.0 (2026-03-04)

**Major: Multi-Agent Review**
- **Multi-agent review mode** — Codex now spawns parallel sub-agents (explorer + architecture reviewer) for deeper reviews. The parent agent synthesizes findings into a single structured verdict.
- **Structured verdict format** — Output now uses APPROVE / APPROVE_WITH_CHANGES / BLOCK verdicts with categorized findings (Critical, Medium Risks, Missing Steps, Assumptions).
- **Concurrency controls** — `max_threads`, `max_depth`, and `job_max_runtime_seconds` are configured per complexity tier. LOW=2 threads, MEDIUM=3, HIGH=4.
- **Read-only sandbox** — Reviews now use `--sandbox read-only` for security. Codex can read all project files but cannot write or execute.
- **Automatic fallback** — If multi_agent fails (older version, feature disabled, runtime error), the skill retries in single-shot mode automatically.
- **Sub-agent deduplication** — Filter step now catches when both sub-agents flag the same issue from different angles.
- **Legacy collab migration note** — Documents the `collab` → `multi_agent` deprecation for users with old configs.

**Changed:**
- Minimum Codex version bumped from 0.100.0 to 0.104.0
- Bash tool timeout increased from 300s (5 min) to 600s (10 min) for multi-agent reviews
- All complexity tiers now use explicit model selection (LOW was previously deferred to config defaults)
- Model routing updated: LOW=spark/medium, MEDIUM=codex-5.3/high, HIGH=codex-5.3/xhigh
- Preflight check now also verifies `multi_agent` feature status
- Report format updated to show review mode (multi-agent vs single-shot fallback)

### v1.5.0 (2025-03)

**Fixed:**
- **Zsh glob bug with nvm path detection** — `find` in PATH setup could fail under zsh with nomatch option. Added proper quoting.
- **PATH setup blocks** — Added explicit PATH setup for npm/node global bins at the start of both preflight and main bash blocks, since Claude Code bash may not inherit full shell profile.

**Changed:**
- Model routing upgraded: all tiers now explicit (never defers to config defaults)
- LOW: spark/medium, MEDIUM: codex-5.3/high, HIGH: codex-5.3/xhigh
- Auto-escalation triggers added: auth/permissions, schema/data contracts, migrations, background jobs, concurrency, external API contracts, unclear ownership boundaries

### v1.3.0 (2025-02-25)

**Fixed (critical):**
- **macOS `mktemp` with `.md` suffix creates literal filenames** — Fixed by creating without extension then renaming.
- **Shell state lost between steps** — Fixed by merging Steps 3-5 into a single bash block.
- **Skill content drift** — Added sync comment and used tilde fences for clean embedding.

**Fixed (important):**
- LOW model routing now explicit
- Version parse hard fails on empty string
- Removed misleading "verify can authenticate" claim
- Removed `trap` approach; cleanup is explicit

### v1.2.0 (2025-02-25)

**Fixed (critical):**
- **Broken stdin piping** — Mixed pipe and heredoc silently dropped the plan. Fixed with combined request file.

### v1.1.0 (2025-02-25)

**Added:**
- Automatic model selection, preflight check, privacy note, plan template, two-layer filter, Top 5 cap

### v1.0.0 (2025-02-24)

- Initial release
