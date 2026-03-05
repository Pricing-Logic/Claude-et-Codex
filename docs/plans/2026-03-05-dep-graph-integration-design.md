# Design: Dependency-Impact Graph Analysis Integration

**Date:** 2026-03-05
**Version target:** v2.1.0
**Status:** Approved

## Problem

The codex-plan-review skill's architecture sub-agent currently *infers* blast radius
from the plan's file list. It reasons about what might be downstream of a change but
has no concrete graph data. When `auth/jwt.ts` touches a shared utility, the sub-agent
may not know that utility has 43 downstream consumers across the codebase.

Codex's own review of the system identified dependency-impact graph analysis as the
highest-value addition — providing concrete topology data instead of LLM inference.

## Design Decision

**Approach 3 (targeted transitive):**
- Outgoing deps: 1 level (what plan files import)
- Incoming direct: 1 level (who imports plan files)
- Incoming 2nd-order: 1 extra level (who imports the direct consumers)

Rationale: Catches the most common high-risk pattern — changing a shared utility that
ripples through direct callers and their callers — without full recursive traversal that
would be expensive and noisy on large repos.

**Tool strategy: tool-aware with fallback**
- JS/TS: try `madge --json` if installed, fallback to grep
- Python: try `pydeps` if installed, fallback to grep
- Go/Rust/Ruby/other: grep-based only
- Incoming consumers: always grep-based (language-agnostic)

**Cap:** Direct consumers shown in full. Second-order capped at 10, total count always shown.

## Architecture

Three coordinated changes to `skill/SKILL.md`:

### Change 1: New Step 2.5 — Dependency Impact Analysis

New bash block between Step 2 (plan preparation) and Steps 3-5 (Codex call).

**Responsibilities:**
1. Language detection via marker files (package.json, pyproject.toml, go.mod, Cargo.toml)
2. Extract file paths from `$PLAN_FILE` — backtick-quoted paths with code extensions
   (.ts, .tsx, .js, .jsx, .py, .go, .rs, .rb, .java)
3. Outgoing deps per file — tool-aware with grep fallback
4. Incoming direct consumers — grep codebase for import patterns referencing each file
5. Incoming 2nd-order — grep for importers of each direct consumer
6. Write compact summary to `$WORK_DIR/dep-graph.md`

**Graceful failure contract:**
- No files found in plan → skip with inline note, continue
- madge/pydeps not installed → silent fallback to grep
- grep finds nothing → "no consumers found" (valid for new files)
- Entire step errors → warn and continue, never block Codex review

### Change 2: Steps 3-5 Modification — Inject Graph + Update Explorer Instruction

Two additions to the existing bash block:

1. Append `dep-graph.md` to the combined Codex request file (after plan content)
2. Update the explorer sub-agent instruction with:
   > "A dependency impact graph is included below the plan. Validate it — find any
   > downstream consumers or import paths the graph missed. Flag any blast-radius file
   > that the plan does not account for."

### Change 3: Step 7 Modification — Surface Blast Radius to User

New "Blast Radius" section in the output report, positioned after Verdict:

```
**Blast Radius:**
- Traced: [file list from plan]
- Direct consumers: [files that import plan files]
- 2nd-order: [up to 10, with "...and N more (X total)" if > 10]
- Codex flagged: [any gaps the explorer sub-agent identified]
```

If dep-graph was skipped (no files found, step failed), omit this section silently.

## Data Shapes

### dep-graph.md injected into Codex prompt

```markdown
## Dependency Impact Graph

### src/auth/jwt.ts
Outgoing: src/lib/db.ts, src/config/env.ts
Incoming (direct): src/api/login.ts, src/api/refresh.ts
Incoming (2nd-order): src/app.ts, src/middleware/auth.ts, src/router.ts

### src/config/env.ts
Outgoing: (none)
Incoming (direct): src/auth/jwt.ts, src/lib/db.ts
Incoming (2nd-order): src/app.ts, src/middleware/auth.ts, src/router.ts,
  src/hooks/useAuth.ts, src/api/users.ts, src/api/orders.ts,
  src/lib/logger.ts, src/services/email.ts, src/jobs/cleanup.ts,
  src/tests/integration.ts ...and 33 more (43 total)

Blast radius: 47 files downstream of plan changes
```

### File path extraction regex

Match backtick-quoted paths containing a known extension:
```
`[^`]+\.(ts|tsx|js|jsx|py|go|rs|rb|java|css|json|yaml|yml|toml|sql)[^`]*`
```

Strip backticks, deduplicate, filter to files that exist on disk.

### Import grep patterns by language

| Language | Pattern |
|----------|---------|
| JS/TS | `^import\|from '[^']*'\|require('[^']*')` |
| Python | `^import \|^from .*import` |
| Go | `"` within import blocks |
| Rust | `^use ` |
| Ruby | `require\|require_relative` |

### Incoming consumer search

For each plan file `src/auth/jwt.ts`, search the codebase for:
```bash
grep -r "jwt" --include="*.ts" --include="*.tsx" -l .
```
Use the basename without extension to avoid path-dependent matching.

## Constraints / Non-Goals

- No recursive traversal beyond 2 levels (direct + 2nd-order)
- No modifying the Codex call flags or sandbox policy
- No new files outside SKILL.md (everything inline — single-file install model)
- No blocking the Codex review if dep-graph fails
- No adding more LLM reviewers

## Success Criteria

1. Step 2.5 runs and produces `dep-graph.md` for a TS/JS project with plan files listed
2. Codex request includes the graph as additional context
3. Step 7 output includes a Blast Radius section
4. Fallback works: removing madge still produces a grep-based graph
5. Failure is non-blocking: empty plan still completes the review
