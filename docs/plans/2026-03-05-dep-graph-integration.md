# Dependency-Impact Graph Integration — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add dependency-impact graph analysis to the codex-plan-review skill so Codex receives concrete blast-radius topology data instead of inferring it.

**Architecture:** Three coordinated edits to `skill/SKILL.md` only — new Step 2.5 bash block, injection of graph into the Codex request, and a Blast Radius section in Step 7 output. No new files outside SKILL.md. Single-file install model preserved.

**Tech Stack:** Bash, grep (universal fallback), madge (JS/TS optional), pydeps (Python optional). All bash, no Python/Node helper scripts.

---

## Task 1: Validate the dep-graph bash logic in isolation

Test the core logic before inserting it into SKILL.md. This catches syntax errors and validates output format without touching the skill.

**Files:**
- Create (temp, delete after): `.codex-dep-test.sh`

**Step 1: Write the test script**

Create `.codex-dep-test.sh` with this content:

```bash
#!/usr/bin/env bash
# Test dep-graph logic against current repo
set -e

PLAN_FILE="$(pwd)/docs/plans/2026-03-05-dep-graph-integration-design.md"
WORK_DIR="$(pwd)/.codex-dep-test-$$"
mkdir -p "$WORK_DIR"
DEP_GRAPH_FILE="$WORK_DIR/dep-graph.md"
DEP_GRAPH_AVAILABLE=false

# Language detection
PROJ_LANG="unknown"
[ -f "package.json" ] && PROJ_LANG="js"
[ -f "pyproject.toml" ] || [ -f "requirements.txt" ] && PROJ_LANG="python"
[ -f "go.mod" ] && PROJ_LANG="go"
[ -f "Cargo.toml" ] && PROJ_LANG="rust"
echo "Detected language: $PROJ_LANG"

# Extract file paths from plan (any path-like string with known extension that exists on disk)
PLAN_FILES=$(grep -oE '[a-zA-Z0-9/_.-]+\.(ts|tsx|js|jsx|py|go|rs|rb|java|md|sh)' "$PLAN_FILE" 2>/dev/null | sort -u | while IFS= read -r f; do [ -f "$f" ] && echo "$f"; done)
echo "Files found in plan:"
echo "$PLAN_FILES"
echo "---"

if [ -z "$PLAN_FILES" ]; then
  echo "NOTE: No recognizable file paths found in plan — skipping dependency analysis."
else
  DEP_GRAPH_AVAILABLE=true
  printf "## Dependency Impact Graph\n\n" > "$DEP_GRAPH_FILE"
  TOTAL_DOWNSTREAM=0

  while IFS= read -r FILE; do
    [ -z "$FILE" ] && continue
    [ -f "$FILE" ] || continue
    printf "### %s\n" "$FILE" >> "$DEP_GRAPH_FILE"

    # Outgoing deps — try madge for JS/TS, fallback to grep
    OUTGOING=""
    if [ "$PROJ_LANG" = "js" ] && command -v madge >/dev/null 2>&1; then
      OUTGOING=$(madge --json "$FILE" 2>/dev/null | grep -oE '"[./][^"]*"' | tr -d '"' | tr '\n' ',' | sed 's/,$//' || true)
    fi
    if [ -z "$OUTGOING" ]; then
      OUTGOING=$(grep -E "from '|require\(" "$FILE" 2>/dev/null | grep -oE "'[./][^']+'" | tr -d "'" | tr '\n' ',' | sed 's/,$//' || true)
    fi
    printf "Outgoing: %s\n" "${OUTGOING:-(none)}" >> "$DEP_GRAPH_FILE"

    # Incoming direct consumers — grep for basename in codebase
    BASENAME=$(basename "$FILE" | sed 's/\.[^.]*$//')
    DIRECT=$(grep -rl "$BASENAME" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.py" --include="*.go" --include="*.rs" --include="*.md" . 2>/dev/null \
      | grep -v "\.codex-tmp" | grep -v "\.git" | grep -v "^\./$FILE" | sort | head -20 || true)
    DIRECT_LIST=$(echo "$DIRECT" | tr '\n' ',' | sed 's/,$//' | sed 's/^,//')
    DIRECT_COUNT=$(echo "$DIRECT" | grep -c '[^[:space:]]' || echo 0)
    printf "Incoming (direct): %s\n" "${DIRECT_LIST:-(none)}" >> "$DEP_GRAPH_FILE"

    # Incoming 2nd-order — grep for importers of each direct consumer
    SECOND_ORDER_ALL=""
    while IFS= read -r CONSUMER; do
      [ -z "$CONSUMER" ] && continue
      CON_BASE=$(basename "$CONSUMER" | sed 's/\.[^.]*$//')
      SECOND=$(grep -rl "$CON_BASE" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" --include="*.py" --include="*.go" --include="*.md" . 2>/dev/null \
        | grep -v "\.codex-tmp" | grep -v "\.git" | grep -v "^\./$CONSUMER" | sort || true)
      SECOND_ORDER_ALL=$(printf "%s\n%s" "$SECOND_ORDER_ALL" "$SECOND")
    done <<< "$DIRECT"

    SECOND_UNIQUE=$(echo "$SECOND_ORDER_ALL" | sort -u | grep '[^[:space:]]' || true)
    SECOND_COUNT=$(echo "$SECOND_UNIQUE" | grep -c '[^[:space:]]' || echo 0)
    SECOND_TOP=$(echo "$SECOND_UNIQUE" | head -10 | tr '\n' ',' | sed 's/,$//')

    if [ "$SECOND_COUNT" -gt 10 ]; then
      REMAINDER=$((SECOND_COUNT - 10))
      printf "Incoming (2nd-order): %s ...and %d more (%d total)\n" "$SECOND_TOP" "$REMAINDER" "$SECOND_COUNT" >> "$DEP_GRAPH_FILE"
    else
      printf "Incoming (2nd-order): %s\n" "${SECOND_TOP:-(none)}" >> "$DEP_GRAPH_FILE"
    fi

    TOTAL_DOWNSTREAM=$((TOTAL_DOWNSTREAM + DIRECT_COUNT + SECOND_COUNT))
    printf "\n" >> "$DEP_GRAPH_FILE"
  done <<< "$PLAN_FILES"

  printf "Blast radius: %d files downstream of plan changes\n" "$TOTAL_DOWNSTREAM" >> "$DEP_GRAPH_FILE"
  echo "=== DEP GRAPH OUTPUT ==="
  cat "$DEP_GRAPH_FILE"
  echo "=== END ==="
fi

rm -rf "$WORK_DIR"
```

**Step 2: Run it**

```bash
bash .codex-dep-test.sh
```

Expected: Prints detected language, file list, and a dep-graph with Outgoing/Incoming sections. No errors.

**Step 3: Verify output makes sense**

Check that:
- `skill/SKILL.md` appears as a file if it is referenced in the plan
- Incoming consumers are plausible (other .md files that reference SKILL.md by name)
- No bash errors or empty output due to variable issues

**Step 4: Delete the test script**

```bash
rm .codex-dep-test.sh
```

---

## Task 2: Insert Step 2.5 into SKILL.md

**Files:**
- Modify: `skill/SKILL.md` (insert after line 119, before `### Steps 3-5`)

**Step 1: Read the current boundary**

Confirm line 119 ends with:
```
If a `writing-plans` skill is available, use it to produce the plan.
```
And line 120 is blank, line 121 is `### Steps 3-5: Build Request...`

**Step 2: Insert Step 2.5 section**

Insert the following block between line 119 and the `### Steps 3-5` heading. The block goes between the end of Step 2's prose and the Steps 3-5 heading:

```markdown
### Step 2.5: Dependency Impact Analysis

Run before calling Codex. Traces import/consumer relationships for plan files to give
Codex concrete topology data. Non-blocking — any failure skips gracefully.

```bash
# --- Dependency Impact Analysis (Step 2.5) ---
DEP_GRAPH_FILE="$WORK_DIR/dep-graph.md"
DEP_GRAPH_AVAILABLE=false

# Language detection
PROJ_LANG="unknown"
[ -f "package.json" ] && PROJ_LANG="js"
[ -f "pyproject.toml" ] && PROJ_LANG="python"
[ -f "requirements.txt" ] && PROJ_LANG="python"
[ -f "go.mod" ] && PROJ_LANG="go"
[ -f "Cargo.toml" ] && PROJ_LANG="rust"

# Extract file paths from plan that exist on disk
PLAN_FILES=$(grep -oE '[a-zA-Z0-9/_.-]+\.(ts|tsx|js|jsx|py|go|rs|rb|java)' "$PLAN_FILE" 2>/dev/null \
  | sort -u | while IFS= read -r f; do [ -f "$f" ] && echo "$f"; done)

if [ -z "$PLAN_FILES" ]; then
  echo "NOTE: No recognizable file paths found in plan — skipping dependency analysis."
else
  DEP_GRAPH_AVAILABLE=true
  printf "## Dependency Impact Graph\n\n" > "$DEP_GRAPH_FILE"
  TOTAL_DOWNSTREAM=0

  while IFS= read -r FILE; do
    [ -z "$FILE" ] && continue
    printf "### %s\n" "$FILE" >> "$DEP_GRAPH_FILE"

    # Outgoing deps — madge for JS/TS if available, grep fallback
    OUTGOING=""
    if [ "$PROJ_LANG" = "js" ] && command -v madge >/dev/null 2>&1; then
      OUTGOING=$(madge --json "$FILE" 2>/dev/null | grep -oE '"[./][^"]*"' | tr -d '"' | tr '\n' ',' | sed 's/,$//' || true)
    fi
    if [ -z "$OUTGOING" ]; then
      OUTGOING=$(grep -E "from '|require\(" "$FILE" 2>/dev/null | grep -oE "'[./][^']+'" | tr -d "'" | tr '\n' ',' | sed 's/,$//' || true)
    fi
    printf "Outgoing: %s\n" "${OUTGOING:-(none)}" >> "$DEP_GRAPH_FILE"

    # Incoming direct consumers
    BASENAME=$(basename "$FILE" | sed 's/\.[^.]*$//')
    DIRECT=$(grep -rl "$BASENAME" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
      --include="*.py" --include="*.go" --include="*.rs" . 2>/dev/null \
      | grep -v "\.codex-tmp" | grep -v "\.git" | sort | head -20 || true)
    DIRECT_LIST=$(echo "$DIRECT" | tr '\n' ',' | sed 's/,$//')
    DIRECT_COUNT=$(echo "$DIRECT" | grep -c '[^[:space:]]' 2>/dev/null || echo 0)
    printf "Incoming (direct): %s\n" "${DIRECT_LIST:-(none)}" >> "$DEP_GRAPH_FILE"

    # Incoming 2nd-order
    SECOND_ORDER_ALL=""
    while IFS= read -r CONSUMER; do
      [ -z "$CONSUMER" ] && continue
      CON_BASE=$(basename "$CONSUMER" | sed 's/\.[^.]*$//')
      SECOND=$(grep -rl "$CON_BASE" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" \
        --include="*.py" --include="*.go" --include="*.rs" . 2>/dev/null \
        | grep -v "\.codex-tmp" | grep -v "\.git" | sort || true)
      SECOND_ORDER_ALL=$(printf "%s\n%s" "$SECOND_ORDER_ALL" "$SECOND")
    done <<< "$DIRECT"

    SECOND_UNIQUE=$(echo "$SECOND_ORDER_ALL" | sort -u | grep '[^[:space:]]' || true)
    SECOND_COUNT=$(echo "$SECOND_UNIQUE" | grep -c '[^[:space:]]' 2>/dev/null || echo 0)
    SECOND_TOP=$(echo "$SECOND_UNIQUE" | head -10 | tr '\n' ',' | sed 's/,$//')

    if [ "$SECOND_COUNT" -gt 10 ]; then
      REMAINDER=$((SECOND_COUNT - 10))
      printf "Incoming (2nd-order): %s ...and %d more (%d total)\n" "$SECOND_TOP" "$REMAINDER" "$SECOND_COUNT" >> "$DEP_GRAPH_FILE"
    else
      printf "Incoming (2nd-order): %s\n" "${SECOND_TOP:-(none)}" >> "$DEP_GRAPH_FILE"
    fi

    TOTAL_DOWNSTREAM=$((TOTAL_DOWNSTREAM + DIRECT_COUNT + SECOND_COUNT))
    printf "\n" >> "$DEP_GRAPH_FILE"
  done <<< "$PLAN_FILES"

  printf "Blast radius: %d files downstream of plan changes\n" "$TOTAL_DOWNSTREAM" >> "$DEP_GRAPH_FILE"
  echo "Dependency analysis complete. $(grep 'Blast radius' "$DEP_GRAPH_FILE" || true)"
fi
```

**Important:** This block shares `$WORK_DIR` with Steps 3-5 and MUST run in the same bash invocation as Steps 3-5. Add it to the top of the Steps 3-5 bash block (after WORK_DIR is set up, before writing the plan).
```

**Step 3: Verify insertion**

Read lines 118-130 of skill/SKILL.md. Confirm Step 2.5 heading appears and the bash block is intact.

**Step 4: Commit**

```bash
cd "/c/Users/samue/AAA - Pricing Logic/Codex-et-Claude"
git add skill/SKILL.md
git commit -m "feat: add Step 2.5 dependency impact analysis to skill"
```

---

## Task 3: Integrate Step 2.5 into the main bash block (Steps 3-5)

Step 2.5 uses `$WORK_DIR` and `$PLAN_FILE`. These are defined in the Steps 3-5 bash block. Step 2.5 must run inside that same bash block — after WORK_DIR/PLAN_FILE are set up but before building the request.

**Files:**
- Modify: `skill/SKILL.md` (Steps 3-5 bash block)

**Step 1: Locate the insertion point**

Find this line in the bash block (currently around line 161):
```bash
# --- Write the plan ---
cat > "$PLAN_FILE" << 'PLAN_EOF'
```

**Step 2: Move Step 2.5 prose into the bash block**

The Step 2.5 section added in Task 2 is prose instructions. The actual bash code needs to live *inside* the Steps 3-5 bash block. Restructure SKILL.md so that:

- Step 2.5 prose section (outside the bash block) gives Claude the instructions
- The Step 2.5 bash code is placed *inside* the Steps 3-5 bash code block

Specifically, insert the Step 2.5 bash code after the plan is written and before the request is built:

```bash
# --- Write the plan ---
cat > "$PLAN_FILE" << 'PLAN_EOF'
[Your assembled plan goes here]
PLAN_EOF

# --- Step 2.5: Dependency Impact Analysis ---
# [paste the full Step 2.5 bash block here — see Step 2.5 section above]
```

**Step 3: Inject dep-graph into the request**

After this line (currently ~line 238):
```bash
cat "$PLAN_FILE" >> "$REQUEST_FILE"
```

Add:
```bash
# --- Inject dependency graph into request (if computed) ---
if [ "$DEP_GRAPH_AVAILABLE" = "true" ] && [ -s "$DEP_GRAPH_FILE" ]; then
  printf "\n---\n\n" >> "$REQUEST_FILE"
  cat "$DEP_GRAPH_FILE" >> "$REQUEST_FILE"
fi
```

**Step 4: Inject dep-graph into fallback too**

After this line (~line 293):
```bash
cat "$PLAN_FILE" >> "$FALLBACK_FILE"
```

Add:
```bash
if [ "$DEP_GRAPH_AVAILABLE" = "true" ] && [ -s "$DEP_GRAPH_FILE" ]; then
  printf "\n---\n\n" >> "$FALLBACK_FILE"
  cat "$DEP_GRAPH_FILE" >> "$FALLBACK_FILE"
fi
```

**Step 5: Verify the bash block is syntactically valid**

```bash
bash -n skill/SKILL.md 2>&1 | head -20
```

Expected: errors only for heredoc markers (those aren't valid bash outside a script — that's expected). No variable or syntax errors inside the code blocks.

**Step 6: Commit**

```bash
git add skill/SKILL.md
git commit -m "feat: inject dep-graph into Codex request and fallback"
```

---

## Task 4: Update the explorer sub-agent instruction

**Files:**
- Modify: `skill/SKILL.md` (INSTRUCTIONS_EOF heredoc content, ~line 173)

**Step 1: Find the explorer instruction**

Locate this text inside the `INSTRUCTIONS_EOF` heredoc (~line 173):
```
1) Spawn an `explorer` sub-agent for code-reality validation.
   - Read all files listed under "Key Files to Examine" in the plan.
   - Read all files listed under "Files to Modify" in the plan.
   - Check imports, dependencies, and integration points in each file.
   - Output ONLY: concrete mismatches between the plan and actual code...
```

**Step 2: Add graph validation instruction**

Replace it with:
```
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
```

**Step 3: Apply the same addition to the FALLBACK_INSTRUCTIONS_EOF prompt (~line 245)**

Find:
```
You have access to the project files in this directory. Prioritize reading files listed under "Key Files to Examine" in the plan, then check other referenced files as needed.
```

Add after it:
```
If a "Dependency Impact Graph" section is included, validate it — find any downstream
consumers the graph missed and flag blast-radius files the plan doesn't address.
```

**Step 4: Commit**

```bash
git add skill/SKILL.md
git commit -m "feat: update explorer instruction to validate dep graph"
```

---

## Task 5: Add Blast Radius section to Step 7 output

**Files:**
- Modify: `skill/SKILL.md` (Step 7 output template, ~line 406)

**Step 1: Find the Step 7 output format**

Locate this section (~line 406):
```
**Codex Plan Review — Multi-Agent Blind Spot Report**

**Mode:** Multi-agent (explorer + architecture reviewer) | *or* Single-shot fallback
**Model used:** [model name + reasoning effort]
**Complexity assessed:** [LOW / MEDIUM / HIGH — with brief rationale]
**Plan reviewed:** [brief description]

**Verdict:** APPROVE | APPROVE_WITH_CHANGES | BLOCK
```

**Step 2: Add Blast Radius section after Verdict**

```
**Verdict:** APPROVE | APPROVE_WITH_CHANGES | BLOCK

**Blast Radius** *(omit section if dep-graph was skipped)*:
- Traced: [files from plan that were analysed]
- Direct consumers: [files that import plan files]
- 2nd-order: [up to 10 with "...and N more (X total)" if > 10]
- Gaps flagged by Codex: [any blast-radius files the plan doesn't account for, per explorer]
```

**Step 3: Add instruction for when dep-graph was skipped**

At the end of the Step 7 section, add:
```
If Step 2.5 was skipped (no files found in plan, or step failed), omit the Blast Radius
section entirely — do not show an empty section or a "not available" note.
```

**Step 4: Commit**

```bash
git add skill/SKILL.md
git commit -m "feat: add Blast Radius section to Step 7 output format"
```

---

## Task 6: End-to-end test with a real plan

Test the full pipeline with a plan that references actual files in the repo.

**Step 1: Create a minimal test plan**

Create `.codex-test-plan.md`:
```markdown
## Objective
Test the dep-graph integration by referencing skill/SKILL.md.

## Key Files to Examine
- `skill/SKILL.md` — main skill file
- `docs/full-reference.md` — reference doc

## Files to Modify
- `skill/SKILL.md` — adding Step 2.5

## Constraints / Non-Goals
- This is a test plan only
```

**Step 2: Run the dep-graph bash logic manually against this plan**

Copy the Step 2.5 bash block from SKILL.md and run it directly, pointing `$PLAN_FILE` at `.codex-test-plan.md`:

```bash
PLAN_FILE=".codex-test-plan.md"
WORK_DIR="$(pwd)/.codex-dep-test-$$"
mkdir -p "$WORK_DIR"
DEP_GRAPH_FILE="$WORK_DIR/dep-graph.md"
DEP_GRAPH_AVAILABLE=false

# [paste Step 2.5 bash code here]

if [ "$DEP_GRAPH_AVAILABLE" = "true" ]; then
  cat "$DEP_GRAPH_FILE"
fi
rm -rf "$WORK_DIR"
```

**Step 3: Verify output**

Expected dep-graph.md content:
```
## Dependency Impact Graph

### skill/SKILL.md
Outgoing: (none)
Incoming (direct): [files that reference SKILL.md by name]
Incoming (2nd-order): [files that reference those consumers]

### docs/full-reference.md
Outgoing: (none)
Incoming (direct): [README.md, INSTALL.md, etc.]
Incoming (2nd-order): ...

Blast radius: N files downstream of plan changes
```

**Step 4: Clean up test file**

```bash
rm .codex-test-plan.md
```

---

## Task 7: Run codex-plan-review on the full SKILL.md changes

Before finalising, get Codex to review what was built. Use the codex-plan-review skill with the implementation as the "plan". HIGH complexity, xhigh reasoning.

---

## Task 8: Update changelog in full-reference.md

**Files:**
- Modify: `docs/full-reference.md`

**Step 1: Add v2.1.0 entry**

Insert before the `### v2.0.1` entry:

```markdown
### v2.1.0 (2026-03-05)

**Added: Dependency-Impact Graph Analysis**
- **Step 2.5** — New pre-Codex step traces import/consumer relationships for all files
  mentioned in the plan. Produces a blast-radius summary injected into the Codex request.
- **Tool-aware with fallback** — Uses `madge` for JS/TS if installed, `pydeps` for Python
  if installed, grep-based import extraction for everything else. Always graceful.
- **Targeted transitive depth** — Outgoing deps (1 level) + incoming direct consumers
  (1 level) + incoming 2nd-order consumers (1 level). Cap: 10 per file with total count.
- **Explorer instruction updated** — Explorer sub-agent now validates the dep graph and
  flags blast-radius files the plan doesn't account for.
- **Blast Radius section** — Step 7 output now includes a Blast Radius section showing
  traced files, direct consumers, 2nd-order consumers, and any gaps Codex flagged.
```

**Step 2: Commit**

```bash
git add docs/full-reference.md
git commit -m "docs: add v2.1.0 changelog for dep-graph integration"
```

---

## Task 9: Commit, push, and sync installed skill

**Step 1: Final status check**

```bash
cd "/c/Users/samue/AAA - Pricing Logic/Codex-et-Claude"
git log --oneline -6
git status
```

Expected: clean working tree, 5-6 new commits.

**Step 2: Push**

```bash
git push origin main
```

**Step 3: Sync installed skill**

```bash
cp skill/SKILL.md ~/.claude/skills/codex-plan-review/SKILL.md
echo "Installed skill updated."
head -5 ~/.claude/skills/codex-plan-review/SKILL.md
```

Expected: frontmatter shows `version: 2.1.0` (update this in SKILL.md frontmatter too — change `version: 2.0.1` to `version: 2.1.0`).

**Step 4: Restart Claude Code note**

Tell the user: restart Claude Code to pick up the updated installed skill.

---

## Success Criteria

1. `bash .codex-dep-test.sh` produces a dep-graph with real file data from the repo
2. `skill/SKILL.md` contains Step 2.5 inside the Steps 3-5 bash block
3. The dep-graph is appended to both `$REQUEST_FILE` and `$FALLBACK_FILE`
4. The explorer instruction mentions graph validation
5. Step 7 format includes the Blast Radius section
6. A full skill invocation on a plan with file refs produces a Blast Radius in the output
7. A full skill invocation on a plan with NO file refs skips the section silently
8. Codex review of the changes passes (APPROVE or APPROVE_WITH_CHANGES)
9. `git log` shows clean atomic commits
10. Installed skill at `~/.claude/skills/codex-plan-review/SKILL.md` matches repo
