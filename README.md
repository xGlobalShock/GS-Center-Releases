# GS Center (Beta) – Currently in Development

<p align="center">
  <strong>The all-in-one performance control center for Windows gamers and streamers.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Status-BETA-orange?style=for-the-badge" />
<a href="https://github.com/xGlobalShock/GS-Center-Releases/releases/download/v2.9.5/GS-Center-Setup-2.9.5.exe"> 
  <img src="https://img.shields.io/badge/Download-Latest%20Release-brightgreen?style=for-the-badge">
</a>
  <img src="https://img.shields.io/badge/Platform-Windows%2010%2F11%20x64-0078D6?style=for-the-badge&logo=windows" />
  <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" />
</p>

<img width="1473" height="855" alt="GSCenter" src="https://github.com/user-attachments/assets/b780f4c7-5c14-4b83-bb3d-524b9d6f7bcc" />

## Overview

GS Center is a powerful desktop application built for gamers, streamers, and performance-focused users who want full control over their system in one place.

It combines real-time monitoring, registry-level optimizations, cleanup tools, system management, game configuration, streaming setup, and an AI support agent into a single streamlined experience.

No unnecessary setup. No complexity. Just control.

---

### Upgrade to PRO
* Current price: **$4.99/month**
* Upcoming pricing tiers: **$14.99/month** and **$29.99/month**
* Includes access to both Free and Pro feature sets
* Pricing is subject to change in the future

---

## Features

### Dashboard & Live Metrics

- **CPU** - usage, per-core load, clock, power, voltage, temperature
- **GPU** - load, VRAM, core/memory clock, fan RPM, power, temperature, hotspot temp
- **RAM** - used/total, available, cached, standby, top processes by RAM
- **Disk** - read/write speed, temperature, lifespan, free/total per drive
- **Network** - up/down speed, latency, packet loss, SSID, signal, adapter info
- **System** - process count, uptime, Windows build, power plan
- **Health Score** - 0–100 composite from CPU temp, usage, RAM pressure, disk space, GPU temp, latency, disk health
- **Advisor Panel** - system insights (critical / warning / good) and hardware upgrade recommendations
- **Anti-Cheat Compatibility Checker** - flags processes and tweaks that could trigger Vanguard, EAC, BattlEye, FACEIT, or Ricochet

---

### PC Tweaks (17 Registry Optimizations)

| Category | Tweaks |
|----------|--------|
| **VBS** | Disable VBS / Core Isolation |
| **CPU & Priority** | Reduce Input Latency (IRQ8 Priority), Boost Foreground App Priority, Games Priority |
| **GPU** | Enable GPU Low-Latency Mode (HW Scheduling), Disable GPU Timeout Detection (TDR) |
| **Memory** | Use Full RAM Capacity (Disable Memory Compression), Expand System File Cache |
| **Network** | Network Interrupts Priority, Disable Network Throttling |
| **Display** | Disable Fullscreen Optimization, DWM Overlay Test Mode, Fullscreen Optimization Mode |
| **Game DVR** | Disable Game DVR, Game DVR Policy, Disable App Capture |
| **Hardware** | Disable USB Selective Suspend |

- Individual toggle or batch apply
- System restore point creation before applying (Pro)
- One-click reset per tweak
- Per-tweak anti-cheat verdict

---

### Cleanup Toolkit (42+ Cleaners)

**Windows Cache** - Temp files, Windows Update cache, DNS cache, RAM standby, Recycle Bin, Thumbnail cache, Logs, Crash dumps, Error reports, Delivery Optimization

**Game Shader Caches** - Apex Legends, CS2, Fortnite, Forza Horizon 5, League of Legends, Overwatch 2, Rainbow Six Siege, Rocket League, Valorant, Dead by Daylight, Marvel Rivals, Arc Raiders, PUBG, Rust, R.E.P.O.

**GPU Driver Cache** - NVIDIA (DXCache/GLCache), AMD

**Win Tweaks** - Disk Cleanup, Services Optimization, Disable Telemetry, Disable Location, Remove Widgets, Disable Consumer Features, Disable Hibernation, PowerShell 7 Telemetry, Disable Activity History, Disable WPBT, Disable Explorer Auto Discovery, End Task on Taskbar, Disable Store Search, Delete Temp Files

**Win Preferences** - Dark Theme, Center Taskbar, Search on Taskbar, Sticky Keys, Numlock on Boot, Show File Extensions, Show Hidden Files, Mouse Acceleration, GS Power Plan, Multiplane Overlay, Modern Standby, Detailed BSOD, Verbose Logon, Start Menu Recommendations, Bing Search, Cross Device, New Outlook, Task View Button, Settings Home Page, Revert Start Menu

**System Repair** - SFC, DISM, ChkDsk with live progress and a floating always-on-top overlay window

---

### Game Library

Manage and optimize supported games with built-in FPS prediction.

**18 supported titles:**
Apex Legends, Valorant, CS2, Fortnite, Overwatch 2, League of Legends, Rocket League, Dead by Daylight, Marvel Rivals, Arc Raiders, Rust, Minecraft, R.E.P.O., COD Warzone, COD Black Ops 6, COD Black Ops 7, Rainbow Six Siege, PUBG

- Per-game profile editor (full read/write for Apex Legends, framework in place for more)
- Per-game config file read/write with optional read-only lock
- **FPS Predictor** at 1080p / 1440p / 4K with verdicts (Runs Excellent, Runs Smooth, Might Lag, Can't Run)
- Hardware vs game requirements comparison
- Resolution builder (16:9, 16:10, 4:3, custom)

---

### Stream (OBS Presets + Dual-PC Wizard)

- One-click OBS preset deployment: Gaming, Starting Soon, BRB, Ending
- Twitch-tuned encoder + audio settings
- Auto-detects OBS install and launches after deploying
- **Dual-PC Wizard** — guided setup for streaming + gaming dual-PC: NDI vs capture card, audio routing, OBS scenes, NDI presets per link speed, troubleshooter

---

### Network Diagnostics

- Ping tests to 10 regional gaming servers (NA East/West, EU West/Central, Asia, Southeast Asia, South America, Middle East, Oceania, Africa)
- Speed tests via Fast.com, Ookla Speedtest, testmy.net (sandboxed webview)
- Color-coded latency tiers (green ≤50 ms, yellow ≤100 ms, red >100 ms)
- Traceroute

---

### Devices (Resolution & Mouse)

- Active display list with adapter, current resolution and refresh rate
- Enumerate every supported W × H × Hz mode per display
- Apply any supported mode via native C# helper (`ResolutionHelper.exe`)
- **Mouse Polling Rate Test** — live polling-rate sampler with smoothed readout

---

### Apps Manager

**App Installer** - 52 curated applications via winget across 8 categories: Browsers, Communications, Gaming, Gaming Tools, Streaming & Audio, Development, Utilities, Media. One-click install with progress tracking and bulk install.

**App Uninstaller** - Detects apps from registry, WMI, and AppX. Performs leftover cleanup (orphan files, folders, registry keys, services, scheduled tasks). Three scan modes: Safe, Moderate, Advanced.

**Windows Debloat (Pro)** - 100+ pre-installed Windows apps removable from AppX, Windows Capabilities, and Windows Features. Batch removal, reinstall support, protected system apps excluded.

**Startup Manager** - Lists startup items from HKCU/HKLM Run keys plus User and Common Startup folders. Toggle enable/disable, sort/search/filter, protected entries, running-process status sync.

**Disk Analyzer (Pro)** - Recursive directory scanning with drive selector, context menu, LRU caching, protected paths excluded.

**Windows 11 ISO Builder (Pro)**

Import a stock Windows 11 ISO and apply up to 14 modifications:

1. Remove AppX bloatware (Clipchamp, BingNews, Teams, etc.)
2. Bypass TPM / SecureBoot / RAM / Storage checks
3. Disable sponsored apps & content delivery
4. Enable local accounts on OOBE (BypassNRO)
5. Disable telemetry & advertising ID
6. Disable BitLocker device encryption
7. Disable Chat icon, OneDrive backup, Copilot
8. Prevent auto-install of DevHome, Outlook, Teams
9. Suppress Windows Update during OOBE
10. Disable reserved storage
11. Delete CEIP / Windows Update scheduled tasks
12. Write `autounattend.xml` + setup scripts
13. DISM component cleanup (`/ResetBase`)
14. Remove ISO `support\` folder

Output: a new ISO or a bootable USB. Optional driver injection.

---

### Software Updates (Pro)

Scan and update installed software.

- Detect outdated apps, batch update, per-package progress with phases (preparing → downloading → verifying → installing → done/error)
- Cancellation, update history, floating progress dock visible from any page

---

### PC Report Card

Exportable one-page system snapshot including specifications, performance status, applied optimizations, and overall health overview.

---

### Settings

- **Startup** - Auto-cleanup on launch, minimize-to-tray
- **Peformance** - GPU Acceleration, Render Quality
- **Appearance** - Theme, Light Effect, Background
- **About** - Version, Channel Update, What's new, Privacy

---

## System Requirements

| Requirement | Details |
|-------------|---------|
| OS | Windows 10 or Windows 11 (64-bit) |
| Runtime | .NET 8.0 (bundled with installer) |
| Permissions | Administrator access (for registry tweaks, service management, system repair, ISO builder) |
| Internet | Required for auth, updates, speed tests, AI agent |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | React 18, TypeScript, Framer Motion |
| Build | Create React App (react-scripts 5) |
| Desktop | Electron 33 |
| WebGL | OGL (light ray effects) |
| Hardware Monitor | Native C# .NET 8 sidecar |
| Auth & Database | Supabase (Discord / Twitch OAuth) |

---

## Support

Open an issue: https://github.com/xGlobalShock/GS-Center-Releases/issues

View Releases: https://github.com/xGlobalShock/GS-Center-Releases/releases

View Privacy:  https://github.com/xGlobalShock/GS-Center-Releases/blob/main/PRIVACY.md

View Roadmap:  Updating this soon...

---

## Inspired By

A number of excellent projects inspired ideas, workflows, and features found throughout GS Center. Recognition goes to the creators of the following applications:

- Christitus
- Winhance
- Windshark
- UniGetUI
- Revo Uninstaller
- Treesize
- Grafana
- Win11Debloat

---

<p align="center">Built by <a href="https://github.com/xGlobalShock">GlobalShock</a></p>
