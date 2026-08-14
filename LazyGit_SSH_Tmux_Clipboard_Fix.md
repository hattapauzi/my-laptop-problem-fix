# LazyGit Clipboard Copy Fails Through SSH and Tmux

Fix for LazyGit showing this error when copying a commit hash or other value from a remote tmux session:

```text
No clipboard utilities available. Please install xsel, xclip, wl-clipboard or
Termux:API add-on for termux-clipboard-get/set.
```

## Environment

- LazyGit running on a headless Linux system
- Connection through SSH
- LazyGit opened inside tmux, including when launched from LazyVim
- Local terminal and clipboard support OSC 52

## Symptom

Yanking text directly from Neovim successfully updates the local clipboard, but copying a commit attribute from LazyGit fails with the clipboard-utilities error.

This difference is important: Neovim and LazyGit use independent clipboard implementations. A working Neovim clipboard provider does not configure LazyGit's clipboard provider.

## Root Cause

LazyGit's default Linux clipboard detection searches for host-side graphical clipboard programs such as `wl-copy`, `xclip`, or `xsel`.

Those programs are not the correct solution on a headless remote system:

- A Wayland or X11 clipboard belongs to the local graphical session, not the remote shell.
- Installing a clipboard package remotely does not provide a usable graphical display.
- Neovim already works because it sends copied text to the local terminal with OSC 52.

LazyGit supports a custom copy command, so it can use the same terminal-based approach.

## Fix

Add the following to `~/.config/lazygit/config.yml` on the system where LazyGit runs:

```yaml
os:
  copyToClipboardCmd: >
    if [[ "$TERM" =~ ^(screen|tmux) ]]; then
      printf "\\033Ptmux;\\033\\033]52;c;$(printf %s {{text}} | base64 -w 0)\\007\\033\\\\" > /dev/tty
    else
      printf "\\033]52;c;$(printf %s {{text}} | base64 -w 0)\\007" > /dev/tty
    fi
```

The command:

1. Encodes the copied text as Base64.
2. Sends it to the terminal with OSC 52.
3. Wraps the sequence in tmux passthrough escapes when `$TERM` starts with `screen` or `tmux`.
4. Uses plain OSC 52 when LazyGit is not running inside tmux.

## Tmux Requirements

Ensure the tmux configuration contains:

```tmux
set -g set-clipboard on
set -g allow-passthrough on
```

Reload the configuration if these options were newly added:

```bash
tmux source-file ~/.config/tmux/tmux.conf
```

The terminal emulator must also permit OSC 52 clipboard writes. If Neovim OSC 52 yanks already reach the local clipboard, this terminal-side requirement is satisfied.

## Apply the Change

Close and reopen LazyGit after editing its configuration. LazyGit reads the configuration at startup; restarting Neovim or the tmux server is not required when the tmux options are already active.

## Verification

1. Open LazyGit inside the remote tmux session.
2. Focus the commits panel.
3. Use LazyGit's copy action on a commit and choose an attribute such as the commit hash.
4. Paste into a local application.

The copied commit value should appear without the missing-clipboard-utilities error.

The fix was also validated by confirming that:

- LazyGit starts without a YAML or unknown-field configuration error.
- The tmux branch emits a correctly wrapped OSC 52 sequence.
- The direct-terminal branch emits a plain OSC 52 sequence.
- A sample commit hash and subject survive command substitution, Base64 encoding, and decoding unchanged.

## Files Modified

- `~/.config/lazygit/config.yml` — added the OSC 52 copy command
- `~/.config/tmux/tmux.conf` — requires clipboard and passthrough options when not already configured
