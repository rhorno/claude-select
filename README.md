# claude-select

Minimal CLI wrapper for running Claude Code with multiple accounts. Each shell session gets its own account while sharing settings, skills, agents, and CLAUDE.md.

## Why

Using separate `CLAUDE_CONFIG_DIR` directories duplicates all your settings, skills, agents, and CLAUDE.md. claude-select gives each account its own profile directory while symlinking shared resources from `~/.claude/` — one set of settings to maintain.

### Authentication

Each profile gets its own `CLAUDE_CONFIG_DIR`, so `claude login` stores separate credentials per account. Concurrent sessions just work.

- **macOS** — credentials are stored in the Keychain, namespaced by profile directory
- **Linux / WSL** — credentials are stored in the profile directory itself (file-based)

## Install

```bash
# Copy the script somewhere in your PATH
cp claude-select /usr/local/bin/
chmod +x /usr/local/bin/claude-select

# Requires jq for reading and writing the account config (JSON)
# macOS: brew install jq
# Linux: sudo apt install jq (or your package manager)
```

### Optional: make bare `claude` use your default account

Add this to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.):

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
# 1. Add your accounts (each opens browser login + color picker)
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

## How it works

```
~/.claude/                       ← Your main settings (source of truth)
├── settings.json
├── CLAUDE.md
├── agents/
└── skills/

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

Shared resources are symlinked. Account-specific data (projects, history, todos) stays in each profile directory.

## Commands

| Command | Description |
|---|---|
| `claude-select` | Interactive account picker |
| `claude-select <name>` | Launch Claude with that account |
| `claude-select add <name>` | Add account (opens browser login) |
| `claude-select remove <name>` | Remove an account |
| `claude-select login <name>` | Re-authenticate an existing account |
| `claude-select color <name>` | Change an account's status line color |
| `claude-select rename <old> <new>` | Rename an account |
| `claude-select list` | List all accounts (with color swatches) |
| `claude-select default <name>` | Set default account |
| `claude-select sync` | Re-sync shared symlinks |

## Account colors and names

Each account gets a hex color you pick when adding an account. Change it anytime:

```bash
claude-select color work
```

When launching Claude, these environment variables are set:

| Variable | Format | Example |
|---|---|---|
| `CLAUDE_ACCOUNT_NAME` | Account name string | `work` |
| `CLAUDE_ACCOUNT_COLOR` | Semicolon-separated RGB | `97;175;239` |

You can use these in a [custom status line](https://docs.anthropic.com/en/docs/claude-code/status-line) to display which account is active. For example, in your `~/.claude/settings.json`:

```json
{
  "statusLine": "printf '%s' \"$CLAUDE_ACCOUNT_NAME\""
}
```

