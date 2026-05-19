---
name: js-bundle-secret-extraction
description: 'Extract hardcoded secrets, API keys, signing keys, and authentication logic from minified/bundled JavaScript files in production web applications. Covers Webpack, Vite, Rollup, and other bundlers. USE FOR: find API keys in JavaScript, extract signing key from JS bundle, reverse engineer API authentication, find hardcoded secrets in minified JS, analyze Webpack/Vite bundles, locate client-side crypto keys, extract HMAC/signature logic. DO NOT USE FOR: server-side source code review, binary reverse engineering, mobile app decompilation.'
argument-hint: 'URL of the web application or path to the JS bundle to analyze (e.g. "find the API signing key in this app")'
---

# JavaScript Bundle Secret Extraction

> **Prerequisite:** Browser dev tools or `curl`. Python 3 for scripting.

## Step 1 — Identify the Bundle Files

Locate the application's JavaScript bundles by inspecting the HTML source:

```powershell
# Fetch the main page and find script tags
$html = Invoke-WebRequest -Uri "<target_url>" -UseBasicParsing
$html.Content | Select-String -Pattern 'src="([^"]*\.js)"' -AllMatches |
    ForEach-Object { $_.Matches } | ForEach-Object { $_.Groups[1].Value }
```

Common bundle patterns:
- `/assets/index-<hash>.js` (Vite)
- `/static/js/main.<hash>.js` (Create React App / Webpack)
- `/bundle.js`, `/app.js` (generic)

## Step 2 — Download and Beautify

```powershell
# Download the bundle
Invoke-WebRequest -Uri "<target_url>/assets/index-<hash>.js" -OutFile bundle.js

# Beautify for readability (use an online tool or js-beautify)
pip install jsbeautifier
python -m jsbeautifier bundle.js > bundle-pretty.js
```

## Step 3 — Search for Secrets

Search the bundle for common secret patterns:

```powershell
# API keys, tokens, secrets
Select-String -Path bundle.js -Pattern '(api[_-]?key|secret|token|password|auth|sign|encrypt|hmac|md5|sha|hash)' -AllMatches

# Hardcoded strings that look like keys
Select-String -Path bundle.js -Pattern '"[A-Za-z0-9_$@!]{16,}"' -AllMatches

# Base64-encoded values
Select-String -Path bundle.js -Pattern '"[A-Za-z0-9+/]{20,}={0,2}"' -AllMatches

# Environment variables baked in at build time
Select-String -Path bundle.js -Pattern '(VITE_|REACT_APP_|NEXT_PUBLIC_|process\.env)' -AllMatches
```

## Step 4 — Trace the Authentication Flow

Find how API requests are signed or authenticated:

```powershell
# Look for fetch/axios interceptors and header construction
Select-String -Path bundle.js -Pattern '(X-Signature|Authorization|X-Api-Key|fetch\(|axios|headers)' -AllMatches

# Look for crypto operations
Select-String -Path bundle.js -Pattern '(MD5|SHA|CryptoJS|crypto|createHash|createHmac|digest|hmac)' -AllMatches
```

Once you find the signing function, trace backwards to find:
1. **The signing key** — usually a string constant nearby
2. **The signing algorithm** — MD5, HMAC-SHA256, etc.
3. **What gets signed** — typically the request body or specific headers

## Step 5 — Reconstruct the Signing Logic

Build a script that replicates the signing mechanism:

```python
import hashlib
import json
import requests

SIGNING_KEY = "<extracted_key>"

def compute_signature(body=None):
    """Replicate the client-side signing logic."""
    if body is None:
        payload = ""
    else:
        payload = json.dumps(body, separators=(',', ':'))  # compact JSON
    
    raw = SIGNING_KEY + payload
    return hashlib.md5(raw.encode()).hexdigest()

def api_request(method, url, body=None, token=None):
    headers = {"X-Signature": compute_signature(body)}
    if token:
        headers["Authorization"] = f"Bearer {token}"
    if method == "GET":
        return requests.get(url, headers=headers)
    else:
        return requests.post(url, json=body, headers=headers)
```

## Step 6 — Validate Access

Test the extracted credentials against the API:

```python
# Try unauthenticated endpoints first
resp = api_request("GET", "<target_url>/api/users")
print(resp.status_code, resp.json())

# Try authentication
body = {"username": "<user>", "password": "<pass>"}
resp = api_request("POST", "<target_url>/api/token/", body)
print(resp.json())
```

## What to Look For in Minified Code

| Pattern | Likely meaning |
|---------|---------------|
| `"X-Signature"` or custom header names | API signing/auth |
| Long string constants near crypto calls | Signing keys |
| `JSON.stringify` near hash functions | Request body signing |
| `localStorage.getItem("token")` | JWT storage |
| `atob()` / `btoa()` | Base64 encode/decode of secrets |
| `.env` variable names | Build-time injected secrets |

## Common Pitfalls

- **Minified variable names** — the key might be stored as a single letter like `const a="secret"`. Search by the string value, not the variable name.
- **String concatenation** — keys may be split across multiple variables or constructed at runtime.
- **Source maps** — check for `.js.map` files which give you the unminified source for free.
- **Multiple bundles** — secrets may be in vendor chunks or lazy-loaded modules, not just the main bundle.
- **JSON.stringify formatting** — when replicating signatures, match the exact serialisation (compact vs pretty-printed, key ordering).
