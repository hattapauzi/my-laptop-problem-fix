# Hibernation Swapfile Configuration Guide

Date: 2026-05-07
System: CachyOS / Arch-based Linux, Btrfs on LUKS encryption

## Overview

This guide explains how to configure a Btrfs swapfile for hibernation on an encrypted Arch-based Linux system. The swapfile is used as the hibernation image location.

## Prerequisites

- Arch-based Linux distribution (CachyOS)
- Btrfs filesystem
- LUKS encryption
- A dedicated swapfile partition or subvolume

## Working Hibernation Command

Direct hibernation through the systemd hibernate service:

```bash
sudo systemctl start systemd-hibernate.service
```

Alternative using the backend service directly:

```bash
systemctl start systemd-hibernate.service
```

## Configuration Steps

### 1. Create Btrfs-Compatible Swapfile

Btrfs requires special handling for swapfiles. The swapfile must be created on a NOCOW subvolume or use the Btrfs swapfile utility.

```bash
# Disable existing swap (if any)
sudo swapoff /path/to/swapfile 2>/dev/null || true

# Mark parent directory as NOCOW (required for Btrfs swapfiles)
sudo chattr +C /path/to/swapfile_parent_directory

# Remove existing swapfile
sudo rm -f /path/to/swapfile

# Create new Btrfs swapfile with specified size (e.g., 20GB)
sudo btrfs filesystem mkswapfile --size 20G /path/to/swapfile

# Enable the swapfile
sudo swapon /path/to/swapfile
```

### 2. Add Swapfile to /etc/fstab

Add a persistent swap entry to ensure the swapfile is mounted on boot:

```fstab
/path/to/swapfile none swap defaults 0 0
```

### 3. Calculate Btrfs Resume Offset

For Btrfs filesystems, you must calculate the physical offset of the swapfile for the resume process:

```bash
sudo btrfs inspect-internal map-swapfile -r /path/to/swapfile
```

Save this offset value - it will be needed for the boot parameters.

Example output:
```text
<offset_value>
```

### 4. Update mkinitcpio Hooks

Add the `resume` hook to enable hibernation resume. Edit `/etc/mkinitcpio.conf` and modify the HOOKS array:

```bash
# Add 'resume' hook before 'filesystems'
HOOKS=(base systemd autodetect microcode kms modconf block keyboard sd-vconsole plymouth sd-encrypt resume filesystems)
```

Note: The exact hook names may vary. On some systems, you may need to use just `resume` instead of `sd-resume`.

### 5. Rebuild Initramfs

After modifying mkinitcpio.conf, rebuild the initramfs images:

```bash
sudo mkinitcpio -P
```

Verify the rebuild completed successfully for all installed kernels.

### 6. Configure Boot Loader

Add resume parameters to the kernel command line in your bootloader configuration:

```text
resume=/dev/mapper/luks-<uuid> resume_offset=<offset_value>
```

The parameters needed:
- `resume` - Path to the encrypted root device
- `resume_offset` - The Btrfs offset calculated in step 3

### 7. Update Boot Loader Artifacts

If using Limine bootloader, refresh the boot artifacts:

```bash
sudo limine-update
```

### 8. Configure Desktop Environment (Optional)

If using Dank Material Shell or similar, configure the hibernate action to use the working command.

For Dank Material Shell, edit the settings file and set:

```json
"customPowerActionHibernate": "systemctl start systemd-hibernate.service"
```

Then validate and restart:

```bash
python -m json.tool ~/.config/DankMaterialShell/settings.json >/dev/null
systemctl --user restart dms.service
```

### 9. Configure Polkit for Passwordless Hibernate (Optional)

To allow passwordless hibernation without sudo, create a polkit rule:

Create file: `/etc/polkit-1/rules.d/50-hibernate.rules`

```javascript
// Allow specified user to hibernate without password
polkit.addRule(function(action, subject) {
    if (action.id !== "org.freedesktop.systemd1.manage-units") {
        return polkit.Result.NOT_HANDLED;
    }

    if (subject.user !== "your_username" || !subject.active || !subject.local) {
        return polkit.Result.NOT_HANDLED;
    }

    if (action.lookup("unit") === "systemd-hibernate.service" &&
        action.lookup("verb") === "start") {
        return polkit.Result.YES;
    }

    return polkit.Result.NOT_HANDLED;
});
```

Set proper permissions:

```bash
sudo chmod 0640 /etc/polkit-1/rules.d/50-hibernate.rules
sudo chown root:polkitd /etc/polkit-1/rules.d/50-hibernate.rules
```

Reload polkit:

```bash
sudo systemctl reload polkit.service
```

## Verification Commands

After configuration, verify the setup:

```bash
# Check active swap
swapon --show --bytes

# Check kernel command line
cat /proc/cmdline

# Verify resume parameters
cat /sys/power/resume
cat /sys/power/resume_offset

# Verify resume offset matches swapfile
sudo btrfs inspect-internal map-swapfile -r /path/to/swapfile

# Check systemd-hibernate service status
systemctl status systemd-hibernate.service
```

## Troubleshooting

### Systemctl Hibernate Fails

If `systemctl hibernate` fails but the backend service works, use:

```bash
systemctl start systemd-hibernate.service
```

This directly calls the systemd hibernate service instead of going through logind.

### Resume Fails

Check that:
1. Resume offset is correct (matches swapfile physical offset)
2. Resume device path is correct (matches root device)
3. Initramfs was rebuilt after adding resume hook
4. Bootloader was updated with resume parameters

### Swapfile Not Active After Reboot

Verify the fstab entry and ensure the swapfile path is correct.

## Key Files Modified

- `/etc/fstab` - Added swapfile entry
- `/etc/mkinitcpio.conf` - Added resume hook
- `/boot/limine.conf` - Added resume parameters (for Limine bootloader)
- `/etc/polkit-1/rules.d/` - Added hibernate polkit rule
- `~/.config/DankMaterialShell/settings.json` - Configured hibernate action

## Summary

The hibernation configuration requires:
1. Creating a Btrfs-compatible swapfile
2. Calculating the physical offset for Btrfs
3. Adding resume parameters to boot configuration
4. Configuring the desktop environment for the working hibernate command

## References

- [Arch Wiki - Hibernation](https://wiki.archlinux.org/title/Hibernation)
- [Arch Wiki - Btrfs](https://wiki.archlinux.org/title/Btrfs)
- [Systemd-hibernate](https://man.archlinux.org/man/systemd-hibernate.service.8)

## Changelog

- **2026-05-07**: Initial documentation
  - Configured Btrfs swapfile for hibernation
  - Set up passwordless hibernate via polkit
  - Integrated with Dank Material Shell power menu