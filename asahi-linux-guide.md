# Asahi Linux on MacBook — Setup & Configuration Guide

> **Disclaimer:** This has been written by AI, all the research and experiment were carried out by me, however to not brick your laptop, I had AI re-do each section in a much clearer manner than I could provide. This is a personal side project. I hold no accountability for any loss of data or harm to devices. Use at your own risk — but I hope this helps you get the most out of Asahi Linux on your MacBook! :)

---

## How to Use This Guide

- 🟢 **Green** — Apps that need to be opened
- 🟣 **Purple** — Commands to run in your terminal
- 🟡 **Yellow** — Text to paste
- 🔵 **Blue** — Buttons to press
- 🔴 **Red** — Steps with a risk of breaking something — proceed carefully

---

## Table of Contents

1. [Setting It Up](#1-setting-it-up)
2. [Initial Configuration](#2-initial-configuration)
3. [Audio Config](#3-audio-config)
4. [yt-dlp](#4-yt-dlp)
5. [Music](#5-music)
6. [Video Player](#6-video-player)
7. [Keybinds](#7-keybinds)
8. [Fonts](#8-fonts)
9. [Widgets](#9-widgets)
10. [General Misc](#10-general-misc)
11. [Firefox Configuration](#11-firefox-configuration)
12. [Terminal](#12-terminal)
13. [Games](#13-games)
14. [Hardware Acceleration](#14-hardware-acceleration)
15. [Other Languages](#15-other-languages)
16. [Display & HiDPI Scaling](#16-display--hidpi-scaling)
17. [Battery Life & Power Management](#17-battery-life--power-management)
18. [Touchpad & Gestures](#18-touchpad--gestures)
19. [KDE Theming](#19-kde-theming)
20. [Flatpak](#20-flatpak)
21. [Development Environment](#21-development-environment)
22. [Containers (Podman / Docker)](#22-containers-podman--docker)
23. [SSH Setup](#23-ssh-setup)
24. [Backups with Snapper](#24-backups-with-snapper)
25. [Screen Recording & Screenshots](#25-screen-recording--screenshots)
26. [Printing & Scanning](#26-printing--scanning)
27. [VPN](#27-vpn)
28. [Wayland Tips & Quirks](#28-wayland-tips--quirks)
29. [Switching Back to macOS (Dual Boot)](#29-switching-back-to-macos-dual-boot)
30. [Removing Asahi Linux](#30-removing-asahi-linux)

---

## 1. Setting It Up

Run the official Asahi Linux installer from your macOS terminal:

```bash
curl https://alx.sh | sh
```

Follow the on-screen prompts. The installer will guide you through resizing your macOS partition and installing Fedora Asahi Remix (recommended) or Asahi Linux Minimal. You will need to reboot into recovery mode once during the process — the installer will tell you exactly when and how.

> ⚠️ **Back up your data before starting.** Resizing partitions carries a small but real risk of data loss.

After installation completes and you boot into your new system for the first time, run a full update before anything else:

```bash
sudo dnf update -y
```

---

## 2. Initial Configuration

After booting into the new OS, apply the following changes.

### GRUB Configuration

Edit the GRUB config file:

```bash
sudo nano /etc/default/grub
```

Set the file contents to:

```
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_DISABLE_RECOVERY=true
GRUB_DISABLE_OS_PROBER=true
GRUB_CMDLINE_LINUX_DEFAULT="rhgb quiet rootflags=subvol=root zswap.enabled=1 zswap.compressor=lz4 zswap.max_pool_percent=35 zswap.zpool=z3fold hid_apple.swap_ctrl_cmd=1"
GRUB_DISTRIBUTOR="Fedora Linux Asahi Remix"
GRUB_GFXMODE=auto
GRUB_TERMINAL_INPUT="console"
GRUB_TERMINAL_OUTPUT="console"
GRUB_TIMEOUT=1
GRUB_TIMEOUT_STYLE=hidden
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`), then apply the changes:

```bash
sudo grubby --update-kernel=ALL --args="$(grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub | cut -d'"' -f2)"
```

### Why these zswap settings?

zswap compresses memory pages before they hit the swap partition, which dramatically reduces swap I/O. Using `lz4` as the compressor is much faster and runs cooler than the default `lzo` — if your Mac feels warm after enabling zswap, this is why. You can verify the active compressor at any time:

```bash
cat /sys/module/zswap/parameters/compressor
```

---

## 3. Audio Config

Speaker and headphone audio on Asahi Linux works out of the box via the `speakersafetyd` daemon included in Fedora Asahi. However, Bluetooth audio can stutter on MacBook Air M2 models (and others with BCM43XX chips).

### Bluetooth A2DP Stutter Fix

This fix is from [christian-korneck/asahi-bt-a2dp-fix](https://github.com/christian-korneck/asahi-bt-a2dp-fix). It installs a small systemd service that resolves the stuttering.

```bash
git clone https://github.com/christian-korneck/asahi-bt-a2dp-fix.git
cd asahi-bt-a2dp-fix
sudo ./install.sh
```

> 💡 If you don't want the latency fix (a separate thing from stutter), open `install.sh` in a text editor and comment out the last section before running it.

### Quick Temporary Fix (no install needed)

If you just want to test the fix without committing to the install, add this alias to your shell config:

```bash
alias fixbt='handle=$(hcitool con | grep -oP "handle \K[0-9]+"); sudo hcitool cmd 0x3f 0x57 $(printf 0x%02X $handle) 0x00 0x01'
```

Run `fixbt` after connecting your Bluetooth device. This fix does not survive a reboot.

---

## 4. yt-dlp

### Installation

```bash
sudo dnf install unzip
curl -fsSL https://deno.land/install.sh | sh
export DENO_INSTALL="$HOME/.deno"
export PATH="$DENO_INSTALL/bin:$PATH"
```

To make the PATH change permanent, add those two `export` lines to `~/.bashrc` (or `~/.config/fish/config.fish` if using Fish).

### Common Commands

| Purpose | Command |
|---|---|
| Best quality video + audio | `yt-dlp "URL"` |
| Audio only (MP3) | `yt-dlp -x --audio-format mp3 "URL"` |
| List available formats | `yt-dlp -F "URL"` |
| Download 1080p + best audio (MKV) | `yt-dlp -f "bestvideo[height<=1080]+bestaudio" --merge-output-format mkv "URL"` |
| Best MP4 video | `yt-dlp -f "bv*[ext=mp4]+ba[ext=m4a]/b[ext=mp4]" "URL"` |
| Download playlist (indexed filenames) | `yt-dlp -o "%(playlist_index)s - %(title)s.%(ext)s" "PLAYLIST_URL"` |
| With embedded subtitles | `yt-dlp --write-subs --sub-lang en --embed-subs "URL"` |
| Subtitles only (no video) | `yt-dlp --write-subs --sub-lang en --skip-download "URL"` |
| Channel with archive (skip already downloaded) | `yt-dlp --download-archive archive.txt "CHANNEL_URL"` |
| Update yt-dlp | `yt-dlp -U` |
| Audio playlist (MP3) | `yt-dlp -x --audio-format mp3 --audio-quality 0 --yes-playlist "PLAYLIST_URL"` |

### Audio Playlist Script (with metadata + thumbnail)

Save this as a shell script (e.g. `dlmusic.sh`) and run it with a playlist URL as the argument:

```bash
yt-dlp -x --audio-format mp3 --audio-quality 0 \
  --embed-thumbnail --add-metadata \
  -o "downloads/%(title)s.%(ext)s" \
  -a "$1"
```

Usage: `./dlmusic.sh "PLAYLIST_URL"`

---

## 5. Music

### Winedive
A Wine-based wrapper for running Windows music apps natively. Useful if you have a specific Windows-only player you rely on.

### Local Files with yt-dlp
Download your own library from YouTube Music or Spotify (via `spotdl`) and play locally. No streaming subscriptions, no DRM.

Download a full playlist as indexed MP3s:

```bash
yt-dlp -x --audio-format mp3 -i \
  -o "%(playlist_index)s-%(title)s.%(ext)s" \
  "YOUR_PLAYLIST_URL"
```

### Tunemymusic
A web tool at [tunemymusic.com](https://www.tunemymusic.com) for transferring playlists between streaming services (Spotify → YouTube Music, etc.). Useful if you're migrating your library before downloading locally.

---

## 6. Video Player

**Recommended:** [MPV](https://mpv.io/)

MPV is a lightweight, hardware-accelerated video player that works exceptionally well on Asahi Linux.

```bash
sudo dnf install mpv
```

### Enable Hardware Acceleration in MPV

By default MPV does not have hardware decoding enabled. Create or edit the config file:

```bash
mkdir -p ~/.config/mpv
nano ~/.config/mpv/mpv.conf
```

Add the following:

```
hwdec=auto
vo=gpu
gpu-api=vulkan
```

To verify hardware decoding is active when you play a video:

```bash
mpv --hwdec=auto /path/to/video.mp4
```

Look for a line in the output like `Using hardware decoding (vaapi or vulkan-v)`. CPU usage on high-resolution video should drop noticeably compared to software decoding.

### Useful MPV Keybinds

| Key | Action |
|---|---|
| `Space` | Play / Pause |
| `Left` / `Right` | Seek ±5 seconds |
| `[` / `]` | Decrease / increase playback speed |
| `m` | Mute |
| `f` | Toggle fullscreen |
| `s` | Screenshot |
| `q` | Quit |

---

## 7. Keybinds

### Swap Cmd ↔ Ctrl (Mac-style muscle memory)

If you're coming from macOS, you'll be used to using `Cmd` for copy/paste/shortcuts rather than `Ctrl`. You can remap this at the kernel level so it works system-wide.

Edit the GRUB config:

```bash
sudo nano /etc/default/grub
```

Find `GRUB_CMDLINE_LINUX_DEFAULT` and add one of the following inside the quotes:

| Swap | Argument |
|---|---|
| Cmd (Meta) ↔ Control | `hid_apple.swap_ctrl_cmd=1` |
| Option (Alt) ↔ Cmd (Meta) | `hid_apple.swap_opt_cmd=1` |

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`), then apply:

```bash
sudo grubby --update-kernel=ALL --args="hid_apple.swap_ctrl_cmd=1"
```

Reboot for the change to take effect. After rebooting, `Cmd+C` will behave like `Ctrl+C`, `Cmd+V` will paste, etc.

---

## 8. Fonts

### Nerd Fonts (recommended for terminal / dev work)

Nerd Fonts are patched developer fonts that bundle thousands of icons (Font Awesome, Devicons, Powerline symbols, etc.) into a single font file. They're required for tools like Starship prompt, Neovim plugins, and terminal status lines.

#### Option A: Install via Fedora COPR (easiest)

```bash
sudo dnf copr enable che/nerd-fonts
sudo dnf install nerd-fonts-jetbrains-mono   # or your preferred font
```

Other available fonts in this COPR: `nerd-fonts-hack`, `nerd-fonts-fira-code`, `nerd-fonts-cascadia-code`, and more.

#### Option B: Manual download

```bash
# Example: JetBrains Mono Nerd Font
wget -P ~/.local/share/fonts \
  https://github.com/ryanoasis/nerd-fonts/releases/latest/download/JetBrainsMono.zip

cd ~/.local/share/fonts
unzip JetBrainsMono.zip
rm JetBrainsMono.zip

# Rebuild font cache
fc-cache -fv
```

Browse all available fonts at [nerdfonts.com](https://www.nerdfonts.com). Replace `JetBrainsMono.zip` with any font name from the releases page.

Verify the font installed correctly:

```bash
fc-list : family style | grep -i "JetBrains"
```

Then set the font in your terminal emulator's preferences (KDE Konsole: Settings → Edit Current Profile → Appearance → Font).

### Installing Any Other Font (.ttf / .otf)

```bash
mkdir -p ~/.local/share/fonts
cp YourFont.ttf ~/.local/share/fonts/
fc-cache -fv
```

---

## 9. Widgets

### KDE Plasma Widgets

Right-click the desktop → **Add or Manage Widgets** to browse and install widgets from the KDE Store directly.

Useful widgets for Asahi/MacBook users:

- **System Load Viewer** — CPU, RAM, and swap at a glance (good for monitoring zswap behaviour)
- **Net Speed Widget** — real-time network usage in the taskbar
- **App Menu** — replaces the default launcher with a full application menu
- **Latte Dock** — a macOS-style dock for KDE Plasma

### Installing Latte Dock

```bash
sudo dnf install latte-dock
```

Launch it from the application menu, then right-click the dock to configure layout, size, and behaviour.

---

## 10. General Misc

### System Updates

Asahi Linux receives frequent kernel and firmware patches for Apple Silicon — keep up to date:

```bash
sudo dnf update -y
```

### RPM Fusion (extra codecs and software)

Many packages (media codecs, Steam, etc.) require RPM Fusion repositories:

```bash
sudo dnf install \
  https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

After enabling RPM Fusion, install full FFmpeg for better codec support in MPV and other apps:

```bash
sudo dnf install ffmpeg --allowerasing
```

### Useful CLI Tools

```bash
sudo dnf install \
  fastfetch \   # system info display (neofetch replacement)
  htop \        # interactive process viewer
  bat \         # cat with syntax highlighting
  eza \         # modern ls replacement with icons
  fd-find \     # faster find
  ripgrep       # faster grep
```

---

## 11. Firefox Configuration

### Hardware Acceleration

On Fedora Asahi with Wayland, hardware acceleration should work out of the box. To verify:

1. Open Firefox and go to `about:support`
2. Look for the **Compositing** row — it should say **WebRender** (not "WebRender (software)")
3. Look for **HARDWARE_VIDEO_DECODING** — it should say **available by default**

If video decoding is not hardware-accelerated:

1. Go to `about:config` and accept the warning
2. Search for `media.ffmpeg.vaapi.enabled` and set it to `true`
3. Restart Firefox

Check VA-API is working at the system level:

```bash
vainfo
```

This should list your GPU's supported profiles. If it errors, see [Section 14: Hardware Acceleration](#14-hardware-acceleration).

### Useful about:config Tweaks

| Preference | Value | Effect |
|---|---|---|
| `media.ffmpeg.vaapi.enabled` | `true` | Enable VA-API video decoding |
| `gfx.webrender.all` | `true` | Force WebRender on all hardware |
| `media.hardware-video-decoding.enabled` | `true` | Master toggle for hardware video decoding |

> 💡 Install the [enhanced-h264ify](https://addons.mozilla.org/en-US/firefox/addon/enhanced-h264ify/) extension to force YouTube to serve H.264, which is more widely hardware-accelerated than VP9/AV1 on Apple Silicon.

### Troubleshooting

- If you see video artifacts in Google Meet / Zoom: go to `about:config`, set `media.navigator.mediadatadecoder_vpx_enabled` to `false`, restart.
- To reset all custom changes: go to `about:support` → click **Refresh Firefox**.
- Run Firefox with VA-API debug logging: `MOZ_LOG="FFmpegVideo:5" firefox`

> ⚠️ **Known FEX issue:** Errors about `FEXInterpreter`, Node.js, or `muvm` when running installers or scripts are an x86 emulation issue — they do not affect normal Firefox use and can be safely ignored.

---

## 12. Terminal

> ⚠️ **Important:** Do **NOT** set Fish as your system-wide default login shell. This can break scripts, system services, and installers that assume a POSIX shell (bash/sh). Instead, configure your terminal emulator to launch Fish as its startup command — you get Fish interactively without breaking anything.

### Installing Fish Shell

```bash
sudo dnf install fish
```

In **KDE Konsole**: Settings → Edit Current Profile → General → Command → set to `/usr/bin/fish`.

### Disable the Fish Greeting

```bash
set -U fish_greeting ""
```

### Installing Starship Prompt

Starship is a fast, minimal, customisable prompt that works great with Fish and shows git status, directory, language versions, and more:

```bash
sudo dnf install starship
echo "starship init fish | source" >> ~/.config/fish/config.fish
```

Restart your terminal to see it.

### Customise Starship with a Preset Theme

```bash
# Catppuccin Powerline (pastel, powerline-style segments)
starship preset catppuccin-powerline -o ~/.config/starship.toml

# Or: Nerd Font symbols (clean general-purpose preset)
starship preset nerd-font-symbols -o ~/.config/starship.toml
```

> 💡 Starship requires a Nerd Font to display icons — see [Section 8: Fonts](#8-fonts).

### Useful Fish Abbreviations

Fish uses `abbr` instead of `alias` for interactive shortcuts. Add to `~/.config/fish/config.fish`:

```fish
abbr -a update "sudo dnf update"
abbr -a upgrade "sudo dnf upgrade"
abbr -a ls "eza --icons"
abbr -a ll "eza -la --icons"
```

### KDE Plasma Display Warning

If you see `interface 'kde_output_device_mode_v2' has no event 4` in terminal output, this is a harmless KDE Wayland protocol mismatch warning. It does not affect functionality — safely ignore it.

---

## 13. Games

> **Note:** Depending on your MacBook model and RAM, you may be limited in what you can run. Even with an 8 GB M1 or M2 you can run many indie games (e.g. Hollow Knight, Dead Cells) and even more intensive titles like Dark Souls II and Inertial Drift.

There are three methods — try Method 1 first, use Method 3 for stubborn Windows titles.

### Method 1: Steam

The simplest approach.

```bash
sudo dnf install steam
```

After launching Steam: Steam → Settings → Compatibility → tick **Enable Steam Play for all other titles** → select **Proton Experimental**.

> ⚠️ Steam on Asahi can be less stable for some titles. If a game won't launch, try Method 3.

### Method 2: Native GOG / Launcher Games

For native Linux games (from GOG or other stores), create a `.desktop` file so they appear in your app launcher. Example for Dead Cells:

```ini
[Desktop Entry]
Encoding=UTF-8
Value=1.0
Type=Application
Name=Dead Cells
GenericName=Dead Cells
Comment=Dead Cells
Icon=/home/YOUR_USER/GOGGames/Dead Cells/support/icon.png
Exec="/home/YOUR_USER/GOGGames/Dead Cells/start.sh" ""
Categories=Game;
Path=/home/YOUR_USER/GOGGames/Dead Cells
```

Save as `~/.local/share/applications/deadcells.desktop`. Replace `YOUR_USER` with your actual username.

### Method 3: muvm + GameMode (Windows games via Proton — most stable)

This method runs Windows games using muvm (a micro VM), umu-run (a Proton launcher), and DXVK for DirectX translation. More complex but more stable for demanding titles on Apple Silicon.

```bash
sudo dnf install gamemode
```

Create a `.desktop` file for your game. Example for Dark Souls II (GOG):

```ini
[Desktop Entry]
Categories=Game;
Comment=Optimized via muvm & GameMode
Exec=muvm \
  -e GAMEMODERUN=1 \
  -e DXVK_ASYNC=1 \
  -e STEAM_COMPAT_DATA_PATH='/home/YOUR_USER/umu-prefixes/ds2_gog' \
  -e WINEPREFIX='/home/YOUR_USER/umu-prefixes/ds2_gog/pfx' \
  -e "PROTONPATH=/home/YOUR_USER/.local/share/Steam/steamapps/common/Proton - Experimental" \
  -e XALIA_DISABLE=1 \
  umu-run "/home/YOUR_USER/GOGGames/Dark Souls II Scholar of the First Sin/Game/DarkSoulsII.exe"
Icon=/home/YOUR_USER/GOGGames/SWORD.webp
Name=Dark Souls II: Scholar of the First Sin
StartupNotify=true
Terminal=false
Type=Application
```

Save as `~/.local/share/applications/ds2.desktop`. Replace `YOUR_USER` throughout.

> 💡 Create a `~/umu-prefixes/` folder with a subfolder per game (e.g. `ds2_gog`). Each game gets its own Wine prefix to avoid conflicts between titles.

---

## 14. Hardware Acceleration

Asahi Linux includes the **Apple M2 Honeykrisp** Vulkan driver, meaning your GPU is ready — you just need to point your apps at it.

### Verify Vulkan is Working

```bash
vulkaninfo | grep "GPU id"
```

You should see `Apple M1` or `Apple M2` (Honeykrisp driver). If not, run `sudo dnf update -y` — the Honeykrisp driver ships with recent Asahi kernels.

### Enable for Desktop and Browsers

**Step 1:** Clear any conflicting overrides. Open `~/.bashrc` and remove any `export` lines mentioning `MESA`, `DRI`, or `GALLIUM`. Restart your terminal.

**Step 2:** Add only these two lines to `~/.bashrc`:

```bash
export ANV_VIDEO_DISABLE_BITSTREAM=1
export MESA_VK_DEVICE_SELECT=10005:0000
```

Then reload:

```bash
source ~/.bashrc
```

> ⚠️ `ANV_VIDEO_DISABLE_BITSTREAM=1` disables bitstream acceleration, which is currently broken on Apple Silicon and would otherwise cause crashes. `MESA_VK_DEVICE_SELECT` ensures the correct GPU device is always selected.

### Verify VA-API (for video playback)

```bash
vainfo
```

If this lists your GPU's supported decode/encode profiles, video hardware acceleration is working. If it errors:

```bash
sudo dnf install mesa-va-drivers mesa-vdpau-drivers
```

---

## 15. Other Languages

### Japanese Input (Romaji → Hiragana/Kanji) on KDE Plasma

This uses **Fcitx5** (input method framework) with the **Mozc** engine (Google's open-source Japanese IME). Confirmed working on Fedora Asahi with KDE Plasma (including Fedora 44 / KDE 6.6).

#### Step 1: Install the packages

```bash
sudo dnf install fcitx5-mozc kcm-fcitx5
```

> ⚠️ **Fedora 42+ note:** Avoid installing `fcitx5-autostart` — it can cause KDE Plasma shell crashes on some machines. The two packages above are sufficient; Fcitx5 will still auto-start via KDE's virtual keyboard system.

#### Step 2: Log out and back in

Required for KDE to detect the new packages.

#### Step 3: Set Fcitx5 as the Virtual Keyboard

Open **System Settings** → **Keyboard** → **Virtual Keyboard** → select **Fcitx 5** → click **Apply**.

#### Step 4: Add Mozc as an input method

Right-click the keyboard icon in the system tray → **Configure**. In the Available Input Methods box, search for **Mozc**, then click `<` to add it to Current Input Methods. Keep **Keyboard – English (US)** in the list so you can switch back to English. Click **OK**.

#### Step 5: Switch between English and Japanese

Press `Ctrl+Space` in any application to toggle between English and Mozc. As you type in Romaji, Mozc converts it to Hiragana and offers Kanji options in a pop-up.

> 💡 If you receive a warning on login about `GTK_IM_MODULE` and `QT_IM_MODULE` being set, this is safe to ignore — Wayland input method is working correctly regardless.

---

---

## 16. Display & HiDPI Scaling

MacBook screens are high-DPI (Retina), so UI elements may look tiny at native resolution. KDE Plasma handles this well with fractional scaling.

### Enable Fractional Scaling

Open **System Settings** → **Display and Monitor** → **Displays**. Under **Scale**, set it to **200%** for a sharp 1:1 Retina-style experience, or **150%** for more screen real estate with slightly blurrier text.

> 💡 200% is pixel-perfect and sharp. 150% uses fractional scaling which can introduce slight blur on some apps — this is a Wayland limitation, not a KDE one.

### Night Light (Blue Light Filter)

Open **System Settings** → **Display and Monitor** → **Night Color**. Enable it and set your preferred colour temperature and schedule. This is especially useful if you use the machine late at night.

### Adjusting Font DPI Separately

If scaling feels off for just fonts: **System Settings** → **Fonts** → **Force Font DPI** → set to `192` (for 2× scaling) or adjust to taste.

### External Monitors

USB-C to DisplayPort and USB-C to HDMI adapters generally work on Asahi. Thunderbolt docks have variable support — check the [Asahi Linux feature support page](https://asahilinux.org/fedora/) for your specific model before buying one.

If plugging in an external display causes KDE to crash or freeze, try:

```bash
kwin_wayland --replace &
```

---

## 17. Battery Life & Power Management

Asahi Linux has good battery life on Apple Silicon, but there are a few tweaks that help.

### Install power-profiles-daemon

Fedora Asahi ships with `power-profiles-daemon` which integrates with KDE's battery widget:

```bash
sudo dnf install power-profiles-daemon
sudo systemctl enable --now power-profiles-daemon
```

You can then switch between **Power Saver**, **Balanced**, and **Performance** modes directly from the battery icon in the KDE taskbar.

### Install TLP for finer control (optional)

> ⚠️ Do **not** run TLP alongside `power-profiles-daemon` — they conflict. Disable one before enabling the other.

```bash
sudo dnf install tlp tlp-rdw
sudo systemctl enable --now tlp
```

### Check Battery Status

```bash
upower -i /org/freedesktop/UPower/devices/battery_BAT0
```

This shows charge level, charge cycles, energy capacity, and whether the battery is discharging, charging, or fully charged.

### Reduce Background Wake-ups

If the machine feels warm on idle, check what's running:

```bash
powertop
```

PowerTOP shows per-process power usage and lets you toggle tunables. Run with `sudo powertop --auto-tune` to apply all recommended power saving settings in one go (these reset on reboot — use a systemd service or TLP to make them permanent).

---

## 18. Touchpad & Gestures

The MacBook trackpad works well on Asahi Linux via `libinput`. Most gestures are configurable through KDE.

### Basic Touchpad Settings

Open **System Settings** → **Input Devices** → **Touchpad**. From here you can configure:

- Tap to click
- Two-finger scrolling direction (natural / traditional)
- Three-finger tap behaviour
- Palm detection sensitivity

### Gestures with Touchégg / Fusuma

For more advanced gestures (swipe to switch workspaces, pinch to zoom, etc.), install Touchégg:

```bash
sudo dnf install touchegg
sudo systemctl enable --now touchegg
```

Then install the **KDE Touchégg** plugin from the KDE Store (right-click desktop → Add Widgets → Get New Widgets) to configure gesture actions from within System Settings.

### Three-Finger Drag

Three-finger drag (to move windows, like macOS) is not enabled by default. To enable it via libinput:

```bash
sudo nano /etc/X11/xorg.conf.d/99-touchpad.conf
```

Add:

```
Section "InputClass"
    Identifier "libinput touchpad"
    MatchIsTouchpad "on"
    Driver "libinput"
    Option "Tapping" "on"
    Option "TappingDragLock" "on"
    Option "NaturalScrolling" "true"
EndSection
```

---

## 19. KDE Theming

KDE Plasma is highly customisable. Here's how to get a clean, cohesive look.

### Catppuccin (recommended)

Catppuccin is a popular pastel colour scheme with official KDE support. Install via the KDE Store:

1. **System Settings** → **Appearance** → **Global Theme** → **Get New Global Themes**
2. Search for **Catppuccin** and install your preferred flavour (Mocha, Macchiato, Frappé, or Latte)
3. Apply it, then optionally apply the matching **Icon Theme** and **Colour Scheme** from their respective settings pages

For the full Catppuccin experience (including Firefox), visit [catppuccin.com](https://catppuccin.com) — they have themes for virtually every app.

### Lightly (modern window decoration)

Lightly is a clean, modern KDE window decoration and application style inspired by macOS and GNOME:

```bash
sudo dnf copr enable soloturn/lightly
sudo dnf install lightly
```

Apply via **System Settings** → **Appearance** → **Application Style** → select **Lightly**, and **Window Decorations** → select **Lightly**.

### Cursors

For a more macOS-like cursor: **System Settings** → **Appearance** → **Cursors** → **Get New Cursors** → search for **macOS** or **Bibata**.

### Panel / Taskbar

Right-click the taskbar → **Enter Edit Mode** to reposition, resize, or remove the panel. A common MacBook-inspired layout:
- Move the panel to the **top**
- Add a **Global Menu** widget (shows the app menu bar at the top, like macOS)
- Add **Latte Dock** at the bottom for a dock (see Section 9)

---

## 20. Flatpak

Flatpak lets you install sandboxed apps that are independent of your system's package manager — useful for getting up-to-date versions of apps that are behind in the Fedora repos.

### Enable Flathub

```bash
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

### Installing Apps

```bash
# Examples
flatpak install flathub com.spotify.Client
flatpak install flathub md.obsidian.Obsidian
flatpak install flathub com.discordapp.Discord
flatpak install flathub org.gimp.GIMP
```

Search for apps at [flathub.org](https://flathub.org).

### Running Flatpak Apps

They appear in your KDE application launcher automatically. Or from the terminal:

```bash
flatpak run com.spotify.Client
```

### Updating All Flatpaks

```bash
flatpak update
```

> 💡 Flatpak apps are sandboxed and may need permission grants to access your files. If an app can't see your home folder, go to **System Settings** → **Applications** → **Flatpak Permissions** to adjust.

---

## 21. Development Environment

### VS Code

Install via Flatpak for the most up-to-date version:

```bash
flatpak install flathub com.visualstudio.code
```

Or via the official RPM repo:

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'
sudo dnf install code
```

### Git

Git is usually pre-installed. Configure it:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor nano   # or vim, code, etc.
```

### Node.js (via nvm — recommended over system Node)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
# Restart terminal, then:
nvm install --lts
nvm use --lts
```

### Python

Python 3 is pre-installed on Fedora. For isolated project environments:

```bash
python3 -m venv myenv
source myenv/bin/activate
pip install your-package
```

### Neovim

```bash
sudo dnf install neovim
```

For a full modern Neovim config, [LazyVim](https://www.lazyvim.org/) is a popular pre-configured distribution — follow the install guide on their site. Requires a Nerd Font (see Section 8).

---

## 22. Containers (Podman / Docker)

Fedora ships with **Podman** — a rootless, daemonless Docker-compatible container runtime. You can use Docker commands with Podman transparently.

### Install Podman

```bash
sudo dnf install podman podman-compose
```

### Docker compatibility alias

```bash
echo "alias docker=podman" >> ~/.bashrc
source ~/.bashrc
```

Now `docker run`, `docker build`, etc. all work via Podman.

### Podman Desktop (GUI)

```bash
flatpak install flathub io.podman_desktop.PodmanDesktop
```

### Basic Usage

```bash
# Pull and run an image
podman run -it ubuntu bash

# List running containers
podman ps

# List all containers (including stopped)
podman ps -a

# Build from Dockerfile
podman build -t myapp .
```

> 💡 Podman containers run rootless by default (no daemon, no `sudo` needed), which is safer than Docker's traditional model.

---

## 23. SSH Setup

### Generate an SSH Key

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

Accept the default path (`~/.ssh/id_ed25519`) and set a passphrase. Ed25519 is more secure and faster than RSA.

### Add to SSH Agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

To have the key added automatically on login, add those lines to `~/.bashrc` (or `~/.config/fish/config.fish` for Fish).

### Copy Public Key to a Server

```bash
ssh-copy-id user@your-server.com
```

Or manually: copy the output of `cat ~/.ssh/id_ed25519.pub` into `~/.ssh/authorized_keys` on the remote server.

### Add to GitHub / GitLab

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the output, then paste it into GitHub: Settings → SSH and GPG Keys → New SSH key.

### SSH Config File (for shortcuts)

Create or edit `~/.ssh/config`:

```
Host myserver
    HostName 192.168.1.100
    User youruser
    IdentityFile ~/.ssh/id_ed25519
```

Now `ssh myserver` works instead of typing the full command each time.

---

## 24. Backups with Snapper

Fedora Asahi uses **Btrfs** as its filesystem, which supports copy-on-write snapshots. **Snapper** manages these snapshots automatically, giving you a Time Machine-like rollback capability.

### Install Snapper

```bash
sudo dnf install snapper
```

### Create a Snapper Config for Root

```bash
sudo snapper -c root create-config /
```

### Take a Manual Snapshot

```bash
sudo snapper -c root create --description "before major changes"
```

### List Snapshots

```bash
sudo snapper -c root list
```

### Restore a Snapshot

To roll back to a previous snapshot, boot into a snapshot from the GRUB menu (if `grub-btrfs` is installed), or restore manually:

```bash
sudo snapper -c root undochange 1..2   # restore changes between snapshots 1 and 2
```

### Enable Automatic Snapshots

Snapper can automatically snapshot before and after every `dnf` transaction (update, install, remove), giving you a clean rollback path if an update breaks something:

```bash
sudo dnf install python3-dnf-plugin-snapper
```

Once installed, every `dnf` operation will create a pre/post snapshot pair automatically.

---

## 25. Screen Recording & Screenshots

### Screenshots

KDE Plasma has a built-in screenshot tool: press `Print Screen` (or `Fn+Shift` on some MacBook keyboards) to open **Spectacle**.

From the terminal:

```bash
spectacle            # full GUI
spectacle -f         # full screen, no GUI
spectacle -r         # select region
spectacle -a         # active window
```

Screenshots are saved to `~/Pictures/Screenshots` by default.

### Screen Recording

**Obs Studio** is the best option for screen recording and streaming:

```bash
sudo dnf install obs-studio
```

Or via Flatpak for the latest version:

```bash
flatpak install flathub com.obsproject.Studio
```

OBS works with Wayland via PipeWire screen capture. On first launch, select **Wayland** as the capture method when adding a Screen Capture source.

> 💡 For quick one-off recordings without OBS, KDE Plasma 6 has a built-in screen recorder: `Meta+Shift+R` starts and stops a recording, saved to `~/Videos`.

---

## 26. Printing & Scanning

### Printing

CUPS (Common Unix Printing System) handles printing on Linux. Install it and enable it:

```bash
sudo dnf install cups cups-filters system-config-printer
sudo systemctl enable --now cups
```

Open the CUPS web interface to add a printer: go to `http://localhost:631` in Firefox → **Administration** → **Add Printer**.

Most modern printers are supported automatically via **IPP Everywhere** (driverless printing) — just plug in or connect to the same network and CUPS should detect it.

For HP printers specifically:

```bash
sudo dnf install hplip
hp-setup
```

### Scanning

```bash
sudo dnf install simple-scan
```

**Simple Scan** is a straightforward GUI scanner app. Most USB and network scanners are detected automatically via SANE.

---

## 27. VPN

### WireGuard (recommended — fastest and simplest)

WireGuard is built into the Linux kernel and works excellently on Asahi:

```bash
sudo dnf install wireguard-tools
```

If your VPN provider gives you a `.conf` file:

```bash
sudo cp myvpn.conf /etc/wireguard/wg0.conf
sudo systemctl enable --now wg-quick@wg0
```

To connect/disconnect via KDE: install the **plasma-nm** WireGuard plugin:

```bash
sudo dnf install NetworkManager-wireguard-gnome
```

Then add your VPN in **System Settings** → **Connections** → **+** → **WireGuard**.

### OpenVPN

```bash
sudo dnf install NetworkManager-openvpn-gnome
```

Import your `.ovpn` file via **System Settings** → **Connections** → **+** → **Import VPN connection**.

### ProtonVPN (if you use it)

```bash
flatpak install flathub com.protonvpn.www
```

The official ProtonVPN Linux app works on Asahi via Flatpak and integrates with NetworkManager.

---

## 28. Wayland Tips & Quirks

Asahi Linux runs on Wayland by default (not X11). This is generally better, but a few things behave differently.

### XWayland (for legacy X11 apps)

Most apps run natively on Wayland. For older apps that need X11, **XWayland** is included and runs transparently — you generally don't need to do anything.

To check whether an app is running on Wayland or XWayland:

```bash
flatpak run --env=WAYLAND_DEBUG=1 com.example.App 2>&1 | head -20
# or simply:
qdbus org.kde.KWin /KWin supportInformation | grep -i xwayland
```

### Screen Sharing in Browsers / Video Calls

Screen sharing works on Wayland via PipeWire. If a browser or video call app says it can't share your screen:

1. Make sure `xdg-desktop-portal-kde` is installed:
   ```bash
   sudo dnf install xdg-desktop-portal-kde
   ```
2. Restart your session (log out and back in)
3. In Firefox, go to `about:config` and set `media.webrtc.camera.allow-pipewire` to `true`

### Clipboard Managers

Wayland clipboard behaviour is slightly different from X11 — content copied in one app is lost when that app closes. To preserve clipboard history:

```bash
sudo dnf install klipper
```

Klipper is KDE's clipboard manager and comes with Plasma. Check it's running in the system tray.

### App Scaling Issues (XWayland blurriness)

If an X11 app (running via XWayland) looks blurry at your HiDPI scale, set this environment variable:

```bash
export GDK_SCALE=2          # for GTK apps
export QT_SCALE_FACTOR=2    # for Qt apps
```

Add to `~/.bashrc` to make permanent.

---

## 29. Switching Back to macOS (Dual Boot)

If Asahi Linux is installed alongside macOS, you can switch between them at boot.

### At Startup

Hold down the **power button** (on Apple Silicon Macs, not a separate key) until you see the startup options screen. From here, select either your macOS volume or Fedora Asahi.

### Set the Default Boot OS

To make macOS the default (so you don't have to hold the power button every time):

```bash
# From inside Asahi Linux:
sudo systemctl reboot --boot-loader-entry=auto   # boots into firmware picker once
```

Or from **macOS**: System Settings → General → Startup Disk → select your macOS volume.

To make Asahi the default from macOS:

```bash
# In macOS terminal:
sudo bless --mount /Volumes/Fedora --setBoot --nextonly
```

> 💡 The cleaner long-term way is to set your preferred default from the macOS Startup Disk pane — this persists across reboots.

---

## 30. Removing Asahi Linux

If you want to go back to macOS only and reclaim your disk space, the Asahi installer handles uninstallation safely. **Do not try to delete the partition manually in Disk Utility** — this can leave the bootloader in a broken state.

### Back Up First

Before removing, make sure you've copied out everything you want to keep (see the backup checklist in the conversation that prompted this guide):

- `~/.config/` (dotfiles, app configs)
- `~/.ssh/` (SSH keys)
- `~/GOGGames/`, `~/downloads/` (games and media)
- Any `.desktop` files from `~/.local/share/applications/`

### Run the Asahi Uninstaller

Boot into **macOS**, open Terminal, and run the same installer script:

```bash
curl https://alx.sh | sh
```

Choose the **uninstall** option. The installer will remove the Asahi partitions, remove the Asahi bootloader entry, and restore macOS as the sole boot target. Your macOS install is untouched throughout.

After uninstalling, run **Disk Utility** → select your main drive → **First Aid** to verify the partition map is clean.

> ⚠️ Make sure macOS is up to date before running the uninstaller, and ideally have a Time Machine or external backup of macOS itself just in case.

---


## Contributing / Feedback

Found something broken or have an improvement? Feel free to open an issue or PR.

---

*Guide maintained by: [your name / handle]*
*Last updated: 2025*
