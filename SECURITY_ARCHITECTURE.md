# GC Center Licensing & Security Architecture

---

# Executive Summary

GC Center will implement a layered licensing and security architecture designed to:

* Prevent casual piracy
* Prevent account sharing abuse
* Limit device activations
* Protect premium features
* Allow offline usage for legitimate customers
* Maintain a smooth user experience

The goal is not to create an unbreakable application. The goal is to make unauthorized use significantly more difficult while keeping the software convenient for paying customers.

---

# Design Principles

## Security Goals

Protect against:

* Unauthorized copying
* License sharing
* Modified clients
* Fake license responses
* Mass account abuse
* Subscription bypassing

## User Experience Goals

Avoid:

* Frequent login prompts
* Excessive online requirements
* Hardware-upgrade lockouts
* Complex activation procedures

---

# High-Level Architecture

```text
React Renderer
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
Database + Validation Services
```

The renderer process is never trusted.

All premium operations are authorized by the main process.

---

# Licensing Model

## Subscription Tiers

### Monthly

* Price: $4.99/month
* Device Limit: 1

* Price: $14.99/month
* Device Limit: 1

* Price: $29.99/month
* Device Limit: 1

### Annual

* Price: $tbd/year
* Device Limit: tdb

### Lifetime

* Price: $299.99 one-time
* Device Limit: 3

---

# Account-Based Licensing

GC Center uses account-based licensing.

Users authenticate through:

* OAuth providers Twitch / Discord

The account itself acts as the license.

No traditional serial key is required.

license redemption codes may be used for promotions or reseller sales.

---

# Database Schema

## profiles

```sql
id UUID PRIMARY KEY

subscription_type TEXT

device_limit INTEGER

license_active BOOLEAN

license_revoked BOOLEAN

subscription_expires TIMESTAMP

created_at TIMESTAMP

updated_at TIMESTAMP
```

---

## user_devices

```sql
id UUID PRIMARY KEY

user_id UUID

device_fingerprint TEXT

device_name TEXT

activated_at TIMESTAMP

last_seen TIMESTAMP

is_active BOOLEAN
```

---

# Device Registration System

## First Launch

User logs in.

Application sends:

```json
{
  "device_fingerprint": "...",
  "device_name": "Gaming PC"
}
```

Server validates:

* Active subscription
* Device limit availability

If approved:

* Device registered
* Slot consumed

---

# Device Management

Users can manage devices through:

```text
Settings
 └─ Manage Devices
```

Capabilities:

* View active devices
* Rename devices
* Remove devices
* Free activation slots

---

# Device Fingerprinting

GC Center uses a tolerant device fingerprint.

Inputs:

* Motherboard identifier
* CPU identifier
* TPM identifier (when available)
* Windows Machine GUID

A scoring system is used.

Example:

```text
80% Match = Existing Device
Below 80% = New Device
```

This avoids unnecessary lockouts after minor hardware upgrades.

---

# License Validation Flow

## Startup Validation

```text
Application Launch
        │
        ▼
Load Cached Token
        │
        ▼
Contact Supabase
        │
        ▼
Validate Account
Validate Subscription
Validate Device
Validate Entitlements
        │
        ▼
Grant Access
```

---

# Signed Validation Responses

Server returns:

```json
{
  "payload": {
    "licensed": true,
    "tier": "lifetime",
    "device_limit": 5,
    "features": [
      "optimizer",
      "cleaner",
      "ai"
    ]
  },
  "signature": "RSA_SIGNATURE"
}
```

Client verifies:

* Signature authenticity
* Expiration timestamp

Only responses signed by GC Center are trusted.

---

# Feature Entitlements

Premium features are controlled through entitlements.

Example:

```json
{
  "features": [
    "optimizer",
    "cleaner",
    "ai",
    "obs_profiles"
  ]
}
```

Every premium action checks entitlement before execution.

---

# IPC Security Layer

All premium operations are routed through secured IPC handlers.

Example:

```javascript
ipcMain.handle(
  'optimizer:run',
  gatedHandler(runOptimizer)
);
```

Validation occurs before execution.

Unauthorized requests are rejected.

---

# Protected Features

The following require valid entitlements:

* System Optimizer
* Registry Tweaks
* Cleaner Engine
* AI Diagnostic Engine
* OBS Profiles
* Driver Intelligence
* Performance Monitoring
* Premium Presets

---

# Offline Support

## Grace Period

Users may operate offline for:

```text
7 Days
```

after successful validation.

---

## Offline Token

Validation data is cached locally.

```json
{
  "validated": true,
  "expires": "2026-12-31"
}
```

Once expired:

* Online validation required
* Premium features disabled until renewed

---

# Revocation System

Administrators may revoke licenses instantly.

Reasons:

* Chargebacks
* Fraud
* Abuse
* Terms violations

Revocation propagates globally on next validation.

---

# Fraud Detection

Monitor:

* Excessive device activations
* Unusual validation patterns
* Geographic anomalies
* Automated behavior

Potential actions:

* Warning
* Temporary suspension
* Manual review
* Revocation

---

# Application Hardening

## Electron Security

Production builds should:

* Disable DevTools
* Enable Electron Fuses
* Enable Context Isolation
* Disable Node Integration
* Minify code
* Obfuscate sensitive logic

---

## Secure IPC

Renderer process cannot directly execute privileged operations.

All actions must pass through:

```text
Renderer
   │
   ▼
IPC
   │
   ▼
Main Process
   │
   ▼
Authorization
   │
   ▼
Execution
```

---

# Code Signing

All releases must be Authenticode signed.

Benefits:

* SmartScreen reputation
* User trust
* Download authenticity

Code signing is not considered an anti-piracy mechanism.

---

# Monitoring

Track:

* Validation success rate
* Device registrations
* License revocations
* Subscription conversions
* Offline usage frequency

Data should remain privacy-conscious and minimal.

---

# Security Assumptions

This architecture is designed to stop:

✓ Casual piracy

✓ License sharing

✓ Subscription abuse

✓ Fake license generation

✓ Basic client modification

✓ Automated abuse

This architecture does not guarantee protection against:

✗ Professional cracking groups

✗ Custom patched builds

✗ Determined reverse engineers

---

# Success Criteria

The system is considered successful if it:

* Protects subscription revenue
* Prevents large-scale account sharing
* Minimizes support burden
* Maintains a positive customer experience
* Requires minimal ongoing maintenance

---

# Future Enhancements

Potential future additions:

* Cloud optimization profiles
* Account synchronization
* Premium AI services
* Device analytics dashboard
* Advanced fraud scoring
* Enterprise licensing

---

# Conclusion

GC Center will use a layered security model centered around account-based licensing, device entitlements, signed server validation, IPC authorization, and secure Electron practices.

This approach provides strong practical protection while remaining user-friendly and maintainable.
