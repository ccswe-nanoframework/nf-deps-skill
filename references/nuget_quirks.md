# nuget_quirks

Canonical write-up of the `nuget update` pitfalls this skill encodes. Subcommands link in here rather than restating the pitfalls. Read the relevant section when you hit the corresponding step in a subcommand; do not paraphrase these into subcommand bodies.

Every pitfall below has been verified in a real session against the CCSWE.nanoFramework and Emily.Clock repos.

---

## runsettings prompt

**Symptom:** `nuget update <sln>` runs cleanly through the first several test projects, then stops mid-output and never returns. Backgrounded, it eats CPU briefly then sits forever. The output file ends with a line like:

```
File 'nano.runsettings' already exists in project 'X.UnitTests'. Do you want to overwrite it?
```

**Cause:** `nanoFramework.TestFramework` ships `nano.runsettings` as a content file. When `nuget update` reinstalls the package, the install step finds the existing `nano.runsettings` in the project and prompts. Claude's Bash environment cannot answer interactive prompts. The process hangs.

**Fix:** Always pass `-FileConflictAction Ignore`:

```
nuget update "$sln" -FileConflictAction Ignore
```

`Ignore` causes the install step to skip the existing file, leaving the user's customizations intact. This is correct for nanoFramework test projects because the project's `nano.runsettings` should already be tracked in git — there is no value in overwriting it with the package's default version. (`Overwrite` would clobber project-specific settings; `OverwriteAll` is identical for this case but verbose.)

**Recovery if you forgot the flag:** Kill the `nuget.exe` process (`Stop-Process -Name nuget -Force`), re-run with the flag, then run [Step 7 of update.md](../subcommands/update.md) to restore any `nano.runsettings` that got partially deleted before the hang.

---

## Private noise + BOM preservation

### Symptom 1 — Private noise

After `nuget update`, every modified `<Reference>` in every touched `.nfproj` gains a `<Private>True</Private>` child element:

```xml
<Reference Include="System.Net, Version=1.11.52.0, ...">
  <HintPath>..\..\packages\nanoFramework.System.Net.1.11.52\lib\System.Net.dll</HintPath>
  <Private>True</Private>   <!-- new, unwanted -->
</Reference>
```

These do not appear elsewhere in the CCSWE.nanoFramework codebase — they are a `nuget update` artifact. They must be stripped before commit or the diff explodes with unrelated lines.

### Semantic check — strip new entries only, not pre-existing ones

`<Private>True|False</Private>` is a valid MSBuild metadata element (it controls the `CopyLocal` behavior of a reference). A project author may have set it intentionally. **Do not unconditionally strip every `<Private>` line.** The target is the noise `nuget update` just added — i.e. lines that are *new vs HEAD*, not lines already present in the committed `.nfproj`.

Algorithm:

1. For each modified `.nfproj`, list `<Private>` lines in the working tree and in `git show HEAD:<path>`.
2. If HEAD has zero and the working tree has N: all N are new noise — strip them all.
3. If HEAD has K and the working tree has K+M: only the M new lines are noise — strip those, preserve the K originals.
4. If not in a git repo (no HEAD baseline): list every `<Private>` line and ask the user before stripping anything.

The simplest practical check for case (2) — by far the most common — is to confirm `git show HEAD:<path> | grep -c '<Private>'` returns `0`, then proceed to strip all working-tree `<Private>True</Private>` lines.

### Symptom 2 — BOM corruption

The obvious strip-by-regex implementation:

```powershell
# WRONG — strips BOMs
$text = [System.Text.Encoding]::UTF8.GetString([System.IO.File]::ReadAllBytes($p))
$new  = $text -replace '<Private>True</Private>\r?\n',''
[System.IO.File]::WriteAllBytes($p, [System.Text.Encoding]::UTF8.GetBytes($new))
```

silently strips UTF-8 BOMs from files that had one in HEAD (the bytes survive into `$text` as a U+FEFF character, and `UTF8.GetBytes` does not re-emit a preamble). Result: every touched file's diff includes a `<?xml version="1.0" encoding="utf-8"?>` line change.

**Whether a given `.nfproj` has a BOM is per-file, not per-repo** — both states exist in the same nanoFramework projects. The fix must read the BOM state, preserve it in the round-trip, and use an **anchored regex** (`(?m)^[ \t]*<Private>True</Private>[ \t]*\r?\n`) so the deletion consumes the whole line without eating whitespace from adjacent lines.

### Reference implementations

Use whichever fits the current environment. Both have been verified against real CCSWE.nanoFramework `.nfproj` files.

#### PowerShell (Windows + cross-platform via `pwsh`)

```powershell
param([Parameter(ValueFromRemainingArguments = $true)][string[]]$Paths)
$bom = [byte[]](0xEF, 0xBB, 0xBF)
foreach ($p in $Paths) {
    if (-not (Test-Path -LiteralPath $p)) { Write-Warning "skip: $p"; continue }
    $bytes  = [System.IO.File]::ReadAllBytes($p)
    $hadBom = $bytes.Length -ge 3 -and $bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF
    $start  = if ($hadBom) { 3 } else { 0 }
    $text   = [System.Text.Encoding]::UTF8.GetString($bytes, $start, $bytes.Length - $start)
    $new    = [regex]::Replace($text, '(?m)^[ \t]*<Private>True</Private>[ \t]*\r?\n', '')
    if ($new -ne $text) {
        $out = [System.Text.Encoding]::UTF8.GetBytes($new)
        if ($hadBom) {
            $combined = New-Object byte[] ($out.Length + 3)
            [Array]::Copy($bom, 0, $combined, 0, 3)
            [Array]::Copy($out, 0, $combined, 3, $out.Length)
            $out = $combined
        }
        [System.IO.File]::WriteAllBytes($p, $out)
        Write-Host "stripped: $p"
    }
}
```

Run as `pwsh -NoProfile -File <this-script>.ps1 <file1> <file2> ...` or inline via `pwsh -NoProfile -Command "& { ... }" -- <files>`.

#### Python (cross-platform; usually available wherever PowerShell isn't)

```python
import sys, re

for p in sys.argv[1:]:
    with open(p, 'rb') as f:
        raw = f.read()
    had_bom = raw.startswith(b'\xef\xbb\xbf')
    body = raw[3:] if had_bom else raw
    text = body.decode('utf-8')
    new  = re.sub(r'(?m)^[ \t]*<Private>True</Private>[ \t]*\r?\n', '', text)
    if new != text:
        out = new.encode('utf-8')
        if had_bom:
            out = b'\xef\xbb\xbf' + out
        with open(p, 'wb') as f:
            f.write(out)
        print(f'stripped: {p}')
```

Run as `python3 <this-script>.py <file1> <file2> ...`.

#### Edit-tool fallback for small changes

When only a handful of files need a single `<Private>True</Private>` line removed, direct `Edit` tool calls work too — anchor each replacement on a unique surrounding line to avoid ambiguity. The Edit tool preserves BOMs by construction. This is often cleaner for one-off cleanups than running a script.

### Files that started with no BOM

Both reference implementations above handle the no-BOM case correctly: a file that started without a BOM stays without a BOM. The detection branch (`had_bom`) drives both the decode-offset and the re-emit step, so the file's BOM state is a strict round-trip.

---

## runsettings deletion

**Symptom:** After `nuget update`, `git status` shows several `D test/<Project>.UnitTests/nano.runsettings` entries. The package version bump fully completed, but the test project lost its runsettings.

**Cause:** `-FileConflictAction Ignore` (correctly used) causes the install step to skip reinstalling the runsettings. But the *uninstall* step of the previous package version still removes it. Net effect: the file is gone.

**Fix in a git repo:**

```
git restore -- "**/test/**/nano.runsettings"
git restore -- "test/**/nano.runsettings"     # some repos use this layout
```

The deletions become no-ops because the working-tree file is restored from HEAD. Verify with `git status` — no `D ...nano.runsettings` should remain.

**Fix outside a git repo:** None automatic. Warn the user; they need to restore from a backup or by reinstalling the TestFramework package (which would prompt again — back to the [runsettings prompt](#runsettings-prompt) issue). Outside-git is rare for this skill's audience; preserve the warning rather than silently proceeding.

---

## nuspec drift

**Symptom:** A NuGet package the skill's author owns (e.g. `CCSWE.nanoFramework.MdnsServer`) ships a `.nupkg` whose dependency declarations point at *old* versions of upstream packages (`System.Net 1.11.50`), but whose binary inside the `.nupkg` references *new* assembly versions (`System.Net 1.11.52`). Consumers install the package, NuGet resolves the declared old dep, and at runtime nanoCLR refuses to load with `Error: a3000000` (CLR_E_FILE_NOT_FOUND for the new assembly version it actually needs).

**Cause:** `nuget update` updates `packages.config` and the `<Reference Include="...">` strings inside the `.nfproj`. It does **not** touch `*.nuspec` files. The author has to update the nuspec by hand. If they forget, the published nupkg's declared deps drift from the binary's actual deps. The drift is invisible during local build (the project's own `packages.config` is authoritative for local references), but visible to every downstream consumer.

**Anchor incident:** This was the root cause of the Emily.Clock runtime load failure during the CCSWE.nanoFramework 1.1.133 → 1.1.135 cycle. 1.1.133 shipped with stale nuspec deps; consumers couldn't load it. 1.1.135 corrected the nuspecs after this skill's `fix` subcommand surfaced the drift.

**Fix:** The `fix` subcommand's Check 4 catches it. Apply the user-confirmed nuspec-version update. Then republish (`1.1.<bump+1>`) — the published `1.1.133` nupkgs on nuget.org keep their stale deps until superseded; you cannot retroactively fix a published version.

---

## Bare Reference Includes

**Symptom:** A `<Reference Include="X">` with a valid `<HintPath>` into `packages\` but no `Version=`, `Culture=`, or `PublicKeyToken=` in the `Include` attribute:

```xml
<Reference Include="nanoFramework.Logging">
  <HintPath>..\..\packages\nanoFramework.Logging.1.1.161\lib\nanoFramework.Logging.dll</HintPath>
</Reference>
```

These survive in hand-authored projects (especially benchmark projects that were not regenerated by Visual Studio's NuGet UI). They build fine locally but downstream tooling that parses `.nfproj`s expects the full identity.

**Fix:** Fill the canonical assembly identity into the `Include` attribute. Lookup procedure:

1. **Sibling-project search.** Scan every other `.nfproj` in the repository for a `<Reference>` whose `<HintPath>` resolves to the same package folder (`packages\<PackageId>.<Version>\`). If such a reference has a complete `Include` (`X, Version=A.B.C.D, Culture=neutral, PublicKeyToken=...`), copy it verbatim.

2. **DLL inspection fallback.** When no sibling project provides a canonical form, read the assembly identity from the DLL itself:

   ```powershell
   [System.Reflection.AssemblyName]::GetAssemblyName('C:\path\to\packages\<PackageId>.<Version>\lib\<dll-name>.dll').FullName
   # → "X, Version=1.0.0.0, Culture=neutral, PublicKeyToken=c07d481e9758c731"
   ```

   This is reliable and matches what MSBuild would compute itself. `Reflection.AssemblyName.GetAssemblyName` does not load the assembly — it reads metadata only — so there is no nanoFramework / .NET Framework compatibility issue.

3. **Both failed.** Report as unfixable. Do not fabricate an identity.

---

## Cross-cutting: never auto-commit

`nuget update` touches dozens of files. Every dep-bump session in this repo has ended with the user reviewing the diff manually before staging. Do not stage or commit from inside this skill. Stop at `git diff --stat` and let the user own the commit.
