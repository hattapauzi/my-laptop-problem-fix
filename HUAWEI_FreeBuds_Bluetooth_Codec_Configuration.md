# HUAWEI FreeBuds SE 4 ANC - Bluetooth Audio Codec Configuration Guide

## Overview

This document describes how to configure the HUAWEI FreeBuds SE 4 ANC Bluetooth earbuds to use a specific audio codec (SBC-XQ) by default on Linux with PipeWire/WirePlumber audio stack.

## Problem Statement

The user experienced issues with audio not coming through their HUAWEI Bluetooth earbuds when connected to Linux. The default AAC codec was not working properly, but switching to SBC-XQ resolved the issue.

## System Information

- **Distribution**: Arch Linux (CachyOS)
- **Audio Stack**: PipeWire 1.6.4 + WirePlumber + BlueZ
- **Device**: HUAWEI FreeBuds SE 4 ANC
- **Device MAC Address**: `34:D6:93:51:AF:F6`
- **Device Bluetooth Card Name**: `bluez_card.34_D6_93_51_AF_F6`

## Available Audio Codecs

The HUAWEI FreeBuds SE 4 ANC supports the following Bluetooth audio codecs:

| Codec | Profile Name | Priority | Description |
|-------|--------------|----------|-------------|
| **AAC** | `a2dp-sink` | 131 | Advanced Audio Coding (default) |
| **SBC-XQ** | `a2dp-sink-sbc_xq` | 129 | High Quality SBC variant |
| **SBC** | `a2dp-sink-sbc` | 130 | Standard SBC |

## Configuration File

Create the following WirePlumber configuration file to set SBC-XQ as the default codec:

### File Location
```
~/.config/wireplumber/wireplumber.conf.d/51-bluetooth-a2dp-fix.conf
```

### File Content
```conf
## Disable WirePlumber auto-switching to Headset Profile (HFP)
## This prevents unwanted profile switching that can cause audio issues
wireplumber.settings = {
  bluetooth.autoswitch-to-headset-profile = false
}

## Configure BlueZ monitor properties for codec selection
monitor.bluez.properties = {
  ## Only enable A2DP sink to avoid profile switching issues
  bluez5.roles = [ a2dp_sink a2dp_source ]
  
  ## Enable connection info for codec negotiation
  api.bluez5.connection-info = true
}

## Rules to force SBC-XQ codec for specific devices
monitor.bluez.rules = [
  {
    matches = [
      {
        ## HUAWEI FreeBuds SE 4 ANC MAC address
        device.name = "~bluez_card.34_D6_93_51_AF_F6"
      }
    ]
    actions = {
      update-props = {
        ## Prefer SBC-XQ codec (higher quality SBC variant)
        ## Available profiles: sbc, sbc_xq, aac, ldac, aptx, etc.
        bluez5.codecs = "[ sbc_xq sbc ]"
        
        ## Set default profile to SBC-XQ
        bluez5.profile = "a2dp-sink-sbc_xq"
      }
    }
  }
]
```

## Installation Steps

### 1. Create the configuration directory (if it doesn't exist)
```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
```

### 2. Create the configuration file
```bash
cat > ~/.config/wireplumber/wireplumber.conf.d/51-bluetooth-a2dp-fix.conf << 'EOF'
## Disable WirePlumber auto-switching to Headset Profile (HFP)
wireplumber.settings = {
  bluetooth.autoswitch-to-headset-profile = false
}

## Configure BlueZ monitor properties for codec selection
monitor.bluez.properties = {
  ## Only enable A2DP sink to avoid profile switching issues
  bluez5.roles = [ a2dp_sink a2dp_source ]
  
  ## Enable connection info for codec negotiation
  api.bluez5.connection-info = true
}

## Rules to force SBC-XQ codec for specific devices
monitor.bluez.rules = [
  {
    matches = [
      {
        ## HUAWEI FreeBuds SE 4 ANC MAC address
        device.name = "~bluez_card.34_D6_93_51_AF_F6"
      }
    ]
    actions = {
      update-props = {
        ## Prefer SBC-XQ codec (higher quality SBC variant)
        bluez5.codecs = "[ sbc_xq sbc ]"
        
        ## Set default profile to SBC-XQ
        bluez5.profile = "a2dp-sink-sbc_xq"
      }
    }
  }
]
EOF
```

### 3. Restart WirePlumber to apply changes
```bash
systemctl --user restart wireplumber
```

### 4. Reconnect the Bluetooth device
```bash
bluetoothctl disconnect 34:D6:93:51:AF:F6
bluetoothctl connect 34:D6:93:51:AF:F6
```

### 5. Verify the codec is active
```bash
pactl list sinks | grep -E "bluez_output.34_D6_93_51_AF_F6|api.bluez5.codec"
```

Expected output:
```
Name: bluez_output.34_D6_93_51_AF_F6.1
api.bluez5.codec = "sbc_xq"
```

## Manual Codec Switching

You can manually switch between codecs at any time using `pactl`:

### Switch to SBC-XQ (High Quality SBC)
```bash
pactl set-card-profile bluez_card.34_D6_93_51_AF_F6 a2dp-sink-sbc_xq
```

### Switch to AAC
```bash
pactl set-card-profile bluez_card.34_D6_93_51_AF_F6 a2dp-sink
```

### Switch to Standard SBC
```bash
pactl set-card-profile bluez_card.34_D6_93_51_AF_F6 a2dp-sink-sbc
```

## Quick Commands Reference

### View Current Codec
```bash
pactl list sinks | grep api.bluez5.codec
```

### List Available Codecs
```bash
pactl send-message /card/bluez_card.34_D6_93_51_AF_F6/bluez list-codecs
```

### View Device Status
```bash
bluetoothctl info 34:D6:93_51:AF:F6
```

### Check All Profiles
```bash
pactl list cards | grep -A20 "HUAWEI"
```

### Set Device as Default Output
```bash
pactl set-default-sink bluez_output.34_D6_93_51_AF_F6.1
```

### Restart Audio Services
```bash
systemctl --user restart wireplumber pipewire pipewire-pulse
```

## Troubleshooting

### Audio Still Not Working

1. **Check if device is connected:**
   ```bash
   bluetoothctl info 34:D6:93:51:AF_F6 | grep Connected
   ```

2. **Verify WirePlumber is running:**
   ```bash
   systemctl --user status wireplumber
   ```

3. **Check for conflicting PulseAudio:**
   ```bash
   pacman -Q pulseaudio pipewire-pulse
   ```
   If PulseAudio is installed alongside pipewire-pulse, remove it:
   ```bash
   sudo pacman -R pulseaudio pulseaudio-bluetooth
   ```

4. **Check Spotify playback status:**
   ```bash
   dbus-send --print-reply --dest=org.mpris.MediaPlayer2.spotify /org/mpris/MediaPlayer2 org.freedesktop.DBus.Properties.Get string:org.mpris.MediaPlayer2.Player string:PlaybackStatus
   ```

### Codec Not Persisting After Reboot

If the codec setting doesn't persist after a reboot:

1. Ensure the configuration file is in the correct location:
   ```bash
   cat ~/.config/wireplumber/wireplumber.conf.d/51-bluetooth-a2dp-fix.conf
   ```

2. Check WirePlumber logs for errors:
   ```bash
   journalctl --user -u wireplumber -n 50
   ```

3. Manually set the profile after reboot:
   ```bash
   pactl set-card-profile bluez_card.34_D6_93_51_AF_F6 a2dp-sink-sbc_xq
   ```

## Understanding the Configuration

### What `bluez5.roles` Does
Controls which Bluetooth profiles are enabled. Setting only `[ a2dp_sink a2dp_source ]` prevents WirePlumber from automatically switching to HFP/HSP profiles which can cause audio quality issues.

### What `bluez5.codecs` Does
Defines the priority order of codecs. The device will try codecs in the order specified. `[ sbc_xq sbc ]` means it will try SBC-XQ first, then fall back to standard SBC if SBC-XQ fails.

### What `bluez5.profile` Does
Sets the default profile when connecting. `a2dp-sink-sbc_xq` tells the system to use the SBC-XQ profile by default.

## Alternative: System-Wide Configuration

For system-wide settings (affects all users), create the config in:
```
/etc/wireplumber/wireplumber.conf.d/51-bluetooth-a2dp-fix.conf
```

Note: User-specific configs in `~/.config/wireplumber/` take precedence over system-wide configs.

## References

- [Arch Wiki - Bluetooth Headset](https://wiki.archlinux.org/title/Bluetooth_headset)
- [Arch Forums - Force AAC Codec](https://bbs.archlinux.org/viewtopic.php?id=289177)
- [PipeWire Wiki](https://gitlab.freedesktop.org/pipewire/pipewire/-/wikis/home)
- [WirePlumber GitLab](https://gitlab.freedesktop.org/pipewire/wireplumber)

## Changelog

- **2026-05-10**: Initial documentation created
  - Configured SBC-XQ as default codec for HUAWEI FreeBuds SE 4 ANC
  - Disabled auto-switching to HFP profile
  - Documented manual codec switching commands