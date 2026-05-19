---
name: dotnet-ilspy-decompilation
description: 'Decompile .NET DLL/EXE assemblies into readable C# source using ILSpy, then reconstruct a compilable project for analysis, API mapping, and integration work. USE FOR: decompile DLL, decompile EXE, reverse engineer .NET assembly, inspect API surface, extract embedded resources, create API reference, understand internal protocol, analyze third-party DLL, build integration wrapper. DO NOT USE FOR: modifying IL bytecode directly (use dnSpy or ildasm/ilasm), decompiling Java JARs (use jar-bytecode-patching skill), assemblies that require runtime debugging (use dnSpy debugger).'
argument-hint: 'Path to the .NET DLL or EXE and what you want to learn from it (e.g. "decompile and map the public API surface")'
---

# .NET Assembly Decompilation with ILSpy

> **Prerequisite:** ILSpy CLI (`ilspycmd`) or ILSpy GUI installed. .NET SDK for project reconstruction.

## Step 1 — Install ILSpy CLI

Install the ILSpy command-line tool as a .NET global tool:

```powershell
dotnet tool install -g ilspycmd
```

Verify installation:
```powershell
ilspycmd --version
```

## Step 2 — Initial Reconnaissance

Before decompiling, gather metadata about the assembly to understand what you're dealing with:

```powershell
ilspycmd --list <assembly>.dll
```

This lists all types in the assembly. Pipe to a file for large assemblies:
```powershell
ilspycmd --list <assembly>.dll | Out-File assembly-types.txt
```

Check for obfuscation indicators:
```powershell
ilspycmd --list <assembly>.dll | Where-Object { $_ -match '[^a-zA-Z0-9_.\s<>`,\[\]+\-]' }
```

If type/method names are gibberish (e.g. `a0b`, `\u0001`, `ᐁ`), the assembly is **obfuscated** — decompiled output will be harder to read but still structurally useful. If names are meaningful (e.g. `mySession`, `myTransaction`), you're in luck.

Also check the assembly metadata:
```powershell
ilspycmd -t metadata <assembly>.dll
```

## Step 3 — Decompile to a Full Project

ILSpy can generate a complete, compilable C# project. This is the most useful output format for analysis:

```powershell
# Create output directory
New-Item -ItemType Directory -Path decompiled -Force

# Decompile to a project
ilspycmd -p -o decompiled <assembly>.dll
```

The `-p` flag generates a `.csproj` file with proper references and settings. The `-o` flag sets the output directory.

### Key flags reference:

| Flag | Purpose |
|---|---|
| `-p` | Generate project file (.csproj) |
| `-o <dir>` | Output directory |
| `-t <type>` | Decompile only a specific type |
| `-l CSharp` | Force C# output (default) |
| `--nested-directories` | Mirror namespace structure as folders |
| `-lv CSharp10_0` | Target C# language version |

For large assemblies, decompile with nested directories to keep things organized:
```powershell
ilspycmd -p -o decompiled --nested-directories <assembly>.dll
```

## Step 4 — Examine the Output Structure

After decompilation, inspect the generated project:

```powershell
# Check what was generated
Get-ChildItem -Recurse decompiled -Include *.cs, *.csproj, *.resx | Select-Object FullName
```

Typical output structure from a well-organized assembly:
```
decompiled/
    <AssemblyName>.csproj
    Properties/
        AssemblyInfo.cs
    <Namespace>/
        <SubNamespace>/
            <Type>.cs
    <EmbeddedResources>.resx
```

Check the generated `.csproj` for target framework and dependencies:
```powershell
Get-Content decompiled\*.csproj
```

ILSpy auto-detects the original target framework (e.g. `net20`, `net45`, `net6.0`) and sets it in the project file. It also generates `<Reference>` entries for framework assemblies and `<EmbeddedResource>` entries for embedded resources.

## Step 5 — Identify Embedded Resources

.NET assemblies often embed files (executables, config, images). ILSpy extracts these alongside the code:

```powershell
# List embedded resources extracted by ILSpy
Get-ChildItem decompiled -Recurse -Exclude *.cs, *.csproj | Where-Object { !$_.PSIsContainer }
```

The `.csproj` will have `<EmbeddedResource>` entries with `LogicalName` attributes mapping to the original resource names:
```powershell
Select-String -Path decompiled\*.csproj -Pattern "EmbeddedResource"
```

For assemblies that embed other executables (common in adapter/launcher patterns), the resource will appear as a binary file that you can inspect:
```powershell
# Check if any embedded resources are PE executables
Get-ChildItem decompiled -Recurse -Exclude *.cs, *.csproj, *.resx | ForEach-Object {
    $bytes = [System.IO.File]::ReadAllBytes($_.FullName)
    if ($bytes.Length -ge 2 -and $bytes[0] -eq 0x4D -and $bytes[1] -eq 0x5A) {
        Write-Host "PE executable: $($_.FullName)"
    }
}
```

## Step 6 — Create a Solution Wrapper

Wrap the decompiled project in a Visual Studio solution for easy browsing:

```powershell
# From the parent directory of decompiled/
dotnet new sln -n <AssemblyName>
dotnet sln <AssemblyName>.sln add decompiled\<AssemblyName>.csproj
```

Open the solution in VS Code or Visual Studio for full IntelliSense navigation across the decompiled source.

## Step 7 — Attempt a Test Build

Try building the decompiled project. This reveals missing dependencies and validates decompilation quality:

```powershell
dotnet build decompiled\<AssemblyName>.csproj 2>&1 | Tee-Object build-output.txt
```

Common issues and fixes:

### Missing Framework Assemblies
If the assembly targets an old framework (e.g. .NET 2.0), you may need to adjust the target:
```xml
<!-- In .csproj, try changing -->
<TargetFramework>net20</TargetFramework>
<!-- To a supported version -->
<TargetFramework>net48</TargetFramework>
```

### Unsafe Code
If the build fails with unsafe code errors, ensure the project allows it:
```xml
<AllowUnsafeBlocks>True</AllowUnsafeBlocks>
```

### Missing NuGet Packages
If the assembly referenced NuGet packages, add them:
```powershell
dotnet add decompiled\<AssemblyName>.csproj package <PackageName>
```

### PrivateImplementationDetails
ILSpy may emit compiler-generated types (e.g. `<PrivateImplementationDetails>`) with data initializers that show as `/* Not supported: data(...) */`. These are static array initializations — they won't compile but are cosmetic. You can safely ignore them or manually reconstruct the arrays from the hex data.

### Language Version Conflicts
If modern C# syntax doesn't match the target framework:
```xml
<LangVersion>12.0</LangVersion>
```

A successful or near-successful build confirms the decompilation is high quality.


## Important Notes

- **ILSpy preserves original names** when the assembly is not obfuscated — fields, methods, types, and even local variables often retain their original names
- **Decompiled code is for analysis only** — even if it compiles, it may not behave identically to the original due to compiler optimizations, inlined constants, or runtime-generated code
- **Do not redistribute** decompiled source or derivative works without confirming license terms. Decompilation for interoperability/analysis is generally permitted, but redistribution may not be
- **ILSpy version matters** — newer versions handle more C# features (pattern matching, records, async/await). Use the latest stable release for best results
- **For obfuscated assemblies**, consider `de4dot` (https://github.com/de4dot/de4dot) as a pre-processing step before ILSpy to restore names and remove anti-decompilation patterns
