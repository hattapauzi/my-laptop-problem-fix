# Forge ZSH P10k Prompt Segments Not Refreshing After `:model`

Fix for Powerlevel10k prompt segments (model name, token count) not refreshing after Forge `:` commands like `:model`, with screen artifacts (blank lines, stray `%`) when forcing a refresh.

## Environment

- Shell: ZSH with Oh My Zsh
- Prompt theme: Powerlevel10k (p10k) with instant prompt and transient prompt (`always`)
- Plugins: forge zsh plugin, zsh-autosuggestions, fzf
- Terminal: Ghostty + tmux
- OS: Arch-based Linux (CachyOS)

## Symptom

After running `:model` to change the Forge session model, the model name and token context count in the p10k prompt did not update until pressing ENTER (which triggers the `precmd` hook). Forcing a refresh caused screen artifacts: multiple blank lines and stray `%` symbols.

## Root Cause

### Why the prompt doesn't update

The Forge `forge-accept-line` widget handles `:` commands by:

1. Detecting the `:` prefix
2. Running `zle redisplay` to clear the current line
3. Dispatching the action (e.g., `_forge_action_session_model`)
4. The action outputs a message (e.g., `Session model set to mimo-v2.5-pro`)
5. Calling `_forge_reset()` to clear the buffer and redraw the prompt

The original `_forge_reset` (from `forge zsh plugin`) does:

```zsh
_forge_reset() {
    local pad="${BUFFERLINES:-1}" _i
    for ((_i=1; _i<pad; _i++)); do print; done
    BUFFER=""
    CURSOR=0
    zle -I
    zle reset-prompt
}
```

The key issue: `zle reset-prompt` (the zsh builtin) re-renders the **existing** `PROMPT`/`RPROMPT` variables as literal text. But p10k bakes segment output into `PROMPT`/`RPROMPT` as **literal strings** during `_p9k_set_prompt` (which runs in the `precmd` hook). The segment functions (like `prompt_forge_code` → `_forge_prompt_info` → `forge zsh rprompt`) are only re-evaluated when `_p9k_set_prompt` runs again — which happens on the next `precmd` cycle (i.e., when you press ENTER).

So `zle reset-prompt` just repaints the stale prompt content with the old model name.

### Why screen artifacts appeared

The first fix attempt called `_p9k_set_prompt` + `_p9k_reset_prompt` **inline** from within the `forge-accept-line` widget. At that point:

1. The action's output (`echo` + `_forge_log`) had moved the terminal cursor **below** the original prompt position.
2. `zle .reset-prompt` redraws the prompt at the **current cursor position**, not back at the original prompt line.
3. The old prompt stayed visible above, with the action output in between, and the new prompt drawn below — creating cascading blank lines.
4. The `%` symbols appeared because the prompt rendering produced partial lines (lines without trailing newlines), which zsh marks with `%` (NOPROMPTINDICATOR).

### Why `_p9k_reset_prompt` didn't help

p10k's own `_p9k_reset_prompt` function (`~/.oh-my-zsh/custom/themes/powerlevel10k/internal/p10k.zsh:7162`) does handle `prompt_subst` and calls `zle -R` for a full redisplay, but it still uses `zle .reset-prompt` internally, which has the same cursor-position problem when called from within a widget after stdout output.

## Fix

Override `_forge_reset` in `~/.zshrc` to use p10k's own **deferred** prompt restore mechanism — the same pattern p10k uses internally after keyboard interrupts:

```zsh
# Re-evaluate p10k prompt segments (including forge_code) after forge actions
# like :model, :commands, :reasoning-effort, etc. Without this, the model and
# token count don't update until the next ENTER (precmd cycle).
#
# We schedule the prompt rebuild via p10k's own _p9k_restore_prompt mechanism
# (a zle -F file-descriptor watch on /dev/null) rather than calling
# _p9k_set_prompt inline. The scheduled callback runs AFTER the current
# widget returns, when zle is in a clean state. This avoids screen artifacts
# (stray "%" markers, blank lines) caused by rebuilding the prompt while
# the terminal cursor is in an intermediate position (below action output).
#
# The inline `zle -I` + `zle reset-prompt` handles immediate screen clearing;
# the scheduled _p9k_restore_prompt handles the full segment re-evaluation.
if (( $+functions[_forge_reset] && $+functions[_p9k_restore_prompt] )); then
  functions[_forge_reset]='
    local pad="${BUFFERLINES:-1}" _i
    for ((_i=1; _i<pad; _i++)); do print; done
    BUFFER=""
    CURSOR=0
    zle -I
    zle reset-prompt
    _p9k__must_restore_prompt=1
    if (( ! _p9k__restore_prompt_fd )); then
      sysopen -o cloexec -ru _p9k__restore_prompt_fd /dev/null
      zle -F $_p9k__restore_prompt_fd _p9k_restore_prompt
    fi
  '
fi
```

### How it works

1. **`zle -I` + `zle reset-prompt`** — handles immediate screen clearing (same as the original plugin behavior). This clears the input buffer and redraws the current prompt.

2. **`_p9k__must_restore_prompt=1`** — sets a flag that tells p10k a prompt restore is pending.

3. **`sysopen -o cloexec -ru _p9k__restore_prompt_fd /dev/null`** — opens `/dev/null` as a file descriptor. The `-o cloexec` ensures it's closed on exec, `-ru` makes it read-only and unbuffered.

4. **`zle -F $_p9k__restore_prompt_fd _p9k_restore_prompt`** — registers the fd with zle's event loop. When zle processes its event queue (after the current widget returns), it calls `_p9k_restore_prompt`.

5. **`_p9k_restore_prompt`** (defined in p10k at `~/.oh-my-zsh/custom/themes/powerlevel10k/internal/p10k.zsh:7916`) properly:
   - Unsets `_p9k__line_finished` (so reset-prompt will work)
   - Sets `_p9k__refresh_reason=restore` (proper context for segment functions)
   - Calls `_p9k_set_prompt` (rebuilds all segments including `forge_code`)
   - Resets `_p9k__refresh_reason`
   - Calls `_p9k_reset_prompt` (which calls `zle .reset-prompt` + `zle -R`)

Because the rebuild is scheduled via zle's event loop (not called inline from the widget), the terminal cursor is at a clean position and p10k can properly manage the prompt transition without artifacts.

The `if (( ! _p9k__restore_prompt_fd ))` guard ensures we don't open duplicate file descriptors if `_forge_reset` is called multiple times before the scheduled callback runs.

## Verification

After applying the fix, run `exec zsh` and type `:model` to select a different model — the prompt should immediately update with the new model name and token count, with no screen artifacts.

## Files Modified

- `~/.zshrc` — added `_forge_reset` override block (after the p10k source line)

## Key References

- Forge ZSH plugin: `forge zsh plugin` (outputs the plugin source)
- Forge ZSH theme: `forge zsh theme` (outputs the theme source)
- p10k `_p9k_set_prompt`: `~/.oh-my-zsh/custom/themes/powerlevel10k/internal/p10k.zsh` (rebuilds PROMPT/RPROMPT from segments)
- p10k `_p9k_reset_prompt`: same file (calls `zle .reset-prompt` + `zle -R`)
- p10k `_p9k_restore_prompt`: same file (deferred prompt restore via `zle -F`)
- Forge prompt segment: `prompt_forge_code` in `~/.p10k.zsh` (calls `_forge_prompt_info` → `forge zsh rprompt`)
