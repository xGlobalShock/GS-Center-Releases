LAYER 1: Electron Hardening (stop casual access)
├── electron-builder: devtools disabled in production
├── Electron Fuses (RunAsNode, EnableNodeCliInspect, cookie encryption)
├── javascript-obfuscator on main + preload bundles
├── contextIsolation: true (already done)
├── nodeIntegration: false (already done)
└── sandbox: true for all renderers

LAYER 2: IPC Authorization (stop unauthorized calls)
├── gatedHandler() wrapper on ALL premium IPC handlers
├── requireAuth() enforced before any privileged operation
├── requirePro() or requireTier(tier) for tier-gated features
├── Each handler explicitly declares required entitlement
└── No premium handler ever accessible without auth

LAYER 3: Signed Responses (stop fake server responses)
├── Supabase Edge Function signs license payload with Ed25519/RSA
├── Client verifies signature before trusting any license data
├── Unigned responses are rejected and logged
└── Signature includes expiration timestamp

LAYER 4: Hardware Binding (stop token theft + sharing)
├── Device fingerprint: GPU device ID + disk serial + mobo serial + machine GUID
├── Session token encrypted with AES key derived from fingerprint
├── Token only decryptable on original machine
├── Hardware change = token invalidated = re-authenticate
└── (This is the most powerful anti-sharing control available)

LAYER 5: Code Integrity (stop patched clients)
├── Preload script hash verified at startup against embedded expected hash
├── IPC handler registry hash verified at startup
├── DevTools opened in production → degrade to trial mode
├── Debugger attached → degrade to trial mode + silent phone-home
└── Suspicious runtime behavior → progressive degradation

LAYER 6: Offline Management (balance UX + security)
├── 24-hour offline grace (not 7 days)
├── Clock resets on every successful online validation
├── Offline mode triggers ONLY if online validation succeeded at least once
└── All premium features locked immediately after grace period expires

LAYER 7: Revocation + Fraud
├── Re-validate on EVERY app open (not just scheduled)
├── Immediate re-validation after any auth event
├── Rate limit: max 1 validation/hour in steady state, 5 during transitions
├── Suspicious patterns → temp block + email verification challenge
└── Revocation = app becomes unusable within 1 minute of next online check
