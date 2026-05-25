---
name: kali-wsl-toolkit
description: 'Install, configure, and use Kali Linux via WSL2 from Windows terminals and VS Code agents. Covers WSL installation, relocating distros to secondary drives, running Kali tools through PowerShell, avoiding PowerShell/bash quoting conflicts, and using common pentest tools (nmap, nikto, sqlmap, gobuster, ffuf, theHarvester, wfuzz, hydra, sslscan). USE FOR: run Kali tools from VS Code, nmap scan, nikto scan, sqlmap injection test, gobuster directory brute, ffuf fuzzing, theHarvester OSINT, wfuzz parameter fuzzing, hydra brute force, sslscan TLS audit, install Kali WSL, move WSL to another drive, WSL quoting issues, run Linux tools from PowerShell. DO NOT USE FOR: Docker-based scanning (use nuclei-web-scanning or owasp-zap-web-scanning skills), Windows-native tools, unauthorized testing.'
argument-hint: 'Target URL or host and the Kali tool to run'
---

# Kali Linux via WSL2 — Agent Toolkit

> **Prerequisite:** WSL2 enabled on Windows 10/11, explicit authorization for any target scanned. All commands assume execution from a PowerShell terminal in VS Code.

## Step 1 — Install Kali Linux on WSL2

### Fresh Install

```powershell
wsl --install -d kali-linux
```

This downloads and registers the distro. You will be prompted to create a Linux username and password.

### Verify Installation

```powershell
wsl --list --verbose
wsl -d kali-linux -- bash -c "cat /etc/os-release | grep PRETTY"
```

Expected output: `PRETTY_NAME="Kali GNU/Linux Rolling"`

## Step 2 — Relocate WSL Distro to Another Drive (Optional)

If the primary drive lacks space, export/reimport the distro to a secondary drive.

```powershell
# Create target directory
mkdir D:\WSL

# Shut down all WSL instances
wsl --shutdown

# Export the distro (this can take several minutes for large installs)
wsl --export kali-linux D:\WSL\kali-linux.tar

# Unregister from original location (frees C:\ space)
wsl --unregister kali-linux

# Import to new location
wsl --import kali-linux D:\WSL\kali-linux D:\WSL\kali-linux.tar

# Clean up the tar
Remove-Item D:\WSL\kali-linux.tar
```

### Fix Default User After Import

After `--import`, WSL defaults to root. Restore the original user:

```powershell
# Find the non-root user (UID 1000)
wsl -d kali-linux -- grep ":1000:" /etc/passwd

# Set it as default (replace 'agent' with actual username)
wsl -d kali-linux -- bash -c "printf '[user]\ndefault=agent\n' > /etc/wsl.conf"
wsl --shutdown
```

Verify: `wsl -d kali-linux -- whoami` should return the username, not `root`.

## Step 3 — Install Kali Tooling

Choose the package set that matches the engagement:

```bash
# Full default toolkit (~5-7GB) — most common pentest tools
sudo apt update && sudo apt upgrade -y
sudo apt install -y kali-linux-default

# Lighter option — OWASP Top 10 tools only
sudo apt install -y kali-tools-top10

# Minimal — install individual tools as needed
sudo apt install -y nmap nikto sqlmap gobuster ffuf wfuzz hydra sslscan whatweb theharvester
```

To run these from PowerShell without entering a Kali shell:

```powershell
wsl -d kali-linux -- sudo apt update
wsl -d kali-linux -- sudo apt install -y nmap nikto sqlmap gobuster
```

## Step 4 — Running Kali Tools from PowerShell

### The Quoting Problem

PowerShell and bash have conflicting quote/pipe/escape rules. Commands piped through `wsl -d kali-linux -- bash -c "..."` frequently break because:

- PowerShell interprets `|`, `$`, `"`, and `>` before bash sees them
- Nested quotes get mangled across the two shells
- PowerShell's `&&` operator conflicts with bash's

### Solution: Write a Script File, Execute in WSL

The most reliable pattern for complex commands:

```powershell
# 1. Write the bash script from PowerShell using a here-string
$script = @"
#!/bin/bash
TARGET="https://example.com"
nmap -sV -sC -p 80,443 example.com
nikto -h `${TARGET} -output /tmp/nikto-results.txt
cat /tmp/nikto-results.txt
"@
$script | Set-Content -NoNewline -Path "$env:TEMP\scan.sh" -Encoding ascii

# 2. Execute in Kali via the Windows mount
wsl -d kali-linux -- bash /mnt/c/Users/$env:USERNAME/AppData/Local/Temp/scan.sh
```

> **Important:** Inside PowerShell here-strings (`@"..."@`), escape `$` as `` `$ `` to prevent PowerShell variable expansion before the script reaches bash.

### Simple Commands (No Pipes)

For single-tool commands without pipes or complex quoting, inline works:

```powershell
# These work fine directly
wsl -d kali-linux -- nmap -sV -p 80,443 example.com
wsl -d kali-linux -- whatweb https://example.com
wsl -d kali-linux -- sslscan example.com
wsl -d kali-linux -- curl -sI https://example.com
```

### Commands with Pipes (Use bash -c with Single Quotes)

```powershell
wsl -d kali-linux -- bash -c 'nmap -sV example.com | grep open'
wsl -d kali-linux -- bash -c 'curl -s https://example.com | grep -oE "src=[^ >]+"'
```

If single quotes conflict with the command content, fall back to the script file method.

## Step 5 — Common Tool Reference

### Nmap — Port Scanning and Service Detection

```powershell
# Quick service version scan on common web ports
wsl -d kali-linux -- nmap -sV -sC -p 80,443,8080,8443 <target>

# Full TCP scan (slower, comprehensive)
wsl -d kali-linux -- nmap -sV -sC -p- <target>

# Scan with output to file
wsl -d kali-linux -- nmap -sV -oN /tmp/nmap-results.txt <target>
```

### Nikto — Web Server Scanner

```powershell
wsl -d kali-linux -- nikto -h https://<target> -output /tmp/nikto.txt
```

### SQLMap — SQL Injection Testing

```powershell
# Test a GET parameter
wsl -d kali-linux -- sqlmap -u "https://<target>/page?id=1" --batch --level=3

# Test a POST form
wsl -d kali-linux -- sqlmap -u "https://<target>/login" --data="user=test&pass=test" --batch
```

### Gobuster — Directory Brute Force

```powershell
wsl -d kali-linux -- gobuster dir -u https://<target> -w /usr/share/wordlists/dirb/common.txt -t 10
```

### ffuf — Web Fuzzer

```powershell
wsl -d kali-linux -- ffuf -u "https://<target>/FUZZ" -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302,403
```

### theHarvester — OSINT Reconnaissance

```powershell
# Use free sources only (no API keys needed)
wsl -d kali-linux -- theHarvester -d <domain> -b crtsh,rapiddns,hackertarget,urlscan,otx -l 500
```

> **Note:** theHarvester searches public OSINT sources (DNS, cert transparency, search engines). It does NOT scan the web application itself. It is most useful for **custom domains**, not Azure/cloud-hosted subdomains like `*.azurewebsites.net` which have no independently indexed footprint.

### SSLScan — TLS Configuration Audit

```powershell
wsl -d kali-linux -- sslscan <target>:443
```

### WhatWeb — Technology Fingerprinting

```powershell
wsl -d kali-linux -- whatweb https://<target> -v
```

### Hydra — Credential Brute Force

```powershell
# HTTP POST form brute force
wsl -d kali-linux -- hydra -l admin -P /usr/share/wordlists/rockyou.txt <target> http-post-form "/login:user=^USER^&pass=^PASS^:Invalid"
```

### Wfuzz — Parameter and Path Fuzzing

```powershell
wsl -d kali-linux -- wfuzz -c -z file,/usr/share/wordlists/dirb/common.txt --hc 404 "https://<target>/FUZZ"
```

## Step 6 — Reading Results Back into Windows

Kali writes output to its own filesystem by default. To get results into the Windows workspace:

```powershell
# Option A: Write output directly to Windows mount
wsl -d kali-linux -- nmap -sV -oN /mnt/c/Users/$env:USERNAME/source/repos/project/nmap-results.txt <target>

# Option B: Copy after the fact
wsl -d kali-linux -- cp /tmp/results.txt /mnt/c/Users/$env:USERNAME/source/repos/project/
```

The Windows filesystem is mounted at `/mnt/c/` inside WSL. Writing directly to the workspace folder makes results immediately visible in VS Code.

## Step 7 — Wordlists

Kali includes wordlists at `/usr/share/wordlists/`. Common locations:

| Wordlist | Path | Use |
|----------|------|-----|
| dirb common | `/usr/share/wordlists/dirb/common.txt` | Directory brute force |
| SecLists | `/usr/share/seclists/` | Comprehensive (install: `sudo apt install seclists`) |
| rockyou | `/usr/share/wordlists/rockyou.txt` | Password cracking (decompress first: `sudo gzip -d /usr/share/wordlists/rockyou.txt.gz`) |

## Common Pitfalls

- **PowerShell eats your pipes** — `|` inside `wsl -- ...` gets interpreted by PowerShell, not bash. Always wrap piped commands in `bash -c '...'` or use the script file method.
- **`&&` doesn't chain in PowerShell** — Use `;` to chain PowerShell commands. Inside bash scripts, `&&` works normally.
- **Dollar sign expansion** — `$variable` in a here-string gets expanded by PowerShell. Escape as `` `$variable `` or use single-quoted here-strings (`@'...'@`) which don't expand.
- **head/tail/sort intercepted** — If a command after `|` is also a PowerShell alias or missing cmdlet, PowerShell may intercept it. Wrap the full pipeline in `bash -c`.
- **theHarvester needs API keys for most sources** — Free sources (crtsh, rapiddns, hackertarget, urlscan, otx) work without keys. VirusTotal, Shodan, etc. need keys configured in `/etc/theHarvester/api-keys.yaml`.
- **WSL defaults to root after import** — Always set the default user via `/etc/wsl.conf` after relocating a distro.
- **Rate limiting** — Always use rate controls (`-t`, `--rate`, `-rl`) when scanning live targets. Unbounded scanning can trigger WAF bans or cause denial of service.
- **Authorization** — Never scan targets without explicit written authorization. All tools in this skill are for authorized testing only.
