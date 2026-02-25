# Codex Model Selection Guide

How to choose the right Codex model and reasoning level for plan reviews — and why the skill does it automatically.

## The Two Axes

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

## The Decision Matrix

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

## Practical Examples

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

## How the Skill Auto-Selects

The skill reads your plan and maps it to complexity using four signals:

| Signal | LOW | MEDIUM | HIGH |
|--------|-----|--------|------|
| Files changed | 1-3 | 4-8 | 9+ |
| Scope | UI/styling, copy, config | New endpoints, logic changes, refactors | Data model, auth, migrations, cross-system |
| Risk | Reversible, no data impact | Could break features | Data loss, security, breaking changes |
| Dependencies | Self-contained | Touches shared code | Cross-package, external APIs |

Then selects:

| Complexity | Model | Reasoning | Codex Flags |
|------------|-------|-----------|-------------|
| LOW | Spark | Low | `-m gpt-5.3-codex-spark -c model_reasoning_effort="low"` |
| MEDIUM | Config default | Config default | *(none — uses ~/.codex/config.toml)* |
| HIGH | Non-Spark | High | `-m gpt-5.3-codex -c model_reasoning_effort="high"` |

**The principle:** The model doing the work shouldn't pick itself — the orchestrator (Claude Code) that understands the full context should pick. Claude Code reads the plan, assesses complexity, selects the model, then hands off to Codex. Codex just does what it's told.

## Override

You can always force a specific model:

- "use o3" — uses OpenAI's o3 model
- "use spark" — forces the fast Spark model regardless of complexity
- "use gpt-5.3-codex with high reasoning" — explicit model + effort

The skill always respects explicit user requests over auto-selection.

## Setting Your Default

In `~/.codex/config.toml`:

```toml
# Good default for most users
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
```

The skill only overrides this for LOW or HIGH complexity — MEDIUM uses your config as-is.

## A Note on Model Names

Model names change as OpenAI releases new versions. The current names (`gpt-5.3-codex-spark`, `gpt-5.3-codex`) are accurate as of February 2025. If they stop working, check the [Codex CLI docs](https://github.com/openai/codex) for current model IDs.

## Future: Native Auto-Routing

If OpenAI adds a routing/auto model (like `-m auto`), that would handle model selection at the CLI level natively. Until then, the skill acts as the router — it's the same architecture, just implemented one layer up.
