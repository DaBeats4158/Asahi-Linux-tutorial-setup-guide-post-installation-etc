# Asahi Linux on MacBook — Setup & Configuration Guide

> **Disclaimer:** This is a personal side project. I hold no accountability for any loss of data or harm to devices. Use at your own risk — but I hope this helps you get the most out of Asahi Linux on your MacBook! :)

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

## Contributing / Feedback

Found something broken or have an improvement? Feel free to open an issue or PR.

---

*Guide maintained by: [your name / handle]*
*Last updated: 2025*
