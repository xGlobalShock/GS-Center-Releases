# GC Center Hardened Security Architecture

---

# Executive Summary

This document defines the hardened security architecture for GC Center, implementing defense-in-depth principles to prevent piracy, account sharing, device limit abuse, and subscription bypassing. The architecture prioritizes practical security that stops casual and semi-determined attackers while maintaining a frictionless experience for legitimate customers.

**Key Principle:** Security must be layered. No single control is sufficient. Each layer assumes other layers may fail.

---

# Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     RENDERER PROCESS                        │
│   (Never Trusted - React UI Only)                          │
└─────────────────────┬───────────────────────────────────────┘
                      │ Secure IPC (contextBridge)
┌─────────────────────▼───────────────────────────────────────┐
│                     MAIN PROCESS                            │
│   (All Privileged Operations Here)                         │
│   ┌─────────────────────────────────────────────────────┐   │
│   │            License Manager (Singleton)               │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │   │
│   │  │ Session  │ │ Device   │ │ Entitlement         │  │   │
│   │  │ Validator│ │ Fingerprint│ │ Resolver            │  │   │
│   │  └──────────┘ └──────────┘ └──────────────────────┘  │   │
│   └─────────────────────────────────────────────────────┘   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │            Anti-Tamper Module                        │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │   │
│   │  │Integrity│ │Anti-Debug│ │ Native Monitor      │  │   │
│   │  │ Checker  │ │ Detector │ │ Verifier            │  │   │
│   │  └──────────┘ └──────────┘ └──────────────────────┘  │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────────┘
                      │ Authenticated API Calls
┌─────────────────────▼───────────────────────────────────────┐
│                    SUPABASE BACKEND                         │
│   (Row Level Security + Server Functions)                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  PostgreSQL + Realtime + Edge Functions             │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

# Security Layers (Defense in Depth)

## Layer 1: Application Hardening

### Electron Security Baseline

```javascript
// Main process BrowserWindow creation - PRODUCTION ONLY
const secureWindowConfig = {
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    contextIsolation: true,        // CRITICAL: Isolate preload from renderer
    nodeIntegration: false,         // CRITICAL: No Node in renderer
    sandbox: true,                  // CRITICAL: OS-level sandboxing
    webSecurity: true,             // Enforce same-origin policy
    allowRunningInsecureContent: false,
    enableRemoteModule: false,     // Disabled completely
    spellcheck: false,             // Reduce attack surface
  },
  show: false,                     // Hide window until ready
  backgroundThrottling: true,
  // Production hardening
  disableHardwareAcceleration: false, // Keep for performance
  titleBarStyle: 'default',
  webContents: {
    devTools: !isProduction,       // Disable DevTools in production
    // Prevent new window creation (popup blocking)
    partition: 'persist:main',
  }
};
```

### Build-Time Hardening

```javascript
// electron-builder configuration
const buildConfig = {
  // Electron Fuses - critical security fuses
  electronDownloads: undefined,
  afterPack: async (context) => {
    // Enable all Electron Fuses via @electron/fuses
    await flipFuses(
      path.join(outDir, 'GC Center.exe'),
      {
        runAsNode: false,           // Disable ELECTRON_RUN_AS_NODE
        enableCookieEncryption: true,
        enableNodeOptionsEnvironmentVariable: false,
        enableNodeCliInspectArguments: false,
        enableEmbeddedAsarIntegrityValidation: true,
        onlyLoadAppFromAsar: true,   // Must load from ASAR only
      }
    );
  }
};
```

### Code Obfuscation

- **Sensitive IPC handlers**: Obfuscated channel names (no plain text like `optimizer:run`)
- **Critical business logic**: Obfuscated in renderer, but main process logic stays readable for maintainability
- **Build output**: Minified JavaScript/TypeScript

### Native Binary Protection (GCMonitor.exe)

The native monitor runs as a separate process with:
- Signed binary verification before execution
- Communication via authenticated IPC channels only
- No direct renderer access
- Memory integrity checks

---

## Layer 2: Device Identity & Fingerprinting

### Hardware-Bound Identity

Device identity is computed from multiple hardware signals:

```
Device Token = HMAC-SHA256(
  Hardware Signature,
  Application Secret (embedded)
)
```

**Hardware Signature Components:**

| Component | Weight | Notes |
|-----------|--------|-------|
| TPM AIK Quote | 40% | If TPM 2.0 available |
| Motherboard Serial | 20% | Primary identifier |
| CPU ID | 15% | Secondary identifier |
| OS Installation ID | 15% | Windows-specific |
| Machine GUID | 10% | Fallback identifier |

**Scoring Algorithm:**

```
Match Score = (TPM_Match × 0.4) + (MB_Match × 0.2) + (CPU_Match × 0.15) +
              (OS_Match × 0.15) + (GUID_Match × 0.1)

If Match Score >= 0.75: Existing Device (allow)
If Match Score >= 0.50: Hardware Upgrade (prompt user, verify via email)
If Match Score < 0.50: New Device (full verification)
```

### Device Registration Flow

```
User Action            Server Response            Client Action
     │                       │                        │
     ├─ Login ─────────────► │                        │
     │                       │                        │
     │◄── Auth Token ─────── ┤                        │
     │                       │                        │
     ├─ Register Device ────►│                        │
     │  (fingerprint)        │                        │
     │                       ├─ Validate              │
     │                       │  - Subscription active?│
     │                       │  - Slot available?     │
     │                       │  - Not blocked?        │
     │                       │                        │
     │◄── Device Token ───── ┤                        │
     │  + Entitlements       │                        │
     │                       │                        │
     ▼                       ▼                        ▼
```

### Secure Device Storage

```javascript
// Use Electron's safeStorage for device token encryption
const { safeStorage } = require('electron');

function storeDeviceToken(token) {
  const encrypted = safeStorage.encryptString(token);
  // Store in app data, NOT in plaintext
  fs.writeFileSync(deviceTokenPath, encrypted);
}

function retrieveDeviceToken() {
  const encrypted = fs.readFileSync(deviceTokenPath);
  return safeStorage.decryptString(encrypted);
}
```

**safeStorage uses:**
- Windows: DPAPI (Data Protection API)
- macOS: Keychain
- Linux: libsecret

This ensures device tokens cannot be extracted from a user's machine and moved to another.

---

## Layer 3: Session & Authentication Security

### OAuth Flow (Twitch/Discord)

```
┌──────────────────────────────────────────────────────────┐
│                      OAUTH FLOW                          │
└──────────────────────────────────────────────────────────┘

1. User clicks "Login with Twitch/Discord"
2. Open external browser to OAuth provider
3. User authenticates + consents
4. OAuth provider returns authorization code
5. App receives code via deep link (gccenter://oauth)
6. App exchanges code for access token (via Supabase)
7. Supabase validates token with OAuth provider
8. Supabase issues JWT session token
9. App stores session token securely
```

### Session Token Structure

```json
{
  "sub": "user-uuid",
  "session_id": "session-uuid",
  "iat": 1234567890,
  "exp": 1234567890,
  "refresh_until": 1234567890,
  "device_id": "device-uuid",
  "entitlements": ["optimizer", "cleaner", "ai"],
  "subscription": {
    "type": "monthly",
    "tier": "pro",
    "expires": "2026-07-01"
  }
}
```

### Session Security Rules

| Rule | Value | Rationale |
|------|-------|-----------|
| Token lifetime | 1 hour | Short-lived access |
| Refresh window | 7 days | Allows offline grace period |
| Refresh token | Single-use | Prevents token replay |
| Session limit | 3 concurrent | Reasonable for multi-device users |
| Device binding | Required | Session tied to device token |

### Concurrent Session Enforcement

```sql
-- Supabase Edge Function: validate_session
CREATE FUNCTION validate_session(p_session_id UUID, p_device_id UUID)
RETURNS JSON AS $$
DECLARE
  v_session RECORD;
  v_active_sessions INT;
BEGIN
  -- Check session exists and is valid
  SELECT * INTO v_session FROM sessions
  WHERE id = p_session_id
    AND device_id = p_device_id
    AND revoked = false
    AND expires_at > NOW();

  IF NOT FOUND THEN
    RETURN json_build_object('valid', false, 'reason', 'invalid_session');
  END IF;

  -- Count active sessions for this user
  SELECT COUNT(*) INTO v_active_sessions FROM sessions
  WHERE user_id = v_session.user_id
    AND revoked = false
    AND expires_at > NOW();

  IF v_active_sessions > 3 THEN
    -- Revoke oldest sessions until under limit
    UPDATE sessions
    SET revoked = true
    WHERE id IN (
      SELECT id FROM sessions
      WHERE user_id = v_session.user_id
        AND revoked = false
      ORDER BY created_at ASC
      LIMIT v_active_sessions - 3
    );
  END IF;

  RETURN json_build_object(
    'valid', true,
    'session', v_session,
    'entitlements', get_user_entitlements(v_session.user_id)
  );
END;
$$ LANGUAGE plpgsql;
```

---

## Layer 4: Entitlement Management

### Entitlement Definition

Entitlements are server-controlled feature flags tied to subscription status:

```json
{
  "entitlements": {
    "optimizer": {
      "subscription_tiers": ["monthly", "annual", "lifetime"],
      "description": "System Optimizer access"
    },
    "cleaner": {
      "subscription_tiers": ["monthly", "annual", "lifetime"],
      "description": "Cleaner Engine access"
    },
    "ai": {
      "subscription_tiers": ["pro_monthly", "pro_annual", "lifetime"],
      "description": "AI Diagnostic Engine"
    },
    "obs_profiles": {
      "subscription_tiers": ["pro_monthly", "pro_annual", "lifetime"],
      "description": "OBS Profile Management"
    }
  }
}
```

### Entitlement Resolution (Server-Side)

```sql
CREATE FUNCTION get_user_entitlements(p_user_id UUID)
RETURNS JSON AS $$
DECLARE
  v_subscription RECORD;
  v_entitlements TEXT[];
BEGIN
  SELECT * INTO v_subscription FROM profiles
  WHERE id = p_user_id AND license_active = true AND license_revoked = false;

  IF NOT FOUND THEN
    RETURN '[]'::JSON;
  END IF;

  IF v_subscription.subscription_expires < NOW() THEN
    RETURN '[]'::JSON;
  END IF;

  -- Map subscription type to entitlements
  CASE v_subscription.subscription_type
    WHEN 'monthly' THEN
      v_entitlements := ARRAY['optimizer', 'cleaner'];
    WHEN 'pro_monthly' THEN
      v_entitlements := ARRAY['optimizer', 'cleaner', 'ai', 'obs_profiles'];
    WHEN 'annual' THEN
      v_entitlements := ARRAY['optimizer', 'cleaner'];
    WHEN 'pro_annual' THEN
      v_entitlements := ARRAY['optimizer', 'cleaner', 'ai', 'obs_profiles'];
    WHEN 'lifetime' THEN
      v_entitlements := ARRAY['optimizer', 'cleaner', 'ai', 'obs_profiles'];
    ELSE
      v_entitlements := ARRAY[]::TEXT[];
  END CASE;

  RETURN to_json(v_entitlements);
END;
$$ LANGUAGE plpgsql;
```

### IPC Entitlement Guard

```javascript
// All premium IPC handlers MUST go through this guard
function gatedHandler(featureName, handler) {
  return async (event, ...args) => {
    // 1. Verify session exists (not just renderer calling)
    const session = await getSession(event.sender);
    if (!session) {
      throw new Error('UNAUTHORIZED');
    }

    // 2. Verify entitlement
    if (!session.entitlements.includes(featureName)) {
      throw new Error('ENTITLEMENT_REQUIRED');
    }

    // 3. Log attempt (for fraud detection)
    await logFeatureAccess(session.userId, featureName);

    // 4. Execute handler
    return handler(event, ...args);
  };
}

// Usage
ipcMain.handle('premium:optimizer', gatedHandler('optimizer', runOptimizer));
ipcMain.handle('premium:ai', gatedHandler('ai', runAIEngine));
```

---

## Layer 5: Anti-Tamper & Integrity

### Runtime Integrity Verification

```javascript
// Startup integrity check
async function verifyApplicationIntegrity() {
  const checks = [
    checkPreloadHash(),      // Verify preload hasn't been modified
    checkMainProcessHash(),  // Verify main process files
    checkNativeMonitor(),    // Verify GCMonitor.exe signature
    checkAsarIntegrity(),   // Verify ASAR archive integrity
  ];

  const results = await Promise.all(checks);
  const allPassed = results.every(r => r.valid);

  if (!allPassed) {
    // Log failure details (never show user specifics)
    await logSecurityEvent('INTEGRITY_CHECK_FAILED', {
      failed: results.filter(r => !r.valid).map(r => r.name)
    });

    // In production: deny startup or limit functionality
    if (isProduction) {
      throw new Error('Application integrity check failed');
    }
  }
}

// Hash verification using crypto
async function checkPreloadHash() {
  const expectedHash = process.env.PRELOAD_HASH; // Set at build time
  const actualHash = await computeFileHash(preloadPath);
  return {
    name: 'preload',
    valid: timingSafeEqual(expectedHash, actualHash)
  };
}
```

### Anti-Debug Protection

```javascript
// main.js - Anti-debug measures
function enableAntiDebug() {
  // Detect DevTools attachment
  win.webContents.on('devtools-opened', () => {
    if (isProduction) {
      // In production, this should never happen
      logSecurityEvent('DEVTOOLS_OPENED_PRODUCTION');
    }
  });

  // Detect debugger attachment via built-in Electron protection
  // Note: This is not foolproof - determined attackers can bypass
  // But it stops casual debugging attempts

  // Block common debugging ports
  const originalListen = net.Server.prototype.listen;
  net.Server.prototype.listen = function(...args) {
    const port = args[0];
    if (port === 9222 || port === 9223 || port === 5858) {
      logSecurityEvent('DEBUG_PORT_BLOCKED', { port });
      return originalListen.apply(this, [0, ...args.slice(1)]);
    }
    return originalListen.apply(this, args);
  };
}
```

### Native Monitor Verification

```csharp
// GCMonitor.exe verifies itself before running
// In Program.cs
public static bool VerifySignature() {
    try {
        var cert = X509Certificate.CreateFromSignedFile(Assembly.GetExecutingAssembly().Location);
        // Verify signed by GC Center's certificate
        return cert.Subject.Contains("CN=GC Center");
    } catch {
        Environment.Exit(1); // Die if not properly signed
    }
}
```

### Modified Client Detection

```javascript
// Detect if client has been patched by checking critical function integrity
const criticalFunctions = [
  'gatedHandler',
  'verifyApplicationIntegrity',
  'checkPreloadHash'
];

async function detectPatchedClient() {
  for (const fn of criticalFunctions) {
    const hash = await computeFunctionHash(fn);
    if (!knownHashes[fn].includes(hash)) {
      // Function has been modified
      await logSecurityEvent('CLIENT_MODIFIED', {
        function: fn,
        detectedHash: hash
      });
      return true;
    }
  }
  return false;
}
```

---

## Layer 6: Offline Security

### Cached Credential System

**WARNING:** Offline caching is a security boundary. Must be implemented carefully.

```javascript
// Secure offline cache using safeStorage
class OfflineCredentialCache {
  constructor() {
    this.cacheExpiry = 7 * 24 * 60 * 60 * 1000; // 7 days
  }

  async cacheForOffline(tokenData) {
    const cache = {
      entitlements: tokenData.entitlements,
      subscription: tokenData.subscription,
      deviceId: tokenData.deviceId,
      cachedAt: Date.now(),
      expiresAt: Date.now() + this.cacheExpiry,
      // Include short verification hash (not the actual token)
      verifyHash: crypto.createHash('sha256')
        .update(JSON.stringify(tokenData) + secretSalt)
        .digest('hex').substring(0, 16)
    };

    const encrypted = safeStorage.encryptString(JSON.stringify(cache));
    fs.writeFileSync(this.cachePath, encrypted);
  }

  async getOfflineStatus() {
    try {
      const encrypted = fs.readFileSync(this.cachePath);
      const cache = JSON.parse(safeStorage.decryptString(encrypted));

      if (Date.now() > cache.expiresAt) {
        return { valid: false, reason: 'expired' };
      }

      // Re-validate online when connection available
      if (navigator.onLine) {
        const onlineStatus = await revalidateOnline(cache);
        if (!onlineStatus.valid) {
          return onlineStatus;
        }
      }

      return { valid: true, entitlements: cache.entitlements };
    } catch {
      return { valid: false, reason: 'no_cache' };
    }
  }
}
```

### Offline Usage Rules

| Condition | Behavior |
|-----------|----------|
| Within 7 days | Full access with cached credentials |
| After 7 days | Premium features disabled, re-auth required |
| Online available | Always validate fresh, never skip |
| Revocation received | Disable immediately on next online check |

---

## Layer 7: Server-Side Security (Supabase)

### Row Level Security (RLS)

```sql
-- Enable RLS on all tables
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_devices ENABLE ROW LEVEL SECURITY;
ALTER TABLE sessions ENABLE ROW LEVEL SECURITY;

-- Profiles: Users can only read/write their own profile
CREATE POLICY "Users can view own profile"
  ON profiles FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
  ON profiles FOR UPDATE
  USING (auth.uid() = id);

-- Devices: Users can manage their own devices
CREATE POLICY "Users can manage own devices"
  ON user_devices FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Sessions: Users can only see their own sessions
CREATE POLICY "Users can view own sessions"
  ON sessions FOR SELECT
  USING (auth.uid() = user_id);
```

### API Rate Limiting

```javascript
// Supabase Edge Function with rate limiting
// supabase/functions/licensing/index.ts
import { Ratelimit } from '@supabase/supabase-js';

const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, '60 s'), // 10 requests per minute per user
});

export async function licensing(req: Request): Promise<Response> {
  const { user_id } = await req.json();

  const { success, remaining } = await ratelimit.limit(user_id);

  if (!success) {
    return new Response(JSON.stringify({
      error: 'RATE_LIMITED',
      retryAfter: remaining
    }), { status: 429 });
  }

  // ... rest of licensing logic
}
```

### Input Validation (All Edge Functions)

```typescript
// Every user input must be validated
interface DeviceRegistration {
  device_fingerprint: string;  // 64 char hex
  device_name: string;         // Max 64 chars, alphanumeric + spaces
}

function validateDeviceRegistration(input: unknown): DeviceRegistration {
  const data = input as DeviceRegistration;

  // Type check
  if (typeof data !== 'object' || data === null) {
    throw new Error('Invalid input');
  }

  // Fingerprint: exactly 64 hex characters
  if (!/^[a-f0-9]{64}$/i.test(data.device_fingerprint)) {
    throw new Error('Invalid fingerprint format');
  }

  // Device name: sanitize and length check
  const name = String(data.device_name || '').trim();
  if (name.length < 1 || name.length > 64) {
    throw new Error('Invalid device name');
  }

  return {
    device_fingerprint: name.toLowerCase(),
    device_name: sanitize(name)
  };
}
```

### Database Audit Logging

```sql
-- Security events table (append-only)
CREATE TABLE security_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES profiles(id),
  event_type TEXT NOT NULL,
  event_data JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Never delete from this table
-- Implement using database triggers or application logging

CREATE POLICY "Admin can read security events"
  ON security_events FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE id = auth.uid()
        AND is_admin = true
    )
  );
```

---

## Layer 8: Fraud Detection

### Anomaly Detection Rules

```sql
-- Monitor excessive device activations
CREATE FUNCTION detect_excessive_devices(p_user_id UUID)
RETURNS JSON AS $$
DECLARE
  v_device_count INT;
  v_activation_rate FLOAT;
BEGIN
  SELECT COUNT(*),
    COUNT(*)::FLOAT / EXTRACT(EPOCH FROM (NOW() - MIN(created_at))) * 86400
  INTO v_device_count, v_activation_rate
  FROM user_devices
  WHERE user_id = p_user_id;

  -- More than 3 devices in 24 hours is suspicious
  IF v_device_count > 3 AND v_activation_rate > 3 THEN
    RETURN json_build_object(
      'anomaly', true,
      'type', 'excessive_devices',
      'count', v_device_count,
      'rate', v_activation_rate
    );
  END IF;

  RETURN json_build_object('anomaly', false);
END;
$$ LANGUAGE plpgsql;

-- Monitor unusual validation patterns
CREATE FUNCTION detect_unusual_validations(p_user_id UUID)
RETURNS JSON AS $$
DECLARE
  v_recent_validations INT;
  v_unique_ips INT;
BEGIN
  SELECT COUNT(*), COUNT(DISTINCT ip_address)
  INTO v_recent_validations, v_unique_ips
  FROM validation_logs
  WHERE user_id = p_user_id
    AND created_at > NOW() - INTERVAL '1 hour';

  -- Single user validating from 5+ IPs in 1 hour is suspicious
  IF v_unique_ips > 5 THEN
    RETURN json_build_object(
      'anomaly', true,
      'type', 'multiple_ips',
      'ips', v_unique_ips
    );
  END IF;

  RETURN json_build_object('anomaly', false);
END;
$$ LANGUAGE plpgsql;
```

### Automated Response Matrix

| Detection | Severity | Automated Response |
|-----------|----------|-------------------|
| 3+ devices in 24h | Medium | Flag for review, allow with warning |
| 5+ IPs in 1h | High | Temporarily suspend, require email verification |
| 10+ validation failures | Medium | Rate limit user |
| Modified client detected | Critical | Revoke all sessions, require re-auth |
| Revocation signal received | Critical | Immediate disable |

---

## Layer 9: Revocation & Incident Response

### Instant Revocation System

```sql
-- License revocation stored server-side (not client-cached)
CREATE TABLE license_revocations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID UNIQUE REFERENCES profiles(id),
  reason TEXT NOT NULL,
  revoked_at TIMESTAMP DEFAULT NOW(),
  revoked_by UUID REFERENCES profiles(id)
);

-- Check during every validation
CREATE FUNCTION is_license_revoked(p_user_id UUID)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM license_revocations WHERE user_id = p_user_id
  );
END;
$$ LANGUAGE plpgsql;
```

### Revocation Propagation

```
Revocation Event (Admin Action)
        │
        ▼
Update license_revocations table
        │
        ▼
On user's next validation (within 60s)
        │
        ▼
Server returns revoked=true
        │
        ▼
Client immediately disables premium
        │
        ▼
Session token invalidated
        │
        ▼
Device token invalidated
```

### Emergency Shutdown

If critical vulnerability or mass abuse detected:

```javascript
// Edge function: check_emergency_flags
export async function checkEmergencyFlags(): Promise<{
  globalShutdown: boolean;
  minimumVersion: string;
}> {
  // Fetched from a hardcoded config endpoint
  // Can be toggled instantly server-side
  return {
    globalShutdown: false, // If true, all clients shut down
    minimumVersion: '1.0.0'
  };
}
```

---

## Layer 10: Code Signing & Release Security

### Authenticode Signing (Windows)

```javascript
// Required for all releases
const signConfig = {
  certificate: process.env.SIGNING_CERT_PATH,
  timestampServer: 'http://timestamp.digicert.com',
  algorithm: 'sha256',
  // EV certificate preferred for SmartScreen reputation
  // Subject: CN=GC Center, O=GC Center, C=US
};
```

### Release Integrity

```javascript
// Build output verification
const buildArtifacts = [
  'GC Center Setup 1.0.0.exe',
  'GC Center 1.0.0.exe',
  'GCMonitor.exe'
];

async function verifyReleaseIntegrity() {
  for (const artifact of buildArtifacts) {
    const hash = await computeHash(path.join(outDir, artifact));
    console.log(`${artifact}: ${hash}`);
    // Publish hashes to release notes + website
  }
}
```

---

# Database Schema (Hardened)

```sql
-- Core tables with security considerations

CREATE TABLE profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- Auth (managed by Supabase Auth)

  -- Subscription state
  subscription_type TEXT NOT NULL DEFAULT 'none',
  subscription_expires TIMESTAMPTZ,
  license_active BOOLEAN NOT NULL DEFAULT false,
  license_revoked BOOLEAN NOT NULL DEFAULT false,
  revocation_reason TEXT,

  -- Device management
  device_limit INTEGER NOT NULL DEFAULT 1,

  -- Admin fields
  is_admin BOOLEAN NOT NULL DEFAULT false,
  is_internal BOOLEAN NOT NULL DEFAULT false, -- For internal accounts

  -- Metadata
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- Constraints
  CONSTRAINT valid_subscription_type CHECK (
    subscription_type IN ('none', 'monthly', 'pro_monthly', 'annual',
                          'pro_annual', 'lifetime')
  )
);

CREATE TABLE user_devices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,

  -- Device identity
  device_fingerprint TEXT NOT NULL, -- SHA-256 hash
  device_token_encrypted BYTEA,    -- Encrypted device token

  -- Device metadata
  device_name TEXT NOT NULL,
  os_version TEXT,

  -- State
  is_active BOOLEAN NOT NULL DEFAULT true,
  is_blocked BOOLEAN NOT NULL DEFAULT false,

  -- Timestamps
  activated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_validation_at TIMESTAMPTZ,

  -- Constraints
  CONSTRAINT unique_active_device_per_user UNIQUE (user_id, device_fingerprint)
    WHERE is_active = true
);

CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  device_id UUID NOT NULL REFERENCES user_devices(id) ON DELETE CASCADE,

  -- Token data
  refresh_token_hash TEXT NOT NULL, -- Never store raw refresh token
  access_token_nonce TEXT,

  -- State
  revoked BOOLEAN NOT NULL DEFAULT false,
  revoked_at TIMESTAMPTZ,

  -- Expiration
  expires_at TIMESTAMPTZ NOT NULL,
  refresh_expires_at TIMESTAMPTZ NOT NULL,

  -- Timestamps
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_used_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- Metadata
  ip_address INET,
  user_agent TEXT,

  -- Constraints
  CONSTRAINT session_expires_future CHECK (expires_at > created_at)
);

CREATE TABLE license_revocations (
  user_id UUID PRIMARY KEY REFERENCES profiles(id) ON DELETE CASCADE,
  reason TEXT NOT NULL,
  revoked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_by UUID REFERENCES profiles(id)
);

CREATE TABLE security_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Event identification
  event_type TEXT NOT NULL,
  severity TEXT NOT NULL DEFAULT 'info',

  -- Context
  user_id UUID REFERENCES profiles(id) ON DELETE SET NULL,
  device_id UUID REFERENCES user_devices(id) ON DELETE SET NULL,
  session_id UUID REFERENCES sessions(id) ON DELETE SET NULL,

  -- Event data (flexible JSONB)
  event_data JSONB NOT NULL DEFAULT '{}',

  -- Request context
  ip_address INET,
  user_agent TEXT,

  -- Timestamp
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  -- Constraints
  CONSTRAINT valid_severity CHECK (
    severity IN ('debug', 'info', 'warning', 'error', 'critical')
  )
);

-- Indexes for security queries
CREATE INDEX idx_sessions_user_active ON sessions(user_id) WHERE revoked = false;
CREATE INDEX idx_devices_user_active ON user_devices(user_id) WHERE is_active = true;
CREATE INDEX idx_security_events_user ON security_events(user_id, created_at DESC);
CREATE INDEX idx_security_events_type ON security_events(event_type, created_at DESC);

-- Trigger: Update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER profiles_updated_at
  BEFORE UPDATE ON profiles
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

# Security Checklist

## Development Phase
- [ ] Context isolation enabled
- [ ] Node integration disabled
- [ ] Sandbox enabled
- [ ] IPC handlers use gatedHandler
- [ ] All user input validated server-side
- [ ] RLS policies on all tables
- [ ] No sensitive data in renderer process

## Build Phase
- [ ] Electron Fuses configured
- [ ] Code minified
- [ ] Preload hash computed and embedded
- [ ] Native monitor signed
- [ ] Application signed (Authenticode)

## Release Phase
- [ ] Release checksums published
- [ ] SmartScreen reputation established
- [ ] Version verification endpoint updated

## Monitoring Phase
- [ ] Security events logging operational
- [ ] Anomaly detection active
- [ ] Alert thresholds configured
- [ ] Revocation system tested

---

# Security Assumptions

This architecture is designed to stop:
- ✓ Casual piracy
- ✓ License sharing via account password sharing
- ✓ Device limit abuse
- ✓ Subscription bypassing
- ✓ Basic client modification
- ✓ Automated abuse at scale
- ✓ Fake license generation

This architecture does NOT guarantee protection against:
- ✗ Determined reverse engineers with time/resources
- ✗ Custom patched builds with disabled security checks
- ✗ Compromised OAuth provider accounts
- ✗ Insiders with database access
- ✗ Physical access to a legitimately licensed machine

---

# Implementation Priority

## Phase 1 (Critical - Must Have)
1. Electron hardening (context isolation, no node integration)
2. IPC entitlement guards
3. Session validation
4. RLS policies

## Phase 2 (Important)
5. Device fingerprinting with safeStorage
6. Offline cache with expiration
7. Signed validation responses
8. Anomaly detection rules

## Phase 3 (Enhanced)
9. Integrity verification (hash checks)
10. Anti-debug measures
11. Native monitor verification
12. Emergency shutdown capability

## Phase 4 (Future)
13. TPM-based device identity
14. Hardware key storage
15. Enterprise licensing features
16. Advanced fraud scoring

---

# Conclusion

This hardened architecture implements defense-in-depth across ten distinct security layers. Each layer addresses specific attack vectors while maintaining usability for legitimate customers. The architecture assumes the renderer process is completely untrusted, all server responses can be inspected, and determined attackers may bypass individual controls—but bypassing multiple layers simultaneously requires significant effort.

The goal is not to create an unbreakable system (impossible in practice), but to make unauthorized access so time-consuming and effort-intensive that casual pirates and abuse rings move on to easier targets.
