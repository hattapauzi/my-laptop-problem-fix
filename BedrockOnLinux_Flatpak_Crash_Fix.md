# BedrockOnLinux Flatpak Menu Crash Workaround

Date: 2026-07-30  
Status: Temporary rollback workaround for an upstream regression

## Overview

BedrockOnLinux `2.1.1`, using the managed WineGDK engine `wow64-archs-native12`, allowed Minecraft Bedrock `1.26.33.1` to start and create its window, but the game then crashed during account/menu initialization.

The repeatable crash signature in `proton.log` was:

```text
Unhandled page fault
0x140427AB5
```

A user-scoped rollback to BedrockOnLinux `2.1.0`, which pins `wow64-archs-native6`, resolved the crash. Minecraft passed the same signed-in account and menu path, remained running for more than five minutes, and shut down cleanly when its window was closed.

This is a workaround rather than a confirmed upstream root-cause fix. Keep the rollback only until a newer BedrockOnLinux/WineGDK release resolves the affected menu-crash reports.

## Affected Setup

- **Distribution**: CachyOS / Arch-based Linux
- **Package**: BedrockOnLinux Flatpak
- **Application ID**: `io.github.wyze3306.BedrockOnLinux`
- **Broken combination**: BedrockOnLinux `2.1.1` + `wow64-archs-native12`
- **Tested Minecraft version**: `1.26.33.1`
- **Working combination**: BedrockOnLinux `2.1.0` + `wow64-archs-native6`

## Symptoms

Run:

```bash
flatpak run io.github.wyze3306.BedrockOnLinux play
```

Minecraft may:

1. Open normally.
2. Reach the loading screen or menu transition.
3. Exit within approximately 90 seconds.
4. Leave `proton.log` with an unhandled page fault at `0x140427AB5`.

The log is stored at:

```text
~/.var/app/io.github.wyze3306.BedrockOnLinux/data/bedrock-on-linux/logs/proton.log
```

Check for the specific signature with:

```bash
grep -E "Unhandled page fault|140427AB5" \
  ~/.var/app/io.github.wyze3306.BedrockOnLinux/data/bedrock-on-linux/logs/proton.log
```

## Findings

The following changes did **not** resolve this crash:

- Forcing setup again while remaining on `wow64-archs-native12`.
- Increasing the Minecraft executable stack reserve from 1 MiB to 16 MiB. This delayed the failure but did not make the menu path stable.
- Switching from Vulkan/vkd3d to the legacy WineD3D renderer.
- Overriding the native `cryptbase.dll` with Wine's built-in implementation.

Graphics DGC warnings and `AllocStateNoCompletion` trace messages also occurred near the failure, but they were not sufficient crash indicators. The successful `native6` run still contained routine async-state traces. The discriminating failure was the unhandled page fault at `0x140427AB5`.

Recent upstream reports showed the same crash family across different GPU vendors. One report also established a regression boundary between BedrockOnLinux `2.1.0` and `2.1.1`, making the launcher/managed-engine rollback the most targeted available workaround.

## Workaround

These steps preserve the existing system-scoped `2.1.1` installation and install `2.1.0` at user scope. Flatpak selects the user-scoped installation when the normal launch command is used.

Account data, installed Minecraft data, and worlds remain under the existing private Flatpak data directory. Do not use `--delete-data` when installing or removing the rollback.

### 1. Confirm the Current Installation

```bash
flatpak info --system io.github.wyze3306.BedrockOnLinux
```

The repaired system originally had the crashing package installed at system scope. If a user-scoped copy already exists, inspect it before proceeding:

```bash
flatpak info --user io.github.wyze3306.BedrockOnLinux
```

### 2. Download BedrockOnLinux 2.1.0

```bash
curl -fL --retry 3 \
  -o /tmp/BedrockOnLinux-2.1.0-x86_64.flatpak \
  https://github.com/Wyze3306/BedrockOnLinux/releases/download/v2.1.0/BedrockOnLinux-2.1.0-x86_64.flatpak
```

Verify the exact bundle used for this workaround:

```bash
printf '%s  %s\n' \
  '5682bc85c65b5edc1bd458e7daad3aa78ca781f367d96fc2cc7953e9e9fc6975' \
  '/tmp/BedrockOnLinux-2.1.0-x86_64.flatpak' \
  | sha256sum --check
```

Expected result:

```text
/tmp/BedrockOnLinux-2.1.0-x86_64.flatpak: OK
```

If GitHub CLI is installed, also verify the artifact attestation:

```bash
gh attestation verify \
  /tmp/BedrockOnLinux-2.1.0-x86_64.flatpak \
  --repo Wyze3306/BedrockOnLinux
```

### 3. Install the Rollback at User Scope

```bash
flatpak install --user --noninteractive --bundle \
  /tmp/BedrockOnLinux-2.1.0-x86_64.flatpak
```

Confirm that the user-scoped application is selected and reports the `2.1.0` changelog:

```bash
flatpak info --user io.github.wyze3306.BedrockOnLinux
flatpak run --user io.github.wyze3306.BedrockOnLinux changelog
```

### 4. Correct the 2.1.0 Filesystem Override

The `2.1.0` Flatpak metadata exposes the legacy `xdg-data/bedrock-on-linux` directory read-only. That mount can overlay the app's writable private data directory and prevent setup or GPU-safety state updates.

Deny only that inherited host-directory mount:

```bash
flatpak override --user \
  --nofilesystem=xdg-data/bedrock-on-linux \
  io.github.wyze3306.BedrockOnLinux
```

Verify that the private data directory is writable:

```bash
flatpak run --user --command=sh \
  io.github.wyze3306.BedrockOnLinux \
  -c 'test -w "$XDG_DATA_HOME/bedrock-on-linux" && echo writable'
```

Expected result:

```text
writable
```

This override does not grant broader host access. It removes the conflicting legacy bind and leaves Flatpak's normal private application storage available.

### 5. Check Launcher Health

```bash
flatpak run --user io.github.wyze3306.BedrockOnLinux doctor
```

Expected final line:

```text
ok     System ready.
```

Do not launch if the doctor reports an active GPU-safety marker. First ensure no BedrockOnLinux, Wine, or UMU process is still running. Avoid wrapping the game in `timeout`, because forced termination can prevent normal marker cleanup.

### 6. Install the Native6 Managed Engine

Run setup through the rollback launcher:

```bash
flatpak run --user io.github.wyze3306.BedrockOnLinux \
  setup --mc 1.26.33.1
```

The `native6` engine archive is approximately 866 MiB. Allow setup to finish; do not interrupt it while it is downloading, verifying, or activating the engine.

Confirm the selected engine:

```bash
jq -r '.winegdk_built' \
  ~/.var/app/io.github.wyze3306.BedrockOnLinux/data/bedrock-on-linux/settings.json
```

Expected result:

```text
prebuilt:wow64-archs-native6
```

### 7. Launch Minecraft

Use the user-scoped rollback explicitly during verification:

```bash
flatpak run --user io.github.wyze3306.BedrockOnLinux play
```

After verification, the ordinary command also selects the user-scoped installation:

```bash
flatpak run io.github.wyze3306.BedrockOnLinux play
```

## Verification

A successful test should meet all of these conditions:

1. Minecraft reaches the account/menu path that previously crashed.
2. The game remains open beyond the previous failure window.
3. The current log has no `0x140427AB5` page fault.
4. Closing the Minecraft window allows the launcher and Wine processes to exit normally.
5. The launcher health check returns `System ready` after shutdown.

Check the crash signature:

```bash
if grep -Eq "Unhandled page fault|140427AB5" \
  ~/.var/app/io.github.wyze3306.BedrockOnLinux/data/bedrock-on-linux/logs/proton.log; then
  echo "crash signature found"
else
  echo "crash signature absent"
fi
```

Then check launcher state:

```bash
flatpak run --user io.github.wyze3306.BedrockOnLinux doctor
```

On the repaired system, Minecraft remained active for more than five minutes, completed signed-in user and store-license operations, and exited cleanly only after its window was closed.

## Reverting the Workaround

Once upstream publishes and verifies a fix, remove only the user-scoped rollback and its override:

```bash
flatpak override --user --reset io.github.wyze3306.BedrockOnLinux
flatpak uninstall --user io.github.wyze3306.BedrockOnLinux
```

Do **not** add `--delete-data`; the existing account, game, and world data should remain available to the system-scoped installation.

Update and prepare the system-scoped application again:

```bash
flatpak update --system io.github.wyze3306.BedrockOnLinux
flatpak run --system io.github.wyze3306.BedrockOnLinux \
  setup --mc 1.26.33.1 --force
flatpak run --system io.github.wyze3306.BedrockOnLinux play
```

Only revert after confirming that the newer managed engine no longer produces the page fault on the actual menu/gameplay path.

## References

- [BedrockOnLinux releases](https://github.com/Wyze3306/BedrockOnLinux/releases)
- [BedrockOnLinux 2.1.0](https://github.com/Wyze3306/BedrockOnLinux/releases/tag/v2.1.0)
- [Issue #115: Unhandled Page Fault then Crash on Startup](https://github.com/Wyze3306/BedrockOnLinux/issues/115)
- [Issue #116: 2.1.0 to 2.1.1 regression report](https://github.com/Wyze3306/BedrockOnLinux/issues/116)
- [Issue #129](https://github.com/Wyze3306/BedrockOnLinux/issues/129)
- [Issue #132](https://github.com/Wyze3306/BedrockOnLinux/issues/132)

## Changelog

- **2026-07-30**: Initial documentation
  - Recorded the `2.1.1`/`native12` menu-crash signature.
  - Documented the verified user-scoped `2.1.0`/`native6` rollback.
  - Added filesystem-override correction, validation, and rollback steps.
