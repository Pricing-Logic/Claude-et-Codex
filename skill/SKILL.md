---
name: codex-plan-review
version: 2.0.0
description: "Cross-check implementation plans against Codex CLI using multi-agent review to catch blind spots before coding. Use this skill BEFORE executing any implementation plan — when you've drafted a plan for a new feature, a refactor, a bug fix, or any multi-file change. Also use when the user says things like 'plan review', 'check with codex', 'second opinion', 'blind spot check', 'review my plan', or 'cross-check this'. The whole point is to get a second AI perspective on architectural decisions and catch things Claude Code might miss, then filter the feedback to only what actually matters. v2.0 uses Codex's multi_agent feature to spawn parallel sub-agents for deeper code-reality validation and architecture review."
---

# Codex Plan Review — Multi-Agent Blind Spot Detection

You have a plan (or are about to finalize one). Before executing it, you're going to get a second opinion from OpenAI's Codex CLI to catch blind spots. Codex thinks differently than you do — it'll spot things you miss. But it also tends to over-engineer security and add unnecessary complexity. Your job is to be a smart filter: extract the critical insights, discard the noise.

**v2.0** uses Codex's `multi_agent` feature to spawn parallel sub-agents — an **explorer** that reads actual repo files to validate feasibility, and an **architecture reviewer** that evaluates design, sequencing, and risk — then synthesizes both into a single verdict. This produces significantly deeper reviews than single-shot mode.

**Requirements:** Codex CLI >= 0.104.0 (`codex -V` to check). Install with `npm install -g @openai/codex` if missing. The `multi_agent` feature must be available (experimental as of March 2026).

**Privacy note:** This skill gives Codex read-only access to project files in the working directory via `-C` and `--sandbox read-only`. For sensitive or proprietary repos, confirm with the user before proceeding.

**Platform:** macOS, Linux, and Windows (including Git Bash on Windows 11). Temp files are written to a project-relative directory — never `/tmp/` — so all tools resolve the same path.

## When This Triggers

- You've just written or finalized an implementation plan
- The user asks for a "plan review", "second opinion", or "blind spot check"
- Before executing a multi-step plan that touches 3+ files
- The user explicitly invokes `/codex-plan-review`

## The Workflow

### Step 0: Preflight Check

Before anything else, verify Codex is installed, meets the minimum version, and has multi_agent available:

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
MIN_VERSION="0.104.0"
if [ "$(printf '%s\n' "$MIN_VERSION" "$CODEX_VERSION" | sort -V | head -n1)" != "$MIN_VERSION" ]; then
  echo "WARNING: Codex CLI $CODEX_VERSION is below minimum $MIN_VERSION — multi_agent may not be available. Will attempt single-shot fallback."
fi
# Check if multi_agent feature is available
MULTI_AGENT_STATUS=$(codex features list 2>/dev/null | grep -E "^multi_agent" | awk '{print $NF}')
echo "multi_agent feature: ${MULTI_AGENT_STATUS:-unknown}"
```

If `codex exec` later fails with an authentication error, tell the user to run `codex login` or set the `OPENAI_API_KEY` environment variable.

### Step 1: Assess Complexity & Select Configuration

Before calling Codex, assess the plan's complexity to determine model, reasoning effort, and multi-agent concurrency. This should happen automatically — don't ask the user unless they've expressed a preference.

**Complexity signals:**

| Signal | LOW | MEDIUM | HIGH |
|--------|-----|--------|------|
| Files changed | 1-3 | 4-8 | 9+ |
| Scope | UI/styling, copy, config | New endpoints, logic changes, refactors | Data model, auth, migrations, cross-system |
| Risk | Reversible, no data impact | Could break features | Data loss, security, breaking changes |
| Dependencies | Self-contained | Touches shared code | Cross-package, external APIs |

**Model and multi-agent configuration by complexity:**

| Complexity | Model | Reasoning | Max Threads | Max Depth | Timeout (s) |
|------------|-------|-----------|-------------|-----------|-------------|
| LOW | spark | medium | 2 | 1 | 600 |
| MEDIUM | codex-5.3 | high | 3 | 1 | 900 |
| HIGH | codex-5.3 | xhigh | 4 | 1 | 1200 |

Rationale: Spark handles simple checklist-style reviews well. Codex-5.3 reasons more carefully about architecture and edge cases — use it for anything beyond trivial changes. `max_depth=1` prevents recursive sub-agent spawning (a safety guard). Thread count scales with complexity because higher-complexity plans benefit from more parallel exploration.

**Auto-escalation triggers:** If a LOW plan touches any of these, auto-upgrade to MEDIUM routing: auth/permissions, schema/data contracts, migrations, background jobs, concurrency, external API contracts, or unclear ownership boundaries.

**Note:** Model names evolve. If `gpt-5.3-codex-spark` or `gpt-5.3-codex` are no longer available, check `codex` docs for current model IDs and substitute accordingly.

If the user explicitly requests a model or reasoning effort (e.g., "use o3", "use spark", "xhigh thinking", "high reasoning"), always respect that over the auto-selection. Valid reasoning effort values: `low`, `medium`, `high`, `xhigh`.

### Step 2: Prepare the Plan Document

Assemble a clear plan document. If you haven't already written one, draft one now using this minimum template.

**Windows note:** Do NOT use the Write tool to write the plan to `/tmp/` — on Windows, the Write tool, Node.js, and bash all resolve `/tmp` to different physical locations. The plan content must be written inline inside the bash heredoc in Step 3 (shown below). Never write temp files to `/tmp/` paths when running on Windows.

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

### Step 2.5: Dependency Impact Analysis

Before building the Codex request, trace dependency relationships for files mentioned in the plan. This gives Codex concrete blast-radius topology data instead of inferring it. Non-blocking — any failure skips with an inline note.

The dep-graph code runs inside the Steps 3-5 bash block below, after `$WORK_DIR` and `$PLAN_FILE` are defined and the plan is written. Variables `$DEP_GRAPH_FILE` and `$DEP_GRAPH_AVAILABLE` are initialised at the top of that block and used in injection steps after the plan is appended.

### Steps 3-5: Build Request, Call Codex (Multi-Agent), Read Output

These steps MUST run as a single bash command because Claude Code runs each command in a separate shell — variables like `$PLAN_FILE` would be lost between steps. Run the entire block below as one command.

Choose the configuration based on your Step 1 assessment:

**LOW complexity:**
```
-m gpt-5.3-codex-spark -c model_reasoning_effort="medium" -c agents.max_threads=2 -c agents.max_depth=1 -c agents.job_max_runtime_seconds=600
```

**MEDIUM complexity:**
```
-m gpt-5.3-codex -c model_reasoning_effort="high" -c agents.max_threads=3 -c agents.max_depth=1 -c agents.job_max_runtime_seconds=900
```

**HIGH complexity:**
```
-m gpt-5.3-codex -c model_reasoning_effort="xhigh" -c agents.max_threads=4 -c agents.max_depth=1 -c agents.job_max_runtime_seconds=1200
```

```bash
# --- Ensure npm/node global bins are on PATH ---
for p in "$HOME/.local/bin" "$HOME/.npm-global/bin" "/usr/local/bin"; do
  [ -d "$p" ] && case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done
for p in $(find "$HOME/.nvm/versions/node" -maxdepth 2 -name bin -type d 2>/dev/null); do
  case ":$PATH:" in *":$p:"*) ;; *) export PATH="$p:$PATH" ;; esac
done

# --- Create unique temp directory inside the project (works on macOS, Linux, Windows/Git Bash) ---
# Using $(pwd) avoids /tmp path mismatches between Write tool, Node.js, and bash on Windows.
WORK_DIR="$(pwd)/.codex-tmp-$$"
mkdir -p "$WORK_DIR" || { echo "ERROR: Cannot create temp dir $WORK_DIR (read-only or locked directory?)"; exit 1; }
PLAN_FILE="$WORK_DIR/plan.md"
REQUEST_FILE="$WORK_DIR/request.md"
OUTPUT_FILE="$WORK_DIR/output.md"
DEP_GRAPH_FILE="$WORK_DIR/dep-graph.md"
DEP_GRAPH_AVAILABLE=false

# --- Write the plan ---
cat > "$PLAN_FILE" << 'PLAN_EOF'
[Your assembled plan goes here]
PLAN_EOF

# --- Step 2.5: Dependency Impact Analysis ---
# Traces import/consumer relationships for plan files. Non-blocking — failures skip gracefully.
{
PROJ_LANG="unknown"
[ -f "package.json" ] && PROJ_LANG="js"
[ -f "pyproject.toml" ] && PROJ_LANG="python"
[ -f "requirements.txt" ] && PROJ_LANG="python"
[ -f "go.mod" ] && PROJ_LANG="go"
[ -f "Cargo.toml" ] && PROJ_LANG="rust"

# || true: prevents set -e from triggering when last [ -f ] returns false
PLAN_FILES_DEP=$(grep -oE '[a-zA-Z0-9/_.-]+\.(ts|tsx|js|jsx|py|go|rs|rb|java)' "$PLAN_FILE" 2>/dev/null \
  | sort -u | while IFS= read -r f; do [ -f "$f" ] && echo "$f"; done || true)

if [ -z "$PLAN_FILES_DEP" ]; then
  echo "Step 2.5: No code file paths found in plan — skipping dependency analysis."
else
  DEP_GRAPH_AVAILABLE=true
  printf "## Dependency Impact Graph\n\n" > "$DEP_GRAPH_FILE"
  TOTAL_DOWNSTREAM=0

  while IFS= read -r FILE; do
    [ -z "$FILE" ] && continue
    printf "### %s\n" "$FILE" >> "$DEP_GRAPH_FILE"

    OUTGOING=""
    if [ "$PROJ_LANG" = "js" ] && command -v madge >/dev/null 2>&1; then
      OUTGOING=$(madge --json "$FILE" 2>/dev/null | grep -oE '"[./][^"]*"' | tr -d '"' | tr '\n' ',' | sed 's/,$//' || true)
    fi
    if [ -z "$OUTGOING" ]; then
      OUTGOING=$(grep -E "from '|require\(" "$FILE" 2>/dev/null | grep -oE "'[./][^']+'" | tr -d "'" | tr '\n' ',' | sed 's/,$//' || true)
    fi
    printf "Outgoing: %s\n" "${OUTGOING:-(none)}" >> "$DEP_GRAPH_FILE"

    BASENAME=$(basename "$FILE" | sed 's/\.[^.]*$//')
    DIRECT=$(grep -rl "$BASENAME" --include="*.ts" --include="*.tsx" --include="*.js" \
      --include="*.jsx" --include="*.py" --include="*.go" --include="*.rs" . 2>/dev/null \
      | grep -v "\.codex-tmp" | grep -v "\.git" | sort | head -20 || true)
    DIRECT_LIST=$(echo "$DIRECT" | tr '\n' ',' | sed 's/,$//' | sed 's/^,//')
    # grep -c outputs "0" on no match and exits 1 — use || true, NOT || echo 0 (avoids double output)
    DIRECT_COUNT=$(echo "$DIRECT" | grep -c '[^[:space:]]' 2>/dev/null || true)
    printf "Incoming (direct): %s\n" "${DIRECT_LIST:-(none)}" >> "$DEP_GRAPH_FILE"

    SECOND_ORDER_ALL=""
    while IFS= read -r CONSUMER; do
      [ -z "$CONSUMER" ] && continue
      CON_BASE=$(basename "$CONSUMER" | sed 's/\.[^.]*$//')
      SECOND=$(grep -rl "$CON_BASE" --include="*.ts" --include="*.tsx" --include="*.js" \
        --include="*.jsx" --include="*.py" --include="*.go" --include="*.rs" . 2>/dev/null \
        | grep -v "\.codex-tmp" | grep -v "\.git" | sort || true)
      SECOND_ORDER_ALL=$(printf "%s\n%s" "$SECOND_ORDER_ALL" "$SECOND")
    done <<< "$DIRECT"

    SECOND_UNIQUE=$(echo "$SECOND_ORDER_ALL" | sort -u | grep '[^[:space:]]' || true)
    SECOND_COUNT=$(echo "$SECOND_UNIQUE" | grep -c '[^[:space:]]' 2>/dev/null || true)
    SECOND_TOP=$(echo "$SECOND_UNIQUE" | head -10 | tr '\n' ',' | sed 's/,$//')

    if [ "$SECOND_COUNT" -gt 10 ]; then
      REMAINDER=$((SECOND_COUNT - 10))
      printf "Incoming (2nd-order): %s ...and %d more (%d total)\n" "$SECOND_TOP" "$REMAINDER" "$SECOND_COUNT" >> "$DEP_GRAPH_FILE"
    else
      printf "Incoming (2nd-order): %s\n" "${SECOND_TOP:-(none)}" >> "$DEP_GRAPH_FILE"
    fi

    TOTAL_DOWNSTREAM=$((TOTAL_DOWNSTREAM + DIRECT_COUNT + SECOND_COUNT))
    printf "\n" >> "$DEP_GRAPH_FILE"
  done <<< "$PLAN_FILES_DEP"

  printf "Blast radius: %d files downstream of plan changes\n" "$TOTAL_DOWNSTREAM" >> "$DEP_GRAPH_FILE"
  echo "Step 2.5: Dependency analysis complete. $(grep 'Blast radius' "$DEP_GRAPH_FILE" || true)"
fi
} || echo "Step 2.5: Dependency analysis failed — continuing without graph."

# --- Build combined request (multi-agent instructions + plan in one file) ---
cat > "$REQUEST_FILE" << 'INSTRUCTIONS_EOF'
You are reviewing an implementation plan created by another AI assistant (Claude Code). Your job is to find CRITICAL blind spots — things that will cause bugs, data loss, race conditions, breaking changes, or architectural problems.

You have access to the project files in this directory.

## Mandatory Multi-Agent Workflow

You MUST follow this workflow — do not skip to a single-pass review:

1) Spawn an `explorer` sub-agent for code-reality validation.
   - Read all files listed under "Key Files to Examine" in the plan.
   - Read all files listed under "Files to Modify" in the plan.
   - Check imports, dependencies, and integration points in each file.
   - If a "Dependency Impact Graph" section is present below the plan: validate it.
     Find any downstream consumers or import paths the graph missed. Flag any file
     in the blast radius that the plan does not account for.
   - Output ONLY: concrete mismatches between the plan and actual code, missing
     dependencies, broken imports, incorrect API assumptions, file-level risks,
     and any blast-radius gaps. Include exact file paths and line numbers.

2) Spawn a second sub-agent for architecture and risk review.
   - Evaluate system boundaries, component coupling, and sequencing.
   - Check for migration/rollback risks, data consistency issues, and race conditions.
   - Verify the plan's sequence won't create intermediate broken states.
   - Check for missing env variables, config changes, or deployment steps.
   - Output ONLY: architectural risks, missing design decisions, sequencing problems, and safer alternatives. Be specific — cite plan sections.

3) Wait for both agents to complete, then synthesize their findings.

## Review Standards (apply to all agents)

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

## Output Format

Return EXACTLY this structure:

**Verdict:** APPROVE | APPROVE_WITH_CHANGES | BLOCK

**Critical Findings** (ordered by severity):
1. [Finding] — [Why it matters] — Evidence: `file:line`

**Medium Risks** (worth noting but not blocking):
- [Risk] — [Context]

**Missing Steps** (required additions to the plan):
- [Step] — [Why it's needed]

**Assumptions / Open Questions**:
- [Assumption that could not be verified] — [What to check]

Limit Critical Findings to the TOP 5 most severe. If nothing critical was found, say so explicitly.

---

The plan to review follows below.

---

INSTRUCTIONS_EOF
cat "$PLAN_FILE" >> "$REQUEST_FILE"

# --- Inject dependency graph into request (if computed) ---
if [ "$DEP_GRAPH_AVAILABLE" = "true" ] && [ -s "$DEP_GRAPH_FILE" ]; then
  printf "\n---\n\n" >> "$REQUEST_FILE"
  cat "$DEP_GRAPH_FILE" >> "$REQUEST_FILE"
fi

# --- Build single-shot fallback prompt (strips multi-agent instructions) ---
FALLBACK_FILE="$WORK_DIR/fallback.md"
cat > "$FALLBACK_FILE" << 'FALLBACK_INSTRUCTIONS_EOF'
You are reviewing an implementation plan created by another AI assistant (Claude Code). Your job is to find CRITICAL blind spots — things that will cause bugs, data loss, race conditions, breaking changes, or architectural problems.

You have access to the project files in this directory. Prioritize reading files listed under "Key Files to Examine" in the plan, then check other referenced files as needed.

If a "Dependency Impact Graph" section is included, validate it — find any downstream consumers the graph missed and flag blast-radius files the plan doesn't address.

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

Return EXACTLY this structure:

**Verdict:** APPROVE | APPROVE_WITH_CHANGES | BLOCK

**Critical Findings** (ordered by severity):
1. [Finding] — [Why it matters] — Evidence: `file:line`

**Medium Risks** (worth noting but not blocking):
- [Risk] — [Context]

**Missing Steps** (required additions to the plan):
- [Step] — [Why it's needed]

**Assumptions / Open Questions**:
- [Assumption that could not be verified] — [What to check]

Limit Critical Findings to the TOP 5 most severe. If nothing critical was found, say so explicitly.

---

The plan to review follows below.

---

FALLBACK_INSTRUCTIONS_EOF
cat "$PLAN_FILE" >> "$FALLBACK_FILE"

# --- Inject dependency graph into fallback request (if computed) ---
if [ "$DEP_GRAPH_AVAILABLE" = "true" ] && [ -s "$DEP_GRAPH_FILE" ]; then
  printf "\n---\n\n" >> "$FALLBACK_FILE"
  cat "$DEP_GRAPH_FILE" >> "$FALLBACK_FILE"
fi

# --- Call Codex with multi-agent enabled (adjust model flags per Step 1) ---
# NOTE: Do NOT use --full-auto — it overrides --sandbox read-only with workspace-write.
# In exec mode, approval defaults to 'never', so --full-auto is unnecessary.
cat "$REQUEST_FILE" | codex exec \
  --ephemeral \
  --enable multi_agent \
  --sandbox read-only \
  -m gpt-5.3-codex \
  -c model_reasoning_effort="high" \
  -c agents.max_threads=3 \
  -c agents.max_depth=1 \
  -c agents.job_max_runtime_seconds=900 \
  -C "$(pwd)" \
  -o "$OUTPUT_FILE" \
  -

CODEX_EXIT=$?

# --- Check for empty output (silent false-positive guard) ---
if [ $CODEX_EXIT -eq 0 ] && [ ! -s "$OUTPUT_FILE" ]; then
  echo "WARNING: Codex exited successfully but produced no output. Treating as failure."
  CODEX_EXIT=1
fi

# --- Check for multi-agent failure and fallback to single-shot ---
if [ $CODEX_EXIT -ne 0 ]; then
  echo "WARNING: Multi-agent review failed (exit code $CODEX_EXIT). Falling back to single-shot mode..."
  # Use the fallback prompt that doesn't reference sub-agent spawning
  cat "$FALLBACK_FILE" | codex exec \
    --full-auto \
    --ephemeral \
    --sandbox read-only \
    -m gpt-5.3-codex \
    -c model_reasoning_effort="high" \
    -C "$(pwd)" \
    -o "$OUTPUT_FILE" \
    -
  CODEX_EXIT=$?
  if [ $CODEX_EXIT -eq 0 ] && [ ! -s "$OUTPUT_FILE" ]; then
    echo "WARNING: Codex exited successfully but produced no output. Treating as failure."
    CODEX_EXIT=1
  fi
  if [ $CODEX_EXIT -ne 0 ]; then
    echo "ERROR: Codex review failed in both multi-agent and single-shot mode (exit code $CODEX_EXIT)."
    echo "Possible causes:"
    echo "  - Auth: run 'codex login' or set OPENAI_API_KEY"
    echo "  - Repo trust: run 'codex' in this directory once to trust it"
    echo "  - Model unavailable: check 'codex' docs for current model IDs"
    echo "  - Not a git repo: try adding --skip-git-repo-check"
    rm -rf "$WORK_DIR"
    exit 1
  fi
  echo "NOTE: Review completed in single-shot fallback mode (multi_agent was unavailable)."
fi

# --- Read and display the output ---
echo "=== CODEX REVIEW OUTPUT ==="
cat "$OUTPUT_FILE"
echo "=== END OUTPUT ==="

# --- Cleanup ---
rm -rf "$WORK_DIR"
```

Set the Bash tool timeout to `600000` (10 minutes) when running this command — multi-agent reviews take longer than single-shot. If it times out, clean up orphaned dirs with: `rm -rf "$(pwd)"/.codex-tmp-*`

**Important notes on the Codex call:**
- `--enable multi_agent` activates Codex's sub-agent orchestration (spawns parallel worker threads)
- `--sandbox read-only` restricts Codex to reading files only — appropriate for a review task. Use `--sandbox workspace-write` only if you need Codex to run build/test commands
- Do NOT use `--full-auto` — it silently overrides `--sandbox read-only` with `workspace-write`. In exec mode, approval is already set to `never` by default, so `--full-auto` is unnecessary
- `--ephemeral` avoids polluting session history with review artifacts
- `-C "$(pwd)"` gives Codex access to the project files so it can verify assumptions against actual code
- `-o` (`--output-last-message`) captures Codex's final synthesized response to a file for reliable reading
- `-c agents.max_threads=N` controls how many sub-agents can run in parallel
- `-c agents.max_depth=1` prevents sub-agents from spawning their own sub-agents (safety guard)
- `-c agents.job_max_runtime_seconds=N` sets per-worker timeout to prevent runaway agents
- The prompt explicitly instructs Codex to spawn two sub-agents — this is the first layer of control. Codex's multi_agent orchestrator handles the actual spawning, waiting, and result merging
- The fallback logic catches cases where multi_agent is unavailable (older Codex versions, feature disabled, or runtime errors) and retries as a standard single-shot review
- The skill always explicitly selects a model — never defers to user config defaults — for deterministic review quality

**If Codex fails with an auth error:** Tell the user to run `codex login` or set `OPENAI_API_KEY`, then retry.

### Step 6: Filter — The Safety Net

Even with multi-agent review, Codex has a well-known tendency to:

- **Over-securitize**: Adding auth checks, input validation, and error handling everywhere, even for internal functions that receive trusted data. If the plan is for an internal tool or a prototype, most security suggestions are noise.
- **Over-abstract**: Suggesting interfaces, factories, and extra layers for things that don't need them yet. One concrete implementation beats a premature abstraction.
- **Scope creep**: Suggesting adjacent features, "while you're at it" additions, or "best practice" extras that expand the scope beyond what was asked.
- **Framework orthodoxy**: Insisting on patterns that a framework supports but the project doesn't use. If the codebase has its own conventions, those win.
- **Sub-agent echo**: With multi-agent mode, both sub-agents may flag the same issue from different angles. Deduplicate findings — present each issue once with the strongest evidence.

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
| Duplicate findings from sub-agents | Repeated issues already covered |

### Step 7: Present Findings to the User

Present the filtered results in this format:

---

**Codex Plan Review — Multi-Agent Blind Spot Report**

**Mode:** Multi-agent (explorer + architecture reviewer) | *or* Single-shot fallback
**Model used:** [model name + reasoning effort]
**Complexity assessed:** [LOW / MEDIUM / HIGH — with brief rationale]
**Plan reviewed:** [brief description]

**Verdict:** APPROVE | APPROVE_WITH_CHANGES | BLOCK

**Blast Radius** *(omit this section entirely if Step 2.5 was skipped — no empty section, no "not available" note)*:
- Traced: [files from plan that were analysed]
- Direct consumers: [files that import plan files]
- 2nd-order: [up to 10 with "...and N more (X total)" if > 10]
- Gaps flagged by Codex: [blast-radius files the plan doesn't account for, per explorer sub-agent]

**Critical Findings** (integrate these before executing):
1. [Finding] — [Why it matters and what to change] — *Evidence: `file:line`*
2. [Finding] — [Why it matters and what to change] — *Evidence: `file:line`*

**Medium Risks** (awareness, may need action):
- [Risk] — [Context]

**Missing Steps** (add to plan before executing):
- [Step] — [Why needed]

**Noted but Non-Critical** (filtered out, awareness only):
- [Brief note if anything was borderline worth knowing]

---

If Codex found nothing critical, say so clearly: "Codex multi-agent review came back clean — no critical blind spots detected. Both explorer and architecture sub-agents validated the plan. Good to execute."

### Step 8: Integrate and Proceed

If there were critical findings or a BLOCK verdict:
1. Update the plan to address each critical finding
2. Show the user what changed and why
3. Ask if they want to re-review the updated plan or proceed

If the verdict was APPROVE or APPROVE_WITH_CHANGES with no critical findings:
1. Proceed directly to execution
2. Note in the plan that it was cross-checked via multi-agent review

## Edge Cases

**Codex is unavailable or errors out:**
- Preflight check (Step 0) catches installation and version issues early
- If `codex exec` fails with an auth error, tell the user to run `codex login` or set `OPENAI_API_KEY`
- If multi-agent mode fails, the skill automatically falls back to single-shot review
- If both modes fail, show the error and ask if the user wants to proceed without review

**Multi-agent not available:**
- If Codex version is below 0.104.0 or `multi_agent` feature is not enabled, the fallback runs automatically
- The single-shot fallback uses a separate prompt without multi-agent instructions (avoids impossible "spawn sub-agents" directives)
- Report to user: "Ran in single-shot fallback mode — multi_agent was unavailable"

**Codex takes too long:**
- The Bash tool timeout (600000ms / 10 minutes) will kill the process — multi-agent reviews need more time than single-shot
- Individual sub-agents are bounded by `job_max_runtime_seconds` (configurable per complexity tier)
- If it times out, report it and clean up: `rm -rf "$(pwd)"/.codex-tmp-*`

**Codex returns only non-critical noise:**
- This is fine and expected. Report: "Codex multi-agent review complete — no critical issues found. All suggestions were non-critical (security hardening, additional validation) and filtered out."

**Sub-agent echo / duplicates:**
- Multi-agent mode can produce duplicate findings from both sub-agents. Always deduplicate in Step 6 before presenting to the user.

**User wants to use a different model:**
- Always respect explicit user model requests: `codex exec -m [model] ...`
- Common options: `o3`, `o4-mini`, `gpt-5.3-codex-spark`, `gpt-5.3-codex`

**User wants single-shot mode:**
- If the user says "skip multi-agent", "single shot", or "no sub-agents", remove the `--enable multi_agent` flag and agent config overrides from the `codex exec` call. The rest of the workflow remains the same.

**Legacy `collab` configuration:**
- The `multi_agent` feature was previously called `collab`. If you see `[collab]` in a user's `~/.codex/config.toml`, advise them to rename it to `[features]` with `multi_agent = true` to avoid deprecation warnings. The CLI flag is `--enable multi_agent` (not `--enable collab`).
