# Sentinel Cryptographic & Enclave Security Architecture (SCESA)
## Detailed Security Whitepaper & Threat Model

This document outlines the security architecture of the GS Center application. Designed from a defense-in-depth perspective, SCESA is engineered to withstand reverse engineering, client-side cracking, binary tampering, and dynamic analysis by shifting critical application controls from a high-level, interpreted scripting runtime into a compiled, self-defending native enclave.

---

## 1. Threat Model & Security Scope

SCESA assumes a **Compromised Client / Full Privilege Host** threat model. The local attacker is assumed to have local Administrator privileges, access to kernel-level tools, debuggers, memory dumpers, and network interception proxies.

### Target Threat Vectors
1. **Static Analysis & Package Tampering**: Unpacking the Electron ASAR file, injecting malicious JavaScript hooks, and repacking the archive.
2. **Dynamic Analysis & Hooking**: Attaching Chrome DevTools, using Node inspection arguments (`--inspect`), or injecting hooks via frameworks like Frida or MinHook.
3. **Managed Code Decompilation**: Decompiling the .NET-based native monitor (`GCMonitor.exe`) into readable C# Intermediate Language (IL) via tools like dnSpy or ILSpy, followed by IL patching.
4. **Active Runtime Debugging**: Attaching native debuggers (e.g., x64dbg) to set software/hardware breakpoints in either the main Electron process or the C# monitor.
5. **Network Replay & API Mocking**: Intercepting licensing and auth payloads via local proxies (e.g., Fiddler, Charles, mitmproxy) and spoofing successful responses.

---

## 2. Architecture Overview

SCESA establishes a Native Enclave Boundary within the client system, enforcing hard cryptographic gating. Instead of verifying state via simple conditional branches in JavaScript, the application's core functionality is encrypted and cannot execute unless the environment passes native validation.

```mermaid
flowchart TB
    subgraph Client System (Windows - Host OS)
        subgraph Renderer Process (Untrusted)
            UI[React UI / HTML5]
        end
        
        subgraph Main Process (Trusted Orchestrator)
            Main[Electron Main Loop]
            VM[Node.js V8 VM Context]
        end
        
        subgraph Native Enclave (Self-Defending Boundary)
            Addon[C++ Sentinel.node]
            Monitor[Obfuscated GCMonitor.exe]
        end
        
        UI <-->|Isolated contextBridge IPC| Main
        Main <-->|Node-API stable ABI| Addon
        Addon <-->|Encrypted IPC| Monitor
        Main -.->|Loads decrypted modules| VM
    end
    
    subgraph Cloud Backend
        Main <-->|HTTPS Challenge-Response| Supa[Supabase Edge Functions]
    end

    classDef native fill:#1e293b,stroke:#3b82f6,stroke-width:2px;
    classAddon,Monitor native;
```

---

## 3. Cryptographic Enclave & Volatile Execution

### 3.1 Build-Time Module Encryption
To prevent static analysis of the Electron application code, core business logic and system optimization scripts (such as [tweaks.js](file:///e:/Dev/GC%20Center/main-process/tweaks.js) and [cleaners.js](file:///e:/Dev/GC%20Center/main-process/cleaners.js)) are processed through an offline build-time encryption pipeline:
* **Cipher**: **ChaCha20** (a high-performance, cryptographically secure stream cipher).
* **Key Size**: 256-bit symmetric key.
* **Nonce**: 96-bit unique initialization vector (padded to 128-bit/16-byte IV for standard OpenSSL compatibility, where the last 32 bits represent the initial block counter set to `0`).
* **Output**: Plaintext source files are completely omitted from the production package, replaced with binary resource files `tweaks.enc` and `cleaners.enc` under the `/lib` folder.

### 3.2 Ephemeral In-Memory Execution (Cryptographic Gating)
At runtime, the application decrypts and loads modules dynamically inside volatile memory:
1. The main process loader in [main.js](file:///e:/Dev/GC%20Center/electron/main.js) reads the `.enc` binary resources.
2. It passes the encrypted buffer along with key/nonce structures to [sentinel.cpp](file:///e:/Dev/GC%20Center/src/native-addon/sentinel.cpp) (loaded via `sentinel.node`).
3. The C++ addon performs decryption in-place in native heap memory.
4. The decrypted JavaScript source string is returned to Node.js and evaluated inside an isolated, sandboxed execution context using Node's `vm.Script`.
5. **Memory Scrubbing**: All intermediate C++ and JS buffers holding decrypted code are explicitly zero-filled (scrubbed) immediately after compilation to prevent memory dump extraction.

---

## 4. Native Self-Defense Mechanisms

The compiled C++ addon ([sentinel.cpp](file:///e:/Dev/GC%20Center/src/native-addon/sentinel.cpp)) and the native monitor helper ([AntiDebug.cs](file:///e:/Dev/GC%20Center/native-monitor/AntiDebug.cs)) contain active defense systems:

### 4.1 Anti-Debugging
The application performs startup and periodic (every 5 seconds) security sweeps checking for debuggers:
* **PEB Checks**: Direct memory read of the Process Environment Block (PEB) on x64 architectures. The C++ code reads the `BeingDebugged` flag and the `NtGlobalFlag` (verifying the flag is not set to `0x70`, which indicates a heap debugger attachment).
* **Remote Debugger Scans**: Utilizes the native Windows API `CheckRemoteDebuggerPresent` to detect if the process is being debugged by another user-mode process.
* **Hardware Breakpoint Detection**: Queries the active thread's execution context via `GetThreadContext` with the `CONTEXT_DEBUG_REGISTERS` flag. It inspects CPU debug registers `Dr0`, `Dr1`, `Dr2`, and `Dr3`. If any register holds a non-zero address, it indicates a hardware breakpoint has been set by an analyst.
* **Response**: If any check fails, the application calls `ExitProcess(0)` silently or triggers an access violation to terminate before analysis tools can hook the logic.

### 4.2 Anti-Virtualization (Sandbox Detection)
To prevent automated sandbox analysis or emulation, the C++ code reads registry keys under `HKLM\HARDWARE\DESCRIPTION\System\BIOS`:
* It queries bios metadata strings such as `SystemProductName`, `SystemManufacturer`, and `BIOSVersion`.
* It scans for virtualized hardware vendor signatures (e.g., `vmware`, `virtualbox`, `vbox`, `qemu`, `xen`, `hyper-v`, `parallels`). If detected, the application aborts execution.

### 4.3 Hardware Binding (Device Fingerprinting)
To mitigate account-sharing and token theft, the native addon binds sessions to physical devices:
1. Queries the physical drive volume serial number of the C:\ drive using `GetVolumeInformationW`.
2. Queries the CPU hardware signature using the C++ intrinsic `__cpuid` function.
3. Retrieves the motherboard chassis serial number via registry BIOS values (`BaseBoardSerialNumber` or `SystemSerialNumber`).
4. Combines these immutable physical hardware parameters into a single high-entropy string and hashes it with SHA-256. This signature is exposed via [auth.js](file:///e:/Dev/GC%20Center/main-process/auth.js) IPC (`auth:get-fingerprint`), ensuring licensing parameters are bound to physical hardware.

---

## 5. .NET Assembly Obfuscation & Hardening

Because standard .NET assemblies compile to Intermediate Language (IL) that retains rich metadata, `GCMonitor.exe` is protected using advanced obfuscation techniques:

### 5.1 MSBuild-Hooked Obfuscation Pipeline
* **Tooling**: NuGet **Obfuscar** (integrated into [GCMonitor.csproj](file:///e:/Dev/GC%20Center/native-monitor/GCMonitor.csproj) and configured in [obfuscar.xml](file:///e:/Dev/GC%20Center/native-monitor/obfuscar.xml)).
* **Execution Phase**: The obfuscation target runs automatically in the MSBuild pipeline right `AfterTargets="Compile"`. This guarantees that the compiled `GCMonitor.dll` is obfuscated *prior* to being packaged into the single-file self-contained host.
* **Protection Techniques**:
  * **Metadata Renaming**: Renames all classes, methods, parameters, properties, events, and fields to scrambled, non-descriptive characters.
  * **String Encryption**: Encrypts all string literals inside the assembly. String values are only decrypted at runtime in memory when referenced, preventing static string searches (e.g., searching for registry paths or commands).
  * **ILDASM Suppression**: Injects the `SuppressIldasmAttribute` into the assembly header to block the MSIL Disassembler tool from processing the binary.

---

## 6. Runtime Hardening & Hardened Electron Fuses

To protect the Electron runtime environment from being modified, SCESA uses `@electron/fuses` during the packaging phase (inside [after-pack.js](file:///e:/Dev/GC%20Center/scripts/after-pack.js)):

| Fuse Name | Value | Mitigated Threat Vector |
|---|---|---|
| **`RunAsNode`** | `false` | Prevents an attacker from launching `GS Center.exe` with the `ELECTRON_RUN_AS_NODE` environment variable to execute arbitrary JS files. |
| **`EnableNodeCliInspectArguments`** | `false` | Disables launching the app with `--inspect` or `--inspect-brk`, preventing debugging tools from connecting to the V8 engine. |
| **`EnableNodeOptionsEnvironmentVariable`** | `false` | Ignores the `NODE_OPTIONS` environment variable, preventing the injection of rogue preload scripts. |
| **`EnableEmbeddedAsarIntegrityValidation`**| `true` | Enforces cryptographic header validation on the ASAR archive. The OS will refuse to launch the executable if the ASAR hashes have been tampered with or repacked. |
| **`OnlyLoadAppFromAsar`** | `true` | Enforces that all code must be loaded strictly from the signed ASAR archive, preventing directory overrides. |
