# How This Skill Was Built

The codex-plan-review skill was developed in a single session through iterative refinement, with each version being reviewed by Codex itself — the tool it orchestrates. This created a recursive improvement loop: write the skill, have Codex review it, fix what it found, repeat.

## Version History

### v1.0.0 — The First Draft

The original concept: Claude Code writes a plan to `/tmp`, calls `codex exec` with a focused prompt, reads the output, filters it, and presents findings.

**What worked from the start:**
- The Keep/Discard filter table
- The "over-securitize, over-abstract, scope creep, framework orthodoxy" warnings
- The negative constraints in the Codex prompt ("DO NOT suggest...")
- Edge case handling

**What it got wrong:**
- Hardcoded model name (`gpt-5.3-codex-spark`)
- Fixed temp file paths (collision risk)
- Plan text embedded as `$(cat ...)` in a shell argument (ARG_MAX risk)
- Used `timeout 300` (doesn't exist on macOS)

### v1.1.0 — First Codex Review

Sent the v1.0.0 skill to Codex 5.3 (non-spark, high reasoning) for review. It came back with 5 new findings Claude had missed:

1. **`timeout` is macOS-broken** — macOS doesn't ship with GNU `timeout`
2. **Shell argument length limits** — large plans embedded via `$(cat ...)` hit `ARG_MAX`
3. **No `trap` cleanup** — temp files orphaned on error/interrupt
4. **No privacy notice** — Codex gets read access to project files
5. **Preflight should check version, not just existence**

Added automatic model selection, plan template with "Key Files to Examine" and "Constraints / Non-Goals", two-layer filter explanation, and version in frontmatter.

### v1.2.0 — Second Codex Review (The Critical Bug)

Sent v1.1.0 to Codex for a final pass. It found a **critical correctness bug**:

```bash
# THIS IS BROKEN:
cat "$PLAN_FILE" | codex exec ... - << 'PROMPT_EOF'
```

This command tries to send two things to stdin simultaneously: the pipe (`cat "$PLAN_FILE" |`) and the heredoc (`<< 'PROMPT_EOF'`). In bash, the heredoc wins and the piped plan is silently dropped. In zsh, behavior is unpredictable.

**Impact:** Codex was reviewing the prompt instructions but never seeing the actual plan. It would return "looks good" because it had nothing to critique. The skill appeared to work but was producing false "clean" verdicts.

**Fix:** Build a single combined file (instructions + plan) and pipe that one file via stdin.

Also fixed: model routing inconsistency (LOW said "use spark" but didn't pass `-m`), version parse empty string, misleading auth claim, `trap` not firing across shells, and placeholder prompts.

### v1.3.0 — Third Codex Review (Platform Bugs)

Sent v1.2.0 for a third review. Codex found:

1. **macOS `mktemp` with `.md` suffix doesn't randomize** — `mktemp /tmp/foo-XXXXXX.md` creates a literal file called `XXXXXX.md` on macOS. The `X` characters are only randomized when they're at the end of the template, with no suffix after them.

   **Fix:** `mktemp` without extension, then `mv` to add `.md`.

2. **Shell state lost between Claude Code commands** — Claude Code runs each bash command in a separate shell process, so `$PLAN_FILE` set in one command is empty in the next.

   **Fix:** Merge Steps 3-5 into a single bash block.

3. **Document drift** — The skill embedded in the shareable document could diverge from the installed SKILL.md.

   **Fix:** Copy from source of truth, verify with `diff`.

Also fixed CRLF line endings in the shareable document and added installation troubleshooting.

## The Recursive Pattern

What made this effective was using the tool to improve itself:

```
Claude writes skill v1.0
    → Codex reviews → finds 5 issues
Claude fixes → skill v1.1
    → Codex reviews → finds CRITICAL stdin bug
Claude fixes → skill v1.2
    → Codex reviews → finds platform-specific bugs
Claude fixes → skill v1.3
    → Codex reviews → clean pass (should fix only)
```

Each AI caught things the other missed:
- **Claude** wrote the architecture, the filter logic, the model selection
- **Codex** caught the platform bugs, the shell edge cases, the stdin semantics

This is exactly what the skill is designed to enable for any implementation plan.

## Lessons Learned

1. **Shell commands that look correct often aren't.** The pipe + heredoc bug looks fine at a glance. It takes careful reasoning about how bash processes stdin to see it's broken.

2. **macOS is not Linux.** `timeout`, `mktemp` with suffixes, and default shells all behave differently. Test on both.

3. **AI agent execution environments aren't normal shells.** Claude Code's separate-shell-per-command model breaks `trap`, environment variables, and any assumption about persistent state.

4. **Two-layer filtering works.** Telling the model to self-filter removes ~60% of noise. Having the orchestrator filter the rest catches the remaining 40%.

5. **The tool reviewing itself is legitimate testing.** Not a replacement for real-world usage, but catches mechanical bugs that desk-checking misses.
