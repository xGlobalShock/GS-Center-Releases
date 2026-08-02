# GS Center - Privacy

This isn't a wall of legal boilerplate. It's a straight answer to the
question most people actually have: **"What does this app do with my
data?"** GS Center is a Windows optimization and gaming tool, and by its
nature it touches a lot of low-level system information. So it's fair to
want to know where that information goes.

Short version: **almost everything stays on your PC.** The app was built
local-first. The only things that ever leave your machine are the ones
that genuinely need a server to work - signing in, your Pro status. Everything else -
hardware readings, tweaks, cleanups, the files it scans - happens on
your computer and never gets uploaded anywhere.

Here's the full picture.

---

## The one-paragraph summary

GS Center reads a lot about your system because that's the whole point of
the app. It monitors your CPU, GPU, RAM, disks, and network in real time;
it applies registry tweaks; it cleans caches; it manages installed apps.
All of that runs locally. We don't sell data, we don't run advertising
SDKs, and there's no hidden background uploader. The handful of features
that talk to the internet are clearly tied to a thing you asked for -
logging in, checking for updates, downloading an app you picked,
verifying your subscription.

---

## What runs entirely on your device

These features never send your data off your PC. They read and act on
your system locally:

- **Live hardware monitoring** - CPU, GPU, RAM, disk, and network metrics
  come from a small local helper program (GCMonitor.exe, built on
  LibreHardwareMonitor) that runs on your machine and streams readings to
  the app. Those numbers are displayed and then discarded as new readings
  arrive. They are not logged to a server.
- **Performance tweaks** - registry edits applied locally through
  Windows. The app only touches specific, documented keys.
- **Cleanup toolkit** - temp files, shader caches, DNS cache, recycle bin,
  etc. The app deletes files on your disk. It does not read their contents
  or copy them anywhere.
- **App uninstaller, startup manager, space analyzer, debloat** - these
  enumerate what's installed on your PC. That inventory stays on your PC.
- **Resolution manager, mouse polling test, OBS presets, ISO builder** -
  all local device and file operations.
- **Overlay HUD and report card** - built from the same local metrics.

---

## What leaves your device, and why

### 1. Signing in

If you want to try our FREE / Pro features, you have to sign in with 
**Discord or Twitch**. We use their standard OAuth login, handled through
Supabase (our authentication and database provider).

- We receive the basic profile those services hand over (such as your
  account ID, display name, and avatar) so we can recognize you and show
  your status.
- Login happens in an isolated browser window. Your Discord/Twitch
  password is never seen, handled, or stored by GS Center.
- We don't store long-lived login secrets on your device. Your session is
  kept only as long as needed to keep you signed in.

### 2. Your Pro / subscription status

Your account role (Free, Pro, etc.) and any subscription expiry are stored
in our Supabase database so the app knows which features to unlock. If you
buy Pro, payment is processed through **PayPal** - GS Center never sees or
stores your card details. We only learn that a payment succeeded.

While you're signed in, the app sends a lightweight "still active" beacon
so we can show accurate session info. It doesn't carry your personal files
or system data.

### 3. Internet features you trigger yourself

Some actions reach out to the internet because that's what they're for:

- **App installer** uses Windows' own `winget` to download apps you pick,
  and fetches app icons from a public icon CDN.
- **Software updates** uses `winget upgrade` to update your installed
  programs.
- **Network diagnostics** pings regional gaming servers and runs speed
  tests (Fast.com, Ookla, testmy.net) inside isolated views, so you can
  measure your connection.
- **App auto-updates** check our public releases repository on GitHub for
  new versions of GS Center.

These are normal outbound requests tied to a feature you used - not
background tracking.

---

## Where data lives

| Data | Where it's stored | Leaves your PC? |
|---|---|---|
| Hardware metrics & telemetry | In memory, shown live | No |
| Tweaks, cleanups, file ops | Your local system | No |
| App settings & preferences | Local file + browser storage on your PC | No |
| Account profile & Pro status | Supabase (our database) | Yes - to run your account |
| Payment | PayPal | We never see card details |

---

## Permissions, and why the app needs them

- **Administrator rights.** GS Center asks to run as administrator because
  applying registry tweaks, cleaning system caches, repairing Windows, and
  building ISOs genuinely require it. The elevated access is used for the
  optimization work you ask for - not to snoop.
- **Protected guardrails.** The app refuses to disable critical security
  items (like Windows Defender) and excludes protected system folders from
  cleanup and scanning, so it can't be used to weaken your machine.

---

## Kids

GS Center is a Windows tuning tool aimed at gamers and PC enthusiasts.
It isn't designed for or directed at children, and we don't knowingly
collect data from anyone under 13.

---

## Changes to this document

As the app evolves, this document will be updated to match what it
actually does. The date at the top reflects the latest revision. The
guiding principle won't change: keep things local, be clear about the few
things that aren't, and never collect what we don't need.
