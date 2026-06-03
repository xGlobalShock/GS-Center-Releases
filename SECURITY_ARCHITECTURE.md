# GC Center Licensing & Security Architecture

## Overview

GC Center uses an **account-based licensing system** with server-side validation and device limits.

The goal is to prevent license sharing, casual piracy, and unauthorized access to premium features while keeping the experience simple for legitimate users.

---

# Architecture

```text
React UI (Renderer)
        │
        ▼
Electron IPC
        │
        ▼
Electron Main Process
        │
        ▼
License Manager
        │
        ▼
Supabase Backend
        │
        ▼
Database
```

The renderer is never trusted.

All premium actions are authorized by the Electron Main Process.

---

# Licensing Flow

## User Login

```text
Login
   │
   ▼
Supabase Auth
   │
   ▼
Retrieve Subscription
   │
   ▼
Validate Device
```

Subscription tiers determine how many devices can use the account.

Example:

```text
Monthly  → 1 Device
Annual   → 3 Devices
Lifetime → 5 Devices
```

---

# Device Registration

When GC Center launches for the first time on a PC:

```text
Login
   │
   ▼
Generate Device Fingerprint
   │
   ▼
Check Device Limit
   │
   ▼
Register Device
```

The device is then linked to the user's account.

Users can manage and remove devices from their account settings.

---

# License Validation

On startup:

```text
Launch App
   │
   ▼
Load Cached License
   │
   ▼
Contact Server
   │
   ▼
Validate:
 • Account
 • Subscription
 • Device
 • Entitlements
```

If validation succeeds:

```text
Access Granted
```

Otherwise:

```text
Premium Features Locked
```

---

# Signed Responses

The server returns a signed license response:

```json
{
  "licensed": true,
  "tier": "lifetime",
  "features": [
    "optimizer",
    "cleaner",
    "ai"
  ],
  "signature": "..."
}
```

The client verifies the signature before trusting the response.

This prevents fake or modified license responses.

---

# IPC Security

Every premium feature is protected by a license check.

```text
User Clicks Feature
        │
        ▼
IPC Request
        │
        ▼
License Validation
        │
        ▼
Execute Action
```

Examples:

* Optimizer
* Cleaner
* AI Fixes
* OBS Presets
* Premium Tweaks

Without a valid license, the action never executes.

---

# Offline Support

After a successful validation:

```text
License Cached
       │
       ▼
7-Day Offline Grace Period
```

Users can continue using the application while offline.

After the grace period expires, online validation is required again.

---

# Electron Security

Production builds should include:

* Context Isolation
* Disabled Node Integration
* Electron Fuses
* Code Minification
* Code Obfuscation

This makes reverse engineering significantly harder.

---

# Revocation System

Administrators can instantly disable a license.

```text
Revoke License
       │
       ▼
Server Updates Status
       │
       ▼
Next Validation Locks Access
```

Useful for chargebacks, abuse, or fraud.

---

# What This Protects Against

✅ License sharing

✅ Casual piracy

✅ Device abuse

✅ Fake license responses

✅ Basic client modification

✅ Subscription bypass attempts

---

# What It Does Not Protect Against

❌ Dedicated cracking groups

❌ Fully modified custom builds

❌ Advanced reverse engineering

---

# Summary

GC Center's security model is built around:

1. Account-based licensing
2. Device registration limits
3. Signed server validation
4. IPC authorization
5. Offline grace periods
6. Electron hardening

This provides strong real-world protection while remaining user-friendly and maintainable.
