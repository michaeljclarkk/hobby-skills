---
name: owasp-zap-web-scanning
description: 'Run OWASP ZAP web application scans safely and document the results. Covers Docker-based ZAP baseline scans, full active scans, AJAX spidering, scan timeout controls, alert triage, artifact hygiene, and report writing. USE FOR: ZAP baseline scan, ZAP full scan, OWASP ZAP Docker, passive web scan, active web scan, scanner evidence, vulnerable JavaScript libraries, cookie/header findings, ASP.NET web scanning. DO NOT USE FOR: unauthorized testing, destructive fuzzing outside scope, credential theft, denial-of-service testing, exploitation beyond approved targets.'
argument-hint: 'Target URL and approved scope to scan with OWASP ZAP'
---

# OWASP ZAP Web Scanning

> **Prerequisite:** Docker, PowerShell or a POSIX shell, and explicit authorization for the target. Use this only inside the approved scope and stop if the scan begins causing service instability.

## Step 1 - Confirm Scope and Scan Mode

Before running ZAP, record:

| Item | Example |
|------|---------|
| Target URL | `https://example.test/?tenant=demo` |
| Allowed paths | Public customer store only, no manager/admin area |
| Scan type | Baseline/passive first; full active only after approval |
| Time limit | 5 minute spider, bounded active scan |
| Output names | `zap-baseline-target.*`, `zap-full-target.*` |

Use **baseline** for low-impact passive scanning. Use **full active** only when the user explicitly approves active testing, because ZAP will send attack-style payloads.

## Step 2 - Pull and Verify ZAP

```powershell
docker pull ghcr.io/zaproxy/zaproxy:stable
docker run --rm ghcr.io/zaproxy/zaproxy:stable zap.sh -version
```

Record the ZAP version in the report.

## Step 3 - Run a Baseline Scan

Baseline scanning spiders the target and passively inspects responses.

```powershell
$target = "<target-url>"
$prefix = "zap-baseline-target"

docker run --rm -v ${PWD}:/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap-baseline.py `
  -t $target `
  -m 5 `
  -r "$prefix.html" `
  -w "$prefix.md" `
  -J "$prefix.json" `
  -I
```

Important options:

| Option | Purpose |
|--------|---------|
| `-m 5` | Limits spider duration |
| `-r` | HTML report |
| `-w` | Markdown report |
| `-J` | JSON report |
| `-I` | Do not fail the command only because warnings exist |

## Step 4 - Run a Full Active Scan When Approved

Use this only after the user confirms active scanning is allowed.

```powershell
$target = "<target-url>"
$prefix = "zap-full-target"

docker run --rm -v ${PWD}:/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap-full-scan.py `
  -t $target `
  -m 5 `
  -j `
  -r "$prefix.html" `
  -w "$prefix.md" `
  -J "$prefix.json" `
  -I
```

The `-j` option enables the AJAX spider, which helps with client-heavy pages.

If the full scan hangs near the end on expensive rules, start ZAP as a daemon and set active scan duration limits through the API before scanning:

```powershell
docker run --rm -d --name zap-daemon -p 127.0.0.1:8090:8090 -v ${PWD}:/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable zap.sh `
  -daemon `
  -host 0.0.0.0 `
  -port 8090 `
  -config api.disablekey=true

Invoke-WebRequest "http://127.0.0.1:8090/JSON/ascan/action/setOptionMaxRuleDurationInMins/?Integer=10" -UseBasicParsing
Invoke-WebRequest "http://127.0.0.1:8090/JSON/ascan/action/setOptionMaxScanDurationInMins/?Integer=60" -UseBasicParsing
```

Clean up the daemon when finished:

```powershell
docker rm -f zap-daemon
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Status}}\t{{.Names}}"
```

## Step 5 - Triage the Results

Prioritize findings that are reachable in the application and supported by evidence.

High-value ZAP findings often include:

- Vulnerable JavaScript libraries with specific versions and CVEs.
- Missing Content-Security-Policy, anti-clickjacking, SRI, and `X-Content-Type-Options` headers.
- Cookie flags missing `Secure`, `HttpOnly`, or appropriate `SameSite`.
- Application error disclosure or parameter-triggered HTTP 500 responses.
- Cache-control findings where sensitive pages are storable.
- Server and framework disclosure such as IIS/ASP.NET headers.

Do not blindly accept scanner titles. For example, an `Integer Overflow Error` on an ASP.NET query parameter may really be an input-validation and exception-handling issue unless memory corruption is proven.

## Step 6 - Check Artifact Hygiene

Scanner reports can contain cookies, anti-forgery tokens, request headers, or session values. Before preserving or sharing reports, search for sensitive strings:

```powershell
Select-String -Path .\zap-*.html,.\zap-*.md,.\zap-*.json `
  -Pattern 'Cookie:|Set-Cookie|ARRAffinity|AspNetCore|__RequestVerificationToken|Authorization:'
```

If sensitive values appear, either sanitize the artifact or keep it private and document only redacted evidence.

## Step 7 - Write the Report Addendum

Use a concise addendum format:

```markdown
### ZAP Scan - <Target Name>

**Tool:** OWASP ZAP <version> via `ghcr.io/zaproxy/zaproxy:stable`  
**Command:** `<sanitized command>`  
**Artifacts:** `<html>`, `<md>`, `<json>`  
**Coverage:** <URL count>, AJAX spider on/off, active/passive mode  
**Result:** `FAIL-NEW: <n>`, `WARN-NEW: <n>`, `PASS: <n>`

| Risk | Alert Classes | Notable Results |
|------|--------------:|-----------------|
| High | <n> | <short list> |
| Medium | <n> | <short list> |
| Low | <n> | <short list> |
| Info | <n> | <short list> |

**Interpretation:** <what is new, what is corroboration, what needs validation>
```

## Common Pitfalls

- **Scanning beyond scope** - If the app redirects to manager/admin/third-party domains, stop or explicitly exclude them.
- **Duplicate apps** - If two URLs are the same app with different tabs or modes, one active scan may be enough after user confirmation.
- **Long active rules** - DOM XSS and server-side template injection rules can run for a long time; set bounded durations.
- **False precision** - Treat scanner labels as leads, not proof. Validate impact manually.
- **Cookie leakage in reports** - Always search reports before sharing or committing them.
- **Forgotten containers** - Check `docker ps` and remove leftover ZAP containers after cancelled scans.