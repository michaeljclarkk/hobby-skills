---
name: zip-password-cracking
description: 'Crack password-protected ZIP archives using dictionary attacks, brute force, and known-plaintext attacks. Covers standard ZIP encryption and AES-encrypted ZIPs. USE FOR: crack ZIP password, dictionary attack ZIP, brute force archive, encrypted ZIP, password recovery, wordlist attack, known plaintext attack ZIP. DO NOT USE FOR: RAR/7z password cracking (different algorithms), creating password-protected archives, ZIP file repair.'
argument-hint: 'Path to the encrypted ZIP file and any known context about the password (e.g. "crack this ZIP, password is likely short and common")'
---

# ZIP Password Cracking

> **Prerequisite:** Python 3. Optionally `hashcat`, `john`, or `bkcrack` for advanced attacks.

## Step 1 — Inspect the ZIP

Determine the encryption method before choosing an attack:

```python
import zipfile

z = zipfile.ZipFile('<file>.zip')
for info in z.infolist():
    print(f"File: {info.filename}")
    print(f"  Compressed size: {info.compress_size}")
    print(f"  Original size: {info.file_size}")
    print(f"  Compression: {info.compress_type}")  # 0=stored, 8=deflate
    print(f"  Flag bits: {bin(info.flag_bits)}")
    # Bit 0 set = encrypted
    print(f"  Encrypted: {bool(info.flag_bits & 0x1)}")
```

**Encryption types:**
- **ZipCrypto** (traditional) — weak, vulnerable to known-plaintext attacks
- **AES-128/256** — stronger, requires brute force or dictionary

## Step 2 — Dictionary Attack (Python)

Start with a dictionary attack using common password lists. This is the fastest approach for weak passwords:

```python
import zipfile

z = zipfile.ZipFile('<file>.zip')
target = z.namelist()[0]

with open('wordlist.txt', 'r', encoding='utf-8', errors='ignore') as f:
    for line in f:
        pw = line.strip()
        if not pw:
            continue
        try:
            z.read(target, pwd=pw.encode())
            print(f"PASSWORD FOUND: {pw}")
            break
        except (RuntimeError, zipfile.BadZipFile):
            pass
    else:
        print("Password not in wordlist")
```

### Obtaining Wordlists

Common wordlists to try, in order of efficiency:

```powershell
# SecLists 10K most common passwords
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/Common-Credentials/10k-most-common.txt" -OutFile wordlist.txt

# RockYou (larger, 14M passwords — download if 10K fails)
# Available from SecLists or Kali Linux at /usr/share/wordlists/rockyou.txt
```

## Step 3 — Brute Force (Short Passwords)

If dictionary attacks fail and you suspect a short password:

```python
import zipfile, itertools, string

z = zipfile.ZipFile('<file>.zip')
target = z.namelist()[0]
charset = string.ascii_lowercase + string.digits

for length in range(1, 7):
    print(f"Trying length {length}...")
    for combo in itertools.product(charset, repeat=length):
        pw = ''.join(combo)
        try:
            z.read(target, pwd=pw.encode())
            print(f"PASSWORD FOUND: {pw}")
            raise SystemExit
        except (RuntimeError, zipfile.BadZipFile):
            pass
```

## Step 4 — Known-Plaintext Attack (ZipCrypto Only)

If you know or can guess at least 12 bytes of plaintext in any file within the ZIP, use `bkcrack`:

```powershell
# Download bkcrack
Invoke-WebRequest -Uri "https://github.com/kimci86/bkcrack/releases/latest/download/bkcrack-<version>-win64.zip" -OutFile bkcrack.zip
Expand-Archive bkcrack.zip -DestinationPath bkcrack

# Create a file containing the known plaintext bytes
# e.g. if you know a file starts with a PNG header or specific text

# Run the attack
.\bkcrack\bkcrack.exe -C <encrypted>.zip -c <filename_inside> -p <plaintext_file>
```

Once keys are recovered, extract without password:

```powershell
.\bkcrack\bkcrack.exe -C <encrypted>.zip -c <filename> -k <key1> <key2> <key3> -d <output_file>
```

## Step 5 — Hashcat / John the Ripper

For AES-encrypted ZIPs or when Python is too slow:

```powershell
# Extract hash for hashcat
zip2john <file>.zip > zip_hash.txt

# Run hashcat (mode 17200 = PKZIP, 13600 = WinZip AES)
hashcat -m 17200 zip_hash.txt wordlist.txt
```

## Common Pitfalls

- **Python's zipfile module** only handles ZipCrypto — for AES ZIPs, use `pyzipper` or external tools.
- **Encoding matters** — try both `pw.encode('utf-8')` and `pw.encode('latin-1')` for non-ASCII passwords.
- **CRC check shortcut** — Python's zipfile raises `BadZipFile` on wrong passwords before fully decompressing, making dictionary attacks reasonably fast.
- **Multiple files** — if the ZIP contains multiple files, cracking any one reveals the password for all.
