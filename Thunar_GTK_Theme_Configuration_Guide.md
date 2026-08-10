# Thunar File Manager Theme Configuration & Replication Guide

Guide to identifying the GTK theme, icon theme, and color scheme used by the **Thunar file manager** and applying the identical configuration to another Linux system.

---

## 1. Overview & Current System Theme

Thunar is a GTK-based file manager (part of the XFCE desktop suite). On this system (CachyOS Linux), the active theme configuration is:

| Component | Active Setting |
| :--- | :--- |
| **GTK Theme** | `catppuccin-mocha-rosewater-standard+default` |
| **Theme Origin** | Catppuccin GTK Theme (Mocha Dark variant, Rosewater accent) |
| **Icon Theme** | `Adwaita` |
| **Cursor Theme** | `Adwaita` |
| **Color Scheme** | `prefer-dark` (Dark Mode) |
| **GTK Font** | `Sans 10` |
| **Installed Package** | `catppuccin-gtk-theme-mocha` (Arch/AUR) |
| **Theme Location** | `/usr/share/themes/catppuccin-mocha-rosewater-standard+default` |

---

## 2. Inspecting GTK Theme Settings

To inspect which GTK theme Thunar or other GTK apps are currently using on any Linux system, run the following commands:

```bash
# 1. Check GTK 3.0 / 4.0 configuration files
cat ~/.config/gtk-3.0/settings.ini
cat ~/.config/gtk-4.0/settings.ini

# 2. Check GSettings (used by GNOME, Wayland compositors like Niri/Sway/Hyprland)
gsettings get org.gnome.desktop.interface gtk-theme
gsettings get org.gnome.desktop.interface icon-theme
gsettings get org.gnome.desktop.interface color-scheme

# 3. Check XFCE settings (if running XFCE desktop)
xfconf-query -c xsettings -p /Net/ThemeName
xfconf-query -c xsettings -p /Net/IconThemeName

# 4. Check active environment variables
echo $GTK_THEME
```

---

## 3. Replicating the Theme on Another System

### Step 1: Install the Theme Files

#### Option A: Arch Linux / Manjaro / CachyOS (AUR)
Install the theme package via AUR helper:
```bash
yay -S catppuccin-gtk-theme-mocha
```

#### Option B: Ubuntu / Debian / Fedora / Other Distros (Manual Download)
1. Download the latest pre-compiled archive from the official [Catppuccin GTK Releases](https://github.com/catppuccin/gtk/releases).
2. Extract `catppuccin-mocha-rosewater-standard+default` to your target directory:
   * **User-scope (recommended):** `~/.themes/`
     ```bash
     mkdir -p ~/.themes
     tar -xf catppuccin-mocha-rosewater-standard+default.tar.xz -C ~/.themes/
     ```
   * **System-wide:** `/usr/share/themes/`
     ```bash
     sudo tar -xf catppuccin-mocha-rosewater-standard+default.tar.xz -C /usr/share/themes/
     ```

---

### Step 2: Configure GTK Settings

#### 1. Via GTK Config Files (Universal method)
Create or update `~/.config/gtk-3.0/settings.ini` and `~/.config/gtk-4.0/settings.ini`:

```ini
[Settings]
gtk-theme-name=catppuccin-mocha-rosewater-standard+default
gtk-icon-theme-name=Adwaita
gtk-font-name=Sans 10
gtk-application-prefer-dark-theme=1
```

#### 2. Via GSettings (GNOME / Wayland Compositors)
Run the following commands in your shell:

```bash
gsettings set org.gnome.desktop.interface gtk-theme 'catppuccin-mocha-rosewater-standard+default'
gsettings set org.gnome.desktop.interface icon-theme 'Adwaita'
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
```

#### 3. Via XFCE Settings Manager
If using XFCE desktop environment:
```bash
xfconf-query -c xsettings -p /Net/ThemeName -s "catppuccin-mocha-rosewater-standard+default"
xfconf-query -c xsettings -p /Net/IconThemeName -s "Adwaita"
```

#### 4. Via Environment Variable (Shell Override)
Add to your shell profile (`~/.bashrc`, `~/.zshrc`, or `~/.profile`):
```bash
export GTK_THEME=catppuccin-mocha-rosewater-standard+default:dark
```

#### 5. Via Graphical Tools (GUI)
You can also select the theme visually using GTK appearance managers:
* **LXAppearance:** Select `catppuccin-mocha-rosewater-standard+default` under the Widget tab.
* **XFCE Appearance:** Select `catppuccin-mocha-rosewater-standard+default` under Style.
* **GNOME Tweaks:** Select `catppuccin-mocha-rosewater-standard+default` under Legacy Applications.

---

## 4. Verification

After configuring, verify the changes by launching Thunar from terminal or app launcher:
```bash
thunar &
```

To ensure GTK applications read the updated theme without rebooting:
```bash
killall thunar 2>/dev/null
thunar &
```
