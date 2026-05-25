---
name: nuclei-web-scanning
description: 'Run ProjectDiscovery Nuclei web scans safely and document the results. Covers Docker-based Nuclei execution, automatic scan mode, excluding intrusive/OAST/headless templates, rate/concurrency controls, JSONL output, artifact sanitization, informational finding triage, and HTML summary generation. USE FOR: nuclei scan, nuclei templates, ProjectDiscovery, safe web scanner, ASP.NET fingerprinting, HSTS detection, technology detection, Azure Blob detection, scanner JSONL report. DO NOT USE FOR: unauthorized testing, denial-of-service templates, brute force templates, intrusive fuzzing, credential theft, exploitation beyond approved scope.'
argument-hint: 'Target URL and approved scope to scan with Nuclei'
---

# Nuclei Web Scanning

> **Prerequisite:** Docker and explicit authorization for the target. Prefer a controlled scan profile for public-facing applications unless the user specifically approves broader testing.

## Step 1 - Confirm Scope and Safety Profile

Record the target and the exclusions before running Nuclei:

| Item | Example |
|------|---------|
| Target URL | `https://example.test/?tenant=demo` |
| Approved scope | Public customer store only |
| OAST/Interactsh | Disabled for controlled scans |
| Headless templates | Disabled unless needed and approved |
| Excluded tags | `dos`, `bruteforce`, `intrusive`, `fuzz` |
| Rate/concurrency | Low, bounded values |

Default Nuclei runs can be noisy. They may send malformed probes, OAST payloads, and broad template traffic. Start with a controlled run when testing someone else's live service.

## Step 2 - Pull and Verify Nuclei

```powershell
docker pull projectdiscovery/nuclei:latest
docker run --rm projectdiscovery/nuclei:latest -version
```

Record both the Nuclei version and template version when available.

## Step 3 - Run a Controlled Automatic Scan

This profile is useful for broad web reconnaissance while avoiding the most disruptive template classes.

```powershell
$target = "<target-url>"
$out = "nuclei-target-safe.jsonl"

docker run --rm -v ${PWD}:/work projectdiscovery/nuclei:latest `
  -u $target `
  -as `
  -ni `
  -ept headless `
  -etags dos,bruteforce,intrusive,fuzz `
  -rl 5 `
  -c 5 `
  -bs 1 `
  -pc 5 `
  -mhe 200 `
  -timeout 15 `
  -retries 1 `
  -jsonl `
  -o "/work/$out" `
  -or `
  -ot
```

Important options:

| Option | Purpose |
|--------|---------|
| `-as` | Automatic scan/template selection |
| `-ni` | Disable Interactsh/OAST callbacks |
| `-ept headless` | Exclude browser/headless protocol templates |
| `-etags ...` | Exclude noisy or potentially disruptive tags |
| `-rl 5` | Limit requests per second |
| `-c 5` | Limit template concurrency |
| `-bs 1` | Keep bulk size small |
| `-mhe 200` | Bound host errors |
| `-or` | Omit raw requests/responses |
| `-ot` | Omit template content from output |

## Step 4 - Handle Noisy or Timed-Out Runs

If a broad run is too noisy, times out, or starts sending malformed probes outside the intended risk appetite:

1. Stop the run.
2. Keep note of whether it found anything before stopping.
3. Switch to the controlled profile above.
4. Document that the broad run was stopped and why.

Do not treat a stopped broad run with zero matches as proof of no vulnerabilities. It only means that run did not finish with findings.

## Step 5 - Sanitize JSONL Artifacts

Nuclei output can include generated `curl-command` fields or request examples that contain live cookies and headers. Sanitize before sharing, committing, or converting to HTML.

```powershell
$inputPath = ".\nuclei-target-safe.jsonl"
$outputPath = Join-Path (Get-Location) "nuclei-target-safe.sanitized.jsonl"

$rows = Get-Content $inputPath | Where-Object { $_.Trim() } | ForEach-Object {
    $obj = $_ | ConvertFrom-Json
    if ($obj.PSObject.Properties.Name -contains "curl-command") {
        $obj.PSObject.Properties.Remove("curl-command")
    }
    $obj | ConvertTo-Json -Compress -Depth 50
}

[System.IO.File]::WriteAllLines(
    $outputPath,
    [string[]]$rows,
    [System.Text.UTF8Encoding]::new($false)
)
```

Validate the sanitized artifact:

```powershell
Get-Content .\nuclei-target-safe.sanitized.jsonl | ForEach-Object { $_ | ConvertFrom-Json | Out-Null }
Select-String -Path .\nuclei-target-safe.sanitized.jsonl -Pattern 'curl-command|ARRAffinity|AspNetCore|Cookie:|Authorization:' -Quiet
```

The final command should return `False`.

## Step 6 - Triage Findings

Group results by severity and template ID.

Common informational findings worth documenting:

| Template | Meaning |
|----------|---------|
| `azure-blob-core-detect` | Response references Azure Blob Storage |
| `favicon-detect` | Technology fingerprint from favicon hash |
| `microsoft-iis-version` | IIS version disclosed in response headers |
| `tech-detect` | Wappalyzer-style stack detection |
| `waf-detect` | Generic WAF/filter behavior |
| `weak-hsts-detect` | HSTS is present but max-age is weak |

Informational findings usually corroborate stack exposure or hardening gaps. They are not automatically exploitable. Promote severity only when the finding exposes sensitive data, proves reachable exploitation, or combines with another vulnerability.

## Common Pitfalls

- **Preserving unsafe exports** - Nuclei Markdown can include generated curl examples with cookies. Sanitize or delete it.
- **Using default mode blindly** - Default templates may be too noisy for a live public app.
- **OAST surprises** - Disable Interactsh with `-ni` unless callbacks are explicitly approved.
- **Headless load** - Browser templates can be slow and heavier than expected; exclude unless needed.
- **Info-only overstatement** - Technology detection is useful evidence, not an exploit by itself.
- **BOM in JSONL** - PowerShell may write UTF-8 with BOM; use `[System.Text.UTF8Encoding]::new($false)` for clean JSONL.