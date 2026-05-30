# GS Center - Privacy

This isn't a wall of legal boilerplate. It's a straight answer to the
question most people actually have: **"What does this app do with my
data?"** GS Center is a Windows optimization and gaming tool, and by its
nature it touches a lot of low-level system information. So it's fair to
want to know where that information goes.

Short version: **almost everything stays on your PC.** The app was built
local-first. The only things that ever leave your machine are the ones
that genuinely need a server to work - signing in, your Pro status, and
(only if you choose to use it) the AI assistant. Everything else -
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
verifying your subscription, or chatting with the optional AI assistant.

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

If you never sign in and never open the AI assistant, GS Center
effectively operates as an offline tool, aside from the internet features
you explicitly trigger (like installing an app or checking for updates).

---

## What leaves your device, and why

### 1. Signing in (optional)

You can use a large part of GS Center without an account. If you want Pro
features, you sign in with **Discord or Twitch**. We use their standard
OAuth login, handled through Supabase (our authentication and database
provider).

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

### 3. The AI assistant (optional, off until you open it)

This is the one feature that, by design, sends system information to an
outside service - because that's how it diagnoses problems. It's worth
explaining carefully.

When you chat with the AI assistant, the app builds a snapshot of relevant
system context (things like CPU temperature, RAM usage, disk free space,
network latency, GPU load) and sends it, along with your message, to an AI
provider (such as Groq, OpenAI, Anthropic, Gemini, OpenRouter, or GitHub
Models, depending on configuration).

Before anything is sent, the app runs it through a **redactor** that
strips out personally identifying details, including:

- MAC addresses
- Public IP addresses
- Hardware serial numbers
- Your Windows hostname / PC name
- Your username inside file paths (e.g. `C:\Users\<you>\...` becomes
  `C:\Users\<user>\...`)

So the AI sees "a PC with these symptoms," not "this specific person's
named machine."

A few more things about the AI:

- **It can't run commands on its own.** When it wants to fix something, it
  proposes an action from a fixed, built-in library, and **you have to
  confirm** before anything runs. High-risk actions always require your
  approval.
- **Your API keys stay out of the browser layer.** Keys live in the main
  process, a key you supply yourself, or a server-side secret - never
  exposed to the app's front end.
- **Chat memory is off by default.** Out of the box, conversations aren't
  stored on a server. There's an optional cloud-memory mode you can turn
  on; when enabled, redacted messages and fix history are saved to your
  own row in our database, protected so only your account can read them.
- A local, append-only audit log of fixes the AI ran is kept on your PC
  (so you can see what happened). It isn't uploaded unless you opt in.

If you never open the AI assistant, none of this applies.

### 4. Internet features you trigger yourself

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
- **Optional web search** for the AI assistant (off unless enabled) sends
  your search query to a search provider.

These are normal outbound requests tied to a feature you used - not
background tracking.

---

## Analytics: what we count, and what we don't

GS Center keeps some anonymous, **local** usage counters for the AI
feature - things like "a message was sent" or "a fix completed." This
helps gauge reliability. Important details:

- It's tied to a **hashed device ID**, not your name or email. The hash is
  a one-way fingerprint derived locally.
- The actual content of your messages is **not** included. The code
  explicitly drops fields like username, email, IP, and message text
  before writing anything.
- These counts are written to a local log file on your own machine.

There are **no third-party advertising or tracking SDKs** bundled in the
app. We don't sell or share your data with data brokers. There's nothing
to sell - we don't collect it in the first place.

---

## Where data lives

| Data | Where it's stored | Leaves your PC? |
|---|---|---|
| Hardware metrics & telemetry | In memory, shown live | No |
| Tweaks, cleanups, file ops | Your local system | No |
| App settings & preferences | Local file + browser storage on your PC | No |
| AI fix audit log | Local file on your PC | No (unless you opt in) |
| AI usage counters | Local log on your PC (hashed device ID) | No |
| Account profile & Pro status | Supabase (our database) | Yes - to run your account |
| AI chat context & message | Chosen AI provider | Yes - only when you chat |
| Cloud AI memory (optional) | Supabase, locked to your account | Only if you enable it |
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

## Your choices

- **Use it without an account.** Most local features work without signing
  in.
- **Skip the AI.** It's the only feature that sends system context off your
  device, and it's off until you open it.
- **Keep AI memory local.** Cloud memory is opt-in.
- **Bring your own AI key.** If you'd rather talk to a provider directly
  with your own key, you can set that in Settings.
- **Sign out / stop using it.** When you stop using the app, the local
  data stays on your machine until you remove it; you can uninstall to
  clear it.

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

---

## Questions

If you're reviewing GS Center (hi, XDA 👋) or just want clarification on
anything here, reach out through the project's GitHub or the contact
listed on our releases page. Happy to walk through any part of how the app
handles data.
