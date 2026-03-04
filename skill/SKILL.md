---
name: codex-plan-review
version: 1.5.0
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
# Ensure npm/node global bins are on PATH (Claude Code bash may not inherit full shell profile)
for p in "$HOME/.local/bin" "$HOME/.npm-global/bin" "/usr/local/bin"; do
  [ -d "$p" ] && case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done
for p in $(find "$HOME/.nvm/versions/node" -maxdepth 2 -name bin -type d 2>/dev/null); do
  case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done
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
| LOW | spark | medium | `-m gpt-5.3-codex-spark -c model_reasoning_effort="medium"` |
| MEDIUM | codex-5.3 | high | `-m gpt-5.3-codex -c model_reasoning_effort="high"` |
| HIGH | codex-5.3 | xhigh | `-m gpt-5.3-codex -c model_reasoning_effort="xhigh"` |

Rationale: Spark handles simple checklist-style reviews well. Codex-5.3 reasons more carefully about architecture and edge cases — use it for anything beyond trivial changes. Always be explicit about model selection; never defer to user config defaults for deterministic review quality.

**Auto-escalation triggers:** If a LOW plan touches any of these, auto-upgrade to MEDIUM routing: auth/permissions, schema/data contracts, migrations, background jobs, concurrency, external API contracts, or unclear ownership boundaries.

**Note:** Model names evolve. If `gpt-5.3-codex-spark` or `gpt-5.3-codex` are no longer available, check `codex` docs for current model IDs and substitute accordingly.

If the user explicitly requests a model or reasoning effort (e.g., "use o3", "use spark", "xhigh thinking", "high reasoning"), always respect that over the auto-selection. Valid reasoning effort values: `low`, `medium`, `high`, `xhigh`.

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
- **LOW**: add `-m gpt-5.3-codex-spark -c model_reasoning_effort="medium"` to the `codex exec` line
- **MEDIUM**: add `-m gpt-5.3-codex -c model_reasoning_effort="high"` to the `codex exec` line
- **HIGH**: add `-m gpt-5.3-codex -c model_reasoning_effort="xhigh"` to the `codex exec` line

```bash
# --- Ensure npm/node global bins are on PATH ---
for p in "$HOME/.local/bin" "$HOME/.npm-global/bin" "/usr/local/bin"; do
  [ -d "$p" ] && case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done
for p in $(find "$HOME/.nvm/versions/node" -maxdepth 2 -name bin -type d 2>/dev/null); do
  case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done

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

# --- Call Codex (add model flags from Step 1 assessment) ---
cat "$REQUEST_FILE" | codex exec \
  --full-auto \
  --ephemeral \
  -m gpt-5.3-codex \
  -c model_reasoning_effort="high" \
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
- The skill always explicitly selects a model — never defers to user config defaults — for deterministic review quality

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
