# claude-select

Minimal CLI wrapper for running Claude Code with multiple accounts. Run concurrent sessions in different terminal tabs — each with its own account — while sharing settings, skills, agents, and CLAUDE.md.

## The problem

Using separate `CLAUDE_CONFIG_DIR` directories (e.g. `~/.claude-work`, `~/.claude-personal`) duplicates all your settings, skills, agents, and CLAUDE.md. You end up maintaining multiple copies of everything just to separate accounts.

## How this solves it

- Each account gets its own `CLAUDE_CONFIG_DIR` profile directory
- Auth is handled by the macOS Keychain (each profile gets its own namespaced Keychain entry — concurrent sessions just work)
- Shared resources (settings.json, agents, CLAUDE.md, etc.) are symlinked from `~/.claude/` into each profile — one set of settings to maintain

## Install

```bash
# Copy the script somewhere in your PATH
cp claude-select /usr/local/bin/
chmod +x /usr/local/bin/claude-select

# Requires jq
brew install jq
```

### Optional: make bare `claude` use your default account

Add this to your `~/.zshrc`:

```bash
claude() {
  if [[ -f "$HOME/.claude-accounts/config.json" ]]; then
    local default_account
    default_account=$(jq -r '.default // ""' "$HOME/.claude-accounts/config.json")
    if [[ -n "$default_account" ]]; then
      claude-select "$default_account" "$@"
      return
    fi
  fi
  command claude "$@"
}
```

## Quick start

```bash
# 1. Add your accounts (each opens a browser login + color picker)
claude-select add work        # sign in with your work account, pick a color
claude-select add personal    # sign in with your personal account, pick a color

# 2. Set a default
claude-select default work

# 3. Use it
claude-select              # interactive picker
claude-select work         # launch directly
claude-select personal     # launch directly

# Extra flags pass through to claude:
claude-select work --model sonnet
```

## Migrating from separate config dirs

If you already have `~/.claude-work` and `~/.claude-personal`:

```bash
# 1. Add accounts (this creates new profiles and triggers login)
claude-select add work
claude-select add personal
claude-select default work

# 2. Your old ~/.claude-work and ~/.claude-personal dirs can be removed
#    once you've verified everything works.
```

Your settings should already live in `~/.claude/` — that's what gets symlinked into each profile.

## How it works

```
~/.claude/                       ← Your main settings (source of truth)
├── settings.json
├── CLAUDE.md
└── agents/

~/.claude-accounts/
├── config.json                  ← Account registry, default, colors
└── profiles/
    ├── work/
    │   ├── settings.json        → symlink → ~/.claude/settings.json
    │   ├── agents/              → symlink → ~/.claude/agents/
    │   ├── CLAUDE.md            → symlink → ~/.claude/CLAUDE.md
    │   └── projects/            ← Per-account (history, todos, etc.)
    └── personal/
        ├── settings.json        → symlink → ~/.claude/settings.json
        ├── agents/              → symlink → ~/.claude/agents/
        ├── CLAUDE.md            → symlink → ~/.claude/CLAUDE.md
        └── projects/            ← Per-account
```

Auth lives in the macOS Keychain, namespaced per profile. Shared resources are symlinked. Account-specific data (projects, history, todos) stays in each profile directory.

## Commands

| Command | Description |
|---|---|
| `claude-select` | Interactive account picker |
| `claude-select <name>` | Launch Claude with that account |
| `claude-select add <name>` | Add account (opens browser login) |
| `claude-select remove <name>` | Remove an account |
| `claude-select login <name>` | Re-authenticate an existing account |
| `claude-select color <name>` | Change an account's status line color |
| `claude-select list` | List all accounts (with color swatches) |
| `claude-select default <name>` | Set default account |
| `claude-select sync` | Re-sync shared symlinks |

## Account colors

Each account gets a color that shows in the Claude Code status line, so you can instantly tell which account you're using. You pick a color when adding an account, and can change it anytime:

```bash
claude-select color work
```

Available colors: blue, green, red, purple, orange, cyan, pink, yellow.

The color is passed to your [status line script](https://docs.anthropic.com/en/docs/claude-code/status-line) via `CLAUDE_ACCOUNT_NAME` and `CLAUDE_ACCOUNT_COLOR` environment variables. If you use a custom status line, you can read these to display the account indicator however you like.

## Customizing shared resources

By default, these are symlinked from `~/.claude/`:

- `settings.json`, `settings.local.json`
- `CLAUDE.md`
- `agents/`
- `statsig/`

Edit the `SHARED_RESOURCES` array near the top of the script to add or remove items.

## If auth expires

If you get authentication errors after not using an account for a while, just re-login:

```bash
claude-select login work
```
