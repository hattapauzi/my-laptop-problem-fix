# My Laptop Problem Fix

My personal documentation of Linux configuration solutions and fixes.

## Contents

### [Hibernation Configuration Guide](./Hibernation_Configuration_Guide.md)
Configure Btrfs swapfile for hibernation on an encrypted Arch-based Linux system (CachyOS/LUKS).

**Topics covered:**
- Creating Btrfs-compatible swapfile
- Calculating resume offset for Btrfs
- Adding resume hooks to mkinitcpio
- Configuring bootloader parameters
- Setting up passwordless hibernate via polkit
- Integration with Dank Material Shell

### [HUAWEI FreeBuds Bluetooth Codec Configuration](./HUAWEI_FreeBuds_Bluetooth_Codec_Configuration.md)
Configure SBC-XQ audio codec for HUAWEI FreeBuds SE 4 ANC on Linux with PipeWire/WirePlumber.

**Topics covered:**
- WirePlumber Bluetooth configuration
- Forcing specific Bluetooth audio codec
- Manual codec switching commands
- Troubleshooting audio issues

### [DNS Configuration Guide](./DNS_Configuration_Guide.md)
Configure system-wide DNS with Cloudflare/Google and DNS over TLS for encrypted queries.

**Topics covered:**
- systemd-resolved configuration
- NetworkManager global DNS defaults
- DNS over TLS for encrypted queries
- Applying DNS to all network connections

### [BedrockOnLinux Flatpak Menu Crash Workaround](./BedrockOnLinux_Flatpak_Crash_Fix.md)
Work around the Minecraft menu crash at `0x140427AB5` by rolling BedrockOnLinux back from `2.1.1`/`native12` to the verified `2.1.0`/`native6` combination.

**Topics covered:**
- Identifying the repeatable Wine page-fault signature
- Installing and verifying the `2.1.0` Flatpak at user scope
- Correcting the rollback package's read-only data mount
- Activating the `native6` managed engine without resetting game data
- Verifying clean launch and reverting after an upstream fix

### [Forge ZSH `:` Commands Not Working in Tmux](./Forge_ZSH_Tmux_Completion_Fix.md)
Fix Forge `:` command completion showing zsh history modifiers instead of Forge's command picker inside tmux.

**Topics covered:**
- `.zprofile` double-sourcing `.zshrc` causing fzf to override forge Tab binding in tmux login shells
- Difference between login shells (tmux) and non-login shells (direct terminal)

### [Forge ZSH P10k Prompt Segments Not Refreshing After `:model`](./Forge_ZSH_P10k_Prompt_Refresh_Fix.md)
Fix Powerlevel10k prompt segments (model name, token count) not refreshing after Forge `:` commands, with screen artifacts when forcing a refresh.

**Topics covered:**
- p10k prompt segments only re-evaluated during `precmd` (not `zle reset-prompt`)
- Scheduling prompt rebuild via p10k's own `_p9k_restore_prompt` deferred mechanism (`zle -F` on `/dev/null`)
- Avoiding screen artifacts (blank lines, `%` markers) from inline prompt rebuilds
