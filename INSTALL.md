# Installation Guide

## Prerequisites

| Requirement | Check | Install |
|-------------|-------|---------|
| Claude Code | `claude --version` | [claude.ai/claude-code](https://claude.ai/claude-code) |
| Codex CLI >= 0.100.0 | `codex -V` | `npm install -g @openai/codex` |
| OpenAI API key | `codex login` | [platform.openai.com/api-keys](https://platform.openai.com/api-keys) |
| macOS or Linux | `uname` | Windows users: use WSL |

## Install the Skill

### Option 1: Copy (recommended)

```bash
mkdir -p ~/.claude/skills/codex-plan-review
cp skill/SKILL.md ~/.claude/skills/codex-plan-review/SKILL.md
```

### Option 2: Symlink (auto-updates with repo)

```bash
mkdir -p ~/.claude/skills
ln -s "$(pwd)/skill" ~/.claude/skills/codex-plan-review
```

## Configure Codex (Optional)

If you haven't configured a default model, add to `~/.codex/config.toml`:

```toml
model = "gpt-5.3-codex-spark"
model_reasoning_effort = "medium"
```

The skill overrides this for LOW and HIGH complexity plans, so this default is only used for MEDIUM.

## Verify

1. **Restart Claude Code** (skills are discovered at startup)
2. Say: "review my plan" or "second opinion"
3. Or invoke directly: `/codex-plan-review`

## Verify Manually

Run these checks to confirm everything is wired up:

```bash
# 1. Skill file exists
ls ~/.claude/skills/codex-plan-review/SKILL.md

# 2. Codex is installed and recent enough
codex -V

# 3. Codex can authenticate
codex exec --full-auto --ephemeral "echo hello" 2>&1 | head -5
```

## Uninstall

```bash
rm -rf ~/.claude/skills/codex-plan-review
```

Restart Claude Code after removing.

## Troubleshooting

**Skill doesn't trigger when I say "review my plan":**
- Check file path: `cat ~/.claude/skills/codex-plan-review/SKILL.md | head -5` (should show `---` and `name:`)
- Restart Claude Code
- Try the explicit command: `/codex-plan-review`

**"ERROR: Codex CLI not installed":**
- Run: `npm install -g @openai/codex`
- Verify: `which codex` and `codex -V`

**Codex auth failure:**
- Run: `codex login`
- Or set: `export OPENAI_API_KEY=sk-...`

**Review times out (5 minutes):**
- Try a faster model: tell Claude Code "use spark"
- Check your network connection
- Large plans take longer — consider splitting

**Empty or missing output:**
- Check Codex version: `codex -V` (need >= 0.100.0)
- Ensure you're in a project directory (Codex uses `-C $(pwd)`)
