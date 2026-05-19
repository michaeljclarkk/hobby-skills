---
name: text-steganography-decoding
description: 'Detect and decode hidden messages concealed in text using steganographic techniques such as case-based binary encoding, whitespace encoding, zero-width characters, and other text-level covert channels. USE FOR: decode hidden message in text, case steganography, binary from uppercase lowercase, whitespace steganography, zero-width character extraction, invisible text decoding, text-based stego analysis, suspicious formatting in documents. DO NOT USE FOR: image steganography (use stegsolve/zsteg), audio steganography, network steganography, binary file analysis.'
argument-hint: 'Path to the text file or URL containing suspected steganographic content (e.g. "analyze this text file for hidden messages")'
---

# Text Steganography Decoding

> **Prerequisite:** Python 3 with `base64` and `string` standard libraries.

## Step 1 — Initial Analysis

Examine the text for anomalies that suggest hidden data. Common indicators:

- **Irregular capitalisation** in otherwise normal prose (case-based encoding)
- **Trailing whitespace** or inconsistent spacing (whitespace stego)
- **File size** disproportionate to visible content
- **Unicode oddities** — zero-width joiners, non-breaking spaces, homoglyphs

```powershell
# Check for mixed case patterns that seem unnatural
$text = Get-Content <file> -Raw
# Count uppercase vs lowercase ratio
$upper = ([regex]::Matches($text, '[A-Z]')).Count
$lower = ([regex]::Matches($text, '[a-z]')).Count
Write-Host "Upper: $upper, Lower: $lower, Ratio: $([math]::Round($upper/$lower, 3))"
```

If the ratio is close to 0.5 or the capitalisation looks random in well-known text (e.g. a famous manifesto, speech, or passage), it's likely **case-based binary encoding**.

```powershell
# Check for zero-width characters
python -c "
data = open('<file>', 'r', encoding='utf-8').read()
zw = [c for c in data if ord(c) in (0x200B, 0x200C, 0x200D, 0xFEFF, 0x00AD)]
print(f'Zero-width characters found: {len(zw)}')
"
```

## Step 2 — Identify the Source Text

If the text appears to be a well-known document with altered capitalisation, identify it. Recognising the source confirms the stego technique — the original casing is irrelevant; only the attacker's modifications carry data.

Search for distinctive phrases from the text online to identify the original document.

## Step 3 — Extract the Binary Stream

### Case-Based Binary Encoding

The most common text stego: each letter's case represents a bit. Extract only alphabetic characters and map their case to binary:

```python
import string

text = open('<file>', 'r').read()

bits = []
for ch in text:
    if ch in string.ascii_uppercase:
        bits.append('1')
    elif ch in string.ascii_lowercase:
        bits.append('0')
    # Non-alpha characters are ignored

bitstring = ''.join(bits)
print(f"Extracted {len(bitstring)} bits ({len(bitstring)//8} bytes)")
print(f"First 64 bits: {bitstring[:64]}")
```

### Whitespace Encoding

Spaces and tabs as binary:

```python
text = open('<file>', 'r').read()
bits = []
for ch in text:
    if ch == ' ':
        bits.append('0')
    elif ch == '\t':
        bits.append('1')
```

### Zero-Width Character Encoding

```python
text = open('<file>', 'r', encoding='utf-8').read()
bits = []
for ch in text:
    if ch == '\u200b':  # zero-width space
        bits.append('0')
    elif ch == '\u200c':  # zero-width non-joiner
        bits.append('1')
```

## Step 4 — Convert Bits to Bytes

```python
raw_bytes = bytes(int(bitstring[i:i+8], 2) for i in range(0, len(bitstring) - len(bitstring) % 8, 8))
```

## Step 5 — Identify the Payload Format

Check what the extracted bytes actually are:

```python
# Check for magic bytes
print(f"First 4 bytes (hex): {raw_bytes[:4].hex()}")
print(f"First 20 bytes (repr): {raw_bytes[:20]}")

# Common signatures:
# PK\x03\x04 = ZIP archive
# \x89PNG = PNG image
# Base64 text = re-encode step
```

If the raw bytes look like base64 text (only `A-Za-z0-9+/=` characters):

```python
import base64
decoded = base64.b64decode(raw_bytes)
print(f"Decoded {len(decoded)} bytes")
print(f"Magic bytes: {decoded[:4].hex()}")

with open('extracted_payload', 'wb') as f:
    f.write(decoded)
```

## Step 6 — Handle the Payload

Based on the detected file type, extract or process accordingly:

```python
# If ZIP:
import zipfile, io
z = zipfile.ZipFile(io.BytesIO(decoded))
print("Files in archive:", z.namelist())

# If the ZIP is password-protected, move to the zip-password-cracking skill
```

## Common Pitfalls

- **Bit order ambiguity:** Try both uppercase=1 and uppercase=0 if the first attempt produces garbage.
- **Padding:** The bit count may not be divisible by 8 — truncate trailing bits.
- **Multi-layer encoding:** The extracted binary may itself be base64, hex, or another encoding before reaching the final payload.
- **Non-alpha noise:** Always filter out non-alphabetic characters before extracting bits.
