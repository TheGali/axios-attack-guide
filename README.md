# Axios npm Supply Chain Attack (March 30, 2026)

On March 30, 2026, an attacker published compromised versions of the `axios` npm package that install a cross-platform Remote Access Trojan (RAT) on your machine.

**Compromised versions:** `axios@1.14.1` and `axios@0.30.4`

These versions add a hidden dependency on `plain-crypto-js`, which runs a `postinstall` script that deploys the RAT silently during `npm install`.

## What the malware does

- Fingerprints your system (hostname, username, OS, running processes)
- Beacons to a command-and-control server every hour
- Accepts remote commands: shell execution, payload injection, filesystem enumeration, self-termination

## Am I affected?

Run these checks yourself. Read each command before running it.

### Step 1: Check your axios versions

In every project directory where you use axios:

```bash
npm list axios
```

If you see `1.14.1` or `0.30.4`, you are affected.

### Step 2: Search all lockfiles for the malicious dependency

**Mac/Linux:**
```bash
find ~ -maxdepth 5 \( -name "package-lock.json" -o -name "yarn.lock" -o -name "pnpm-lock.yaml" \) -exec grep -l "plain-crypto-js" {} \;
```

**Windows (PowerShell):**
```powershell
Get-ChildItem -Path $HOME -Recurse -Depth 5 -Include package-lock.json,yarn.lock,pnpm-lock.yaml -ErrorAction SilentlyContinue | Select-String -Pattern "plain-crypto-js" -List | Select-Object Path
```

Any matches mean that project pulled in the compromised package.

### Step 3: Check for RAT artifacts on disk

The malware drops a binary disguised as a system process:

| OS | File path | Disguised as |
|---|---|---|
| **macOS** | `/Library/Caches/com.apple.act.mond` | Apple daemon |
| **Windows** | `C:\ProgramData\wt.exe` | Windows Terminal |
| **Linux** | `/tmp/ld.py` | Temp file |

It also creates temporary launchers:

| OS | File path |
|---|---|
| **Windows** | `%TEMP%\6202033.vbs` |
| **Windows** | `%TEMP%\6202033.ps1` |

Check whether these files exist:

**Mac:**
```bash
ls -la /Library/Caches/com.apple.act.mond 2>/dev/null && echo "FOUND — you may be affected" || echo "Not found"
```

**Windows (PowerShell):**
```powershell
@("$env:ProgramData\wt.exe", "$env:TEMP\6202033.vbs", "$env:TEMP\6202033.ps1") | ForEach-Object {
    if (Test-Path $_) { Write-Host "FOUND: $_  — you may be affected" -ForegroundColor Red }
    else { Write-Host "Not found: $_" }
}
```

**Linux:**
```bash
ls -la /tmp/ld.py 2>/dev/null && echo "FOUND — you may be affected" || echo "Not found"
```

### Step 4: Check for active network connections to the C2 server

**Mac/Linux:**
```bash
netstat -an | grep "142.11.206.73"
```

**Windows (PowerShell):**
```powershell
Get-NetTCPConnection -RemoteAddress "142.11.206.73" -ErrorAction SilentlyContinue
```

Any results here mean the RAT is actively communicating.

## Indicators of Compromise (IOCs)

### Network

| Indicator | Value |
|---|---|
| C2 domain | `sfrclak[.]com` |
| C2 IP address | `142.11.206.73` |
| C2 port | `8000` |
| C2 path | `/6202033` |

### Compromised package hashes (SHA-1)

| Package | SHA-1 |
|---|---|
| `axios@1.14.1` | `2553649f2322049666871cea80a5d0d6adc700ca` |
| `axios@0.30.4` | `d6f3f62fd3b9f5432f5782b62d8cfd5247d5ee71` |
| `plain-crypto-js@4.2.1` | `07d889e2dadce6f3910dcbc253317d28ca61c766` |

### Related malicious packages

- `plain-crypto-js@4.2.0` and `4.2.1`
- `@shadanai/openclaw` (versions 2026.3.28-2, 2026.3.28-3, 2026.3.31-1, 2026.3.31-2)
- `@qqbrowser/openclaw-qbot@0.0.130`

### Threat actor identifiers

| Indicator | Value |
|---|---|
| Email | `ifstap@proton.me` |
| Email | `nrwise@proton.me` |
| XOR obfuscation key | `OrDeR_7077` |

## What to do if you're affected

1. **Disconnect from the internet** to stop C2 communication.
2. **Kill the RAT process** and delete the binary from the paths listed above.
3. **Remove the compromised packages:** delete `node_modules` and `package-lock.json` in affected projects, then reinstall with a clean axios version (`npm install axios@1.7.9` or whatever your last known-good version was).
4. **Search your lockfiles** for `plain-crypto-js`, `@shadanai/openclaw`, and `@qqbrowser/openclaw-qbot`. Remove any references.
5. **Rotate all credentials** on the affected machine: SSH keys, API tokens, environment variables, browser-saved passwords. The RAT had shell access. Assume anything on that machine was exfiltrable.
6. **Check git history** for any commits you didn't make.
7. **Monitor** the C2 IP (`142.11.206.73`) in your network logs going forward.

## What happened

The attacker gained publish access to the `axios` npm package and released two versions (`1.14.1` and `0.30.4`) with `plain-crypto-js` injected as a dependency. That package contains a `postinstall` script that executes automatically during `npm install`, deploying the RAT without any user interaction beyond the install itself.

The compromised versions have been pulled from npm. If you installed axios after March 30, 2026, and before the takedown, check your system.

## About this repo

This is a rewrite of the original attack guide for clarity and safety. No executable scripts are included. Every detection command is listed in plain text so you can read it before running it.
