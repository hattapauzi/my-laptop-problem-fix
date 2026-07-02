# Forge ZSH `:` Commands Not Working in Tmux

Fix for Forge `:` commands (like `:model`) not working inside tmux — typing `:` followed by Tab showed zsh's history modifier list instead of Forge's command picker.

## Environment

- Shell: ZSH with Oh My Zsh
- Prompt theme: Powerlevel10k (p10k)
- Plugins: forge zsh plugin, zsh-autosuggestions, fzf
- Terminal: Ghostty + tmux
- OS: Arch-based Linux (CachyOS)

## Symptom

Typing `:` followed by Tab in a tmux pane showed zsh's history modifier list:

```
a  -- absolute path, resolve '..' lexically
A  -- as ':a', then resolve symlinks
c  -- PATH search for command
e  -- leave only extension
g  -- globally apply s or &
h  -- head - strip trailing path element
l  -- lower case all words
P  -- realpath, resolve '..' physically
q  -- quote to escape further substitutions
Q  -- strip quotes
&  -- repeat substitution
r  -- root - strip suffix
s  -- substitute string
t  -- tail - strip directories
u  -- upper case all words
:
```

This is zsh's built-in completion for the `:` builtin, not Forge's command picker. The issue only occurred inside tmux — fresh terminal windows (Ghostty/Kitty) worked fine.

## Root Cause

Tmux launches **login shells** by default (it prepends `-` to `$0`). The `~/.zprofile` file contained:

```zsh
. ~/.zshrc
```

This caused `~/.zshrc` to be sourced **twice** in login shells (tmux):

1. **First load** (normal zsh startup): `~/.zshrc` runs, the forge plugin block runs `eval "$(forge zsh plugin)"`, which binds Tab (`^I`) to the `forge-completion` widget. `_FORGE_PLUGIN_LOADED` is set.

2. **Second load** (via `.zprofile` sourcing `.zshrc` again): Oh My Zsh, fzf, and other tools get re-sourced. The fzf completion script (`/usr/share/fzf/completion.zsh`) rebinds Tab to `fzf-completion`. The forge plugin block is **skipped** because `_FORGE_PLUGIN_LOADED` is already set, so forge doesn't get a chance to re-override fzf.

In non-login shells (Kitty/Ghostty direct), `.zprofile` is not sourced, so `.zshrc` runs once and forge loads last — Tab correctly binds to `forge-completion`.

## Fix

Emptied `~/.zprofile`:

```zsh
# .zprofile is intentionally empty.
# Previously sourced ~/.zshrc here, which caused .zshrc to run twice in login
# shells (e.g. inside tmux). That double-load let fzf rebind Tab after forge
# had already bound it, breaking the ":" sentinel completion in tmux.
# ~/.zshrc is sourced automatically by zsh for interactive shells.
```

Zsh automatically sources `~/.zshrc` for interactive shells, so `.zprofile` doesn't need to re-source it. If environment variables are needed for non-interactive login shells (cron, SSH commands), put them directly in `.zprofile`.

## Verification

After applying the fix, run `exec zsh` (or open a new tmux pane) and type `:` followed by Tab — should show Forge's command picker, not zsh's history modifiers.

## Files Modified

- `~/.zprofile` — removed `. ~/.zshrc` line
