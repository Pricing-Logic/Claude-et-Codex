# Codex Model Selection Guide

How to choose the right Codex model, reasoning level, and multi-agent concurrency for plan reviews — and why the skill does it automatically.

## The Three Axes (v2.0)

### Axis 1: Spark vs Non-Spark

| | Spark (`gpt-5.3-codex-spark`) | Non-Spark (`gpt-5.3-codex`) |
|---|---|---|
| **Optimized for** | Code generation, speed | Reasoning, analysis, architecture |
| **Latency** | Fast (seconds) | Slower (10-60s depending on reasoning) |
| **Cost** | Lower | Higher |
| **Best at** | "Read this code and find the bug" | "Think about whether this architecture will hold up" |
| **Weakness** | Can miss subtle cross-system issues | Overkill for simple tasks |

**Spark = senior developer.** Fast, pattern-matches, good instincts. Reads code and spots bugs quickly.

**Non-spark = architect.** Slower, thinks deeply, considers second-order effects. Reasons about whether the whole design holds together.

### Axis 2: Reasoning Effort

| Level | What it does | Use when |
|-------|-------------|----------|
| **low** | Minimal chain-of-thought. Quick pattern match. | Simple, low-risk changes. "Does this look right?" |
| **medium** | Moderate reasoning. Checks assumptions. | Default for most work. Good balance of speed and depth. |
| **high** | Deep reasoning. Explores edge cases, second-order effects. | High-risk changes: data models, auth, migrations, multi-system. |
| **xhigh** | Maximum reasoning depth. Full exploration of alternatives. | Critical changes: data model redesigns, security-critical, cross-service migrations. |

### Axis 3: Multi-Agent Concurrency

| Threads | What it does | Use when |
|---------|-------------|----------|
| **2** | Explorer + Architecture reviewer | LOW complexity — basic two-agent review |
| **3** | 2 + headroom for focused follow-up | MEDIUM complexity — standard multi-file changes |
| **4** | Maximum parallel exploration | HIGH complexity — large-scale or cross-system changes |

**`max_depth=1`** is always set — sub-agents cannot spawn their own sub-agents. This is a safety guard to prevent runaway agent trees.

## The Decision Matrix

```
                medium reasoning    high reasoning      xhigh reasoning
              +------------------+------------------+------------------+
  Spark       | LOW complexity   |                  |                  |
  2 threads   | Simple changes,  |   (not used —    |   (not used —    |
              | config, styling  | upgrade to non-  | upgrade to non-  |
              |                  |    spark)         |    spark)        |
              +------------------+------------------+------------------+
  Non-Spark   |                  | MEDIUM complex.  | HIGH complexity  |
  3-4 threads |  (not used —     | New endpoints,   | Data models,     |
              | downgrade to     | logic changes,   | auth, migrations |
              |    spark)        | refactors        | cross-system     |
              +------------------+------------------+------------------+
```

## Practical Examples

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

## How the Skill Auto-Selects

The skill reads your plan and maps it to complexity using four signals:

| Signal | LOW | MEDIUM | HIGH |
|--------|-----|--------|------|
| Files changed | 1-3 | 4-8 | 9+ |
| Scope | UI/styling, copy, config | New endpoints, logic changes, refactors | Data model, auth, migrations, cross-system |
| Risk | Reversible, no data impact | Could break features | Data loss, security, breaking changes |
| Dependencies | Self-contained | Touches shared code | Cross-package, external APIs |

Then selects:

| Complexity | Model | Reasoning | Threads | Timeout | Codex Flags |
|------------|-------|-----------|---------|---------|-------------|
| LOW | Spark | Medium | 2 | 600s | `-m gpt-5.3-codex-spark -c model_reasoning_effort="medium" -c agents.max_threads=2 -c agents.max_depth=1 -c agents.job_max_runtime_seconds=600` |
| MEDIUM | Codex-5.3 | High | 3 | 900s | `-m gpt-5.3-codex -c model_reasoning_effort="high" -c agents.max_threads=3 -c agents.max_depth=1 -c agents.job_max_runtime_seconds=900` |
| HIGH | Codex-5.3 | XHigh | 4 | 1200s | `-m gpt-5.3-codex -c model_reasoning_effort="xhigh" -c agents.max_threads=4 -c agents.max_depth=1 -c agents.job_max_runtime_seconds=1200` |

**Auto-escalation triggers:** If a LOW plan touches auth/permissions, schema/data contracts, migrations, background jobs, concurrency, external API contracts, or unclear ownership boundaries, it auto-upgrades to MEDIUM routing.

**The principle:** The model doing the work shouldn't pick itself — the orchestrator (Claude Code) that understands the full context should pick. Claude Code reads the plan, assesses complexity, selects the model and concurrency, then hands off to Codex. Codex just does what it's told.

## Override

You can always force a specific model:

- "use o3" — uses OpenAI's o3 model
- "use spark" — forces the fast Spark model regardless of complexity
- "use gpt-5.3-codex with xhigh reasoning" — explicit model + effort
- "skip multi-agent" or "no sub-agents" — forces single-shot mode
- "4 threads" — overrides thread count

The skill always respects explicit user requests over auto-selection.

## A Note on Model Names

Model names change as OpenAI releases new versions. The current names (`gpt-5.3-codex-spark`, `gpt-5.3-codex`) are accurate as of March 2026. If they stop working, check the [Codex CLI docs](https://github.com/openai/codex) for current model IDs.

## Multi-Agent vs Single-Shot

| | Single-Shot (v1.x) | Multi-Agent (v2.0) |
|---|---|---|
| **Review depth** | One agent does everything | Specialized agents focus on their strengths |
| **Code validation** | Reads some files | Explorer sub-agent systematically reads all relevant files |
| **Architecture analysis** | Mixed in with code reading | Dedicated sub-agent reasons about design |
| **Latency** | 30-120s | 60-300s (more work, but parallel) |
| **Cost** | Lower | Higher (more tokens across agents) |
| **Fallback** | N/A | Automatic — retries single-shot if multi-agent fails |

Multi-agent mode is the default in v2.0 because the quality improvement is significant, especially for MEDIUM and HIGH complexity plans. For LOW complexity plans, the overhead is minimal (2 threads, shorter timeout).
