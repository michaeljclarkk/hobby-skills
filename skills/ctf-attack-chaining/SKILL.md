---
name: ctf-attack-chaining
description: 'Systematic methodology for approaching web application CTF challenges by chaining multiple attack techniques in the correct order. Covers reconnaissance, attack surface mapping, technique selection, and feeding outputs between attack phases. USE FOR: CTF challenge, web app penetration testing workflow, attack chain planning, what to try next, multi-stage exploitation, connecting findings across techniques, deciding attack order, pivoting between techniques. DO NOT USE FOR: single-technique attacks where the method is already known, non-web challenges (pwn, reversing, crypto-only), writing exploits from scratch.'
argument-hint: 'Target URL or challenge description (e.g. "work through this CTF challenge starting from this URL")'
---

# CTF Attack Chaining

> **Prerequisite:** Familiarity with the individual technique skills. This skill is the decision framework that connects them.

## Phase 1 — Initial Reconnaissance

Start with the entry point. Determine what you're dealing with before picking any technique.

### If the entry point is a file:

```python
# Identify the file type
import magic  # or check manually
data = open('<file>', 'rb').read()
print(f"Size: {len(data)} bytes")
print(f"First 20 bytes (hex): {data[:20].hex()}")
print(f"First 100 chars: {data[:100]}")
```

**Decision tree for files:**

| Observation | Next step |
|-------------|-----------|
| Text with irregular capitalisation | → `text-steganography-decoding` skill |
| Text with invisible/zero-width characters | → `text-steganography-decoding` skill |
| Binary file, starts with `PK` (50 4B) | ZIP archive → try extracting, if encrypted → `zip-password-cracking` skill |
| Binary file, other magic bytes | Identify format (PNG, PDF, ELF, etc.) and apply relevant analysis |
| Base64-encoded string | Decode it, then re-assess the output |
| Hex-encoded string | Decode it, then re-assess the output |
| URL or domain | Move to web reconnaissance (Phase 2) |

### If the entry point is a URL:

Move directly to Phase 2.

## Phase 2 — Web Application Reconnaissance

Map the application's attack surface before touching any endpoint.

### 2a. Technology fingerprinting

```powershell
# Fetch the main page
$resp = Invoke-WebRequest -Uri "<target>" -UseBasicParsing

# Identify the framework
$resp.Headers['Server']          # nginx, Apache, gunicorn, etc.
$resp.Headers['X-Powered-By']    # Express, PHP, ASP.NET, etc.

# Check for SPA frameworks
$resp.Content | Select-String -Pattern '(React|Vue|Angular|Svelte|vite|webpack|next)' -AllMatches
```

### 2b. JavaScript bundle analysis

This is almost always worth doing on SPAs. The client-side code is the attacker's gift — it contains route definitions, API endpoints, auth logic, and often secrets.

```powershell
# Find and download JS bundles
$resp.Content | Select-String -Pattern 'src="([^"]*\.js)"' -AllMatches |
    ForEach-Object { $_.Matches } | ForEach-Object { $_.Groups[1].Value }
```

**→ Use `js-bundle-secret-extraction` skill to analyse the bundles.**

Extract from the bundle:
1. **All API endpoints** — gives you the full attack surface
2. **Authentication mechanism** — how requests are signed/authenticated
3. **Signing keys or secrets** — hardcoded in client = game over
4. **Role/permission checks** — reveals the privilege hierarchy
5. **Hidden features** — admin panels, debug endpoints, file operations

### 2c. API endpoint discovery

```python
# Probe standard endpoints
import requests

base = "<target>"
common = [
    "/api/", "/api/users/", "/api/profile/",
    "/api/token/", "/api/login/", "/api/auth/",
    "/api/backup/", "/api/config/", "/api/admin/",
    "/api/docs/", "/api/swagger/", "/api/schema/",
    "/robots.txt", "/sitemap.xml", "/.env",
]

for ep in common:
    r = requests.get(f"{base}{ep}", timeout=5)
    if r.status_code != 404:
        print(f"{r.status_code} {ep:35s} {r.headers.get('content-type','')}")
```

## Phase 3 — The Attack Chain

This is the critical decision framework. Each phase feeds into the next.

```
┌─────────────────────────────────────────────────────────┐
│                    ATTACK CHAIN                         │
│                                                         │
│  1. RECON          → endpoints, tech stack, JS secrets  │
│        ↓                                                │
│  2. UNAUTH ACCESS  → user lists, public data, info leak │
│        ↓                                                │
│  3. GET CREDS      → crack hashes, default passwords    │
│        ↓                                                │
│  4. AUTH ACCESS     → login, explore authenticated features│
│        ↓                                                │
│  5. IDOR           → access other users' private data   │
│        ↓                                                │
│  6. PRIV ESC       → use leaked creds to reach admin    │
│        ↓                                                │
│  7. FULL ACCESS    → database, files, system control    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Step-by-step decisions:

**After recon, ask:** "Can I access any API endpoints without authentication?"
- Yes → **`sensitive-data-exposure` skill** — enumerate what's exposed. Look for user lists, password hashes, credential fields.
- No → Try default credentials, check for registration endpoints.

**After finding user data, ask:** "Do I have password hashes or any way to get credentials?"
- Hashes found → Crack them against a wordlist (start with 10K common, escalate to rockyou).
- Cleartext passwords found → Skip cracking, go straight to login.
- No passwords → Look harder at the JS bundle for hardcoded creds, or try default passwords.

**After getting a login, ask:** "What new endpoints or features are available now?"
- New profile/account endpoints → **`idor-exploitation` skill** — test if you can access other users' data.
- Same features as unauthenticated → The auth is decorative; focus on IDOR and endpoint probing.

**After finding IDOR, ask:** "Does the leaked data include credentials for higher-privilege users?"
- Yes → **`broken-access-control` skill** — authenticate as admin/superadmin.
- No → Check if admin endpoints are accessible without admin role (missing function-level authorisation).

**After privilege escalation, ask:** "What admin-only features exist?"
- Database backup/export → Download and extract credentials, flags, secrets.
- File read/locate → Explore the server filesystem.
- User management → Check for additional hidden users or roles.

## Phase 4 — Output Chaining Reference

How specific outputs feed into subsequent attacks:

| Source | Data obtained | Feeds into |
|--------|--------------|------------|
| JS bundle | API endpoints, signing key | All subsequent API calls |
| `/api/users/` | Usernames, roles, password hashes | Hash cracking → login |
| Hash cracking | Cleartext password for one user | First authenticated session |
| `/api/profile/?user=X` (IDOR) | Other users' cleartext passwords | Login as higher-privilege user |
| Admin dashboard | Database backup endpoint | Download full database |
| Database backup | All credentials, roles, data | Login as highest-privilege user |

## Phase 5 — When You're Stuck

If a technique isn't working, don't brute-force it. Pivot.

**Password won't crack?**
- Try a bigger wordlist before brute force.
- Check if cleartext passwords are exposed elsewhere (IDOR, database backup, config files).
- Look for password reset functionality.

**API returns 403?**
- Check if you need a signing header (inspect JS bundle).
- Try different HTTP methods (GET vs POST vs PUT).
- Try with and without trailing slashes.
- Check if authentication is actually required or just the signature.

**Can't find the next flag?**
- Re-read API responses carefully — flags may be in description fields, alerts, error messages, or response headers.
- Check the UI after login — dashboard alerts, profile sections, hidden elements.
- Try every user role — different roles see different content.

**Stego output is garbage?**
- Flip the bit convention (uppercase=0 instead of 1).
- Check if there's an intermediate encoding layer (base64, hex).
- Verify you're filtering non-alpha characters correctly.

## Technique Selection Cheatsheet

| Signal | Technique |
|--------|-----------|
| Text with weird capitalisation | Case-based steganography |
| Encrypted ZIP/archive | Dictionary attack → brute force → known-plaintext |
| SPA with API backend | JS bundle analysis for secrets |
| User list with IDs | IDOR on profile/account endpoints |
| Password hashes in API | Wordlist cracking (MD5/SHA) |
| Multiple user roles visible | Privilege escalation via credential theft |
| Admin panel accessible | Database backup, file access, config exposure |
| 403 on API with no auth header | Signing key needed — check JS bundle |

## Common Pitfalls

- **Skipping JS analysis** — the client-side bundle is the single highest-value target in most web CTFs. Always read it.
- **Trying complex attacks first** — start with the easiest: unauthenticated endpoint enumeration, default passwords, common wordlists.
- **Ignoring response bodies** — flags and clues hide in description fields, alert messages, error text, and HTTP headers, not just obvious "flag" fields.
- **Single-user tunnel vision** — always enumerate all users and try each role. Different roles unlock different flags and features.
- **Not recording credentials** — maintain a running list of every username, password, role, and token you discover. You'll need them for later stages.
