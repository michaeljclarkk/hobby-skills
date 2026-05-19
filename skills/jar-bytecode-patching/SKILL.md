---
name: jar-bytecode-patching
description: 'Decompile Java JAR files and patch hardcoded constants by modifying class file bytecode directly, then update the JAR. Uses CFR decompiler for analysis and binary patching for modifications. USE FOR: decompile JAR, patch JAR, modify hardcoded values, change timeout, edit class file, bytecode patching, raise limit, change constant, modify compiled Java, hex patch class. DO NOT USE FOR: writing new Java classes from scratch, modifying Java source projects with build systems, Maven/Gradle projects where source is available.'
argument-hint: 'Path to the JAR file and description of what to change (e.g. "raise the 240 second timeout to 15 minutes")'
---

# JAR Bytecode Patching

> **Prerequisite:** Requires a JDK installation with `jar`, `javap` tools. Optionally downloads the CFR decompiler for readable source output.

## Step 1 — Locate JDK Tools

Find the JDK installation. Common locations:
- `C:\java\jdk-*\bin\` (Windows)
- `/usr/lib/jvm/*/bin/` (Linux)
- Check `$env:JAVA_HOME` or `where.exe java`

Required tools: `jar.exe`, `javap.exe`, `java.exe`

Add the JDK bin to PATH for the session:
```powershell
$env:PATH = "C:\java\jdk-19.0.2\bin;$env:PATH"
```

## Step 2 — Inspect JAR Contents

List the classes inside the JAR to identify the relevant packages:

```powershell
jar tf <file>.jar | Where-Object { $_ -match '\.class$' } | ForEach-Object { ($_ -split '/')[0] } | Sort-Object -Unique
```

Then list the specific package classes you're interested in:
```powershell
jar tf <file>.jar | Where-Object { $_ -match '^<package>/' -and $_ -match '\.class$' }
```

## Step 3 — Decompile with CFR

Download CFR decompiler if not already available:
```powershell
Invoke-WebRequest -Uri "https://github.com/leibnitz27/cfr/releases/download/0.152/cfr-0.152.jar" -OutFile "cfr.jar"
```

Decompile the target classes into readable Java source:
```powershell
java -jar cfr.jar <file>.jar --outputdir decompiled --jarfilter "<package>.*"
```

Read the decompiled `.java` files to understand the code and locate the values to change.

## Step 4 — Search for Target Values

Search decompiled sources for the values you need to change:
```powershell
Get-ChildItem -Recurse -Filter "*.java" | Select-String -Pattern "timeout|240|Timeout" | Format-Table -AutoSize -Wrap
```

Also search for other numeric constants that may be related:
```powershell
Get-ChildItem -Recurse -Filter "*.java" | Select-String -Pattern "\d{4,}" | Where-Object { $_.Line -match '(timeout|wait|limit|max|1000|2000|5000|10000|30000|60000)' }
```

## Step 5 — Extract the Target Class

Extract the class file from the JAR for patching:
```powershell
jar xf <file>.jar <package>/<classname>.class
```

## Step 6 — Identify the Constant in Bytecode

Use `javap -verbose` to find the constant pool entry:
```powershell
javap -verbose <package>/<classname>.class | Select-String "<value>" -Context 1,1
```

This reveals the constant pool index (e.g. `#61 = Integer 240000`).

Also use `javap -c -p` to see the bytecode instruction that loads the constant:
```powershell
javap -c -p <package>/<classname>.class | Select-String "<value>"
```

## Step 7 — Binary Patch the Class File

Java class files store `Integer` constant pool entries as:
- **Tag byte:** `0x03` (CONSTANT_Integer)
- **4 bytes:** value in big-endian

Convert the old and new values to big-endian hex and search/replace in the binary:

```powershell
$classFile = "<package>\<classname>.class"
$bytes = [System.IO.File]::ReadAllBytes((Resolve-Path $classFile))

# Example: 240000 = 0x0003A980, replacement 900000 = 0x000DBBA0
# Search for tag byte (0x03) followed by the 4-byte value
$target = [byte[]]@(0x03, 0x00, 0x03, 0xA9, 0x80)
$replacement = [byte[]]@(0x03, 0x00, 0x0D, 0xBB, 0xA0)

$found = $false
for ($i = 0; $i -lt ($bytes.Length - 4); $i++) {
    if ($bytes[$i] -eq $target[0] -and $bytes[$i+1] -eq $target[1] -and
        $bytes[$i+2] -eq $target[2] -and $bytes[$i+3] -eq $target[3] -and
        $bytes[$i+4] -eq $target[4]) {
        Write-Host "Found at offset $i"
        for ($j = 0; $j -lt $replacement.Length; $j++) { $bytes[$i+$j] = $replacement[$j] }
        $found = $true
        break
    }
}

if ($found) {
    [System.IO.File]::WriteAllBytes((Resolve-Path $classFile), $bytes)
    Write-Host "Patched successfully"
} else {
    Write-Host "Pattern not found - try searching without tag byte"
}
```

**If the 5-byte pattern isn't found**, search for just the 4-byte value without the tag byte — the tag may not be adjacent depending on constant pool layout:
```powershell
for ($i = 0; $i -lt ($bytes.Length - 3); $i++) {
    if ($bytes[$i] -eq 0x00 -and $bytes[$i+1] -eq 0x03 -and
        $bytes[$i+2] -eq 0xA9 -and $bytes[$i+3] -eq 0x80) {
        Write-Host "Found value at offset $i (prev byte: 0x$([Convert]::ToString($bytes[$i-1],16)))"
    }
}
```

### Converting Values to Hex

Use PowerShell to convert decimal to big-endian hex bytes:
```powershell
$value = 900000
$hex = '{0:X8}' -f $value  # "000DBBA0"
# Split into bytes: 0x00, 0x0D, 0xBB, 0xA0
```

### Common Timeout Values Reference

| Milliseconds | Minutes | Hex (big-endian) |
|---|---|---|
| 240000 | 4 | `00 03 A9 80` |
| 600000 | 10 | `00 09 27 C0` |
| 900000 | 15 | `00 0D BB A0` |
| 1800000 | 30 | `00 1B 77 40` |
| 3600000 | 60 | `00 36 EE 80` |

## Step 8 — Verify the Patch

Confirm the patched value with `javap`:
```powershell
javap -verbose <package>/<classname>.class | Select-String "<new_value>"
```

## Step 9 — Update the JAR

**Always back up the original JAR first**, then inject the patched class:
```powershell
Copy-Item <file>.jar <file>.jar.bak
jar uf <file>.jar <package>/<classname>.class
```

Verify the update took effect by extracting and checking again:
```powershell
New-Item -ItemType Directory -Path verify_temp -Force | Out-Null
Push-Location verify_temp
jar xf ..\<file>.jar <package>/<classname>.class
javap -c -p <package>/<classname>.class | Select-String "<new_value>"
Pop-Location
Remove-Item -Recurse verify_temp
```

## Important Notes

- **Always back up** the original JAR before patching
- Only patch **constant pool entries** (integers, longs, strings) — do not attempt to change bytecode instructions or method signatures
- If patching a `Long` constant instead of `Integer`, the tag byte is `0x05` and the value is 8 bytes big-endian
- If patching a `String`, locate the `CONSTANT_Utf8` entry (tag `0x01`) — structure is: tag, 2-byte length, then UTF-8 bytes. String replacement only works if the new string is the **same length** as the old one
- The decompiled source in the `decompiled/` directory is for **reference only** — do not try to recompile it as it depends on proprietary runtime classes
- After patching, deploy the modified JAR to the target machine and restart the service
