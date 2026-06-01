# Subcommand: `fix`

**Usage:** `/nf-deps fix [project-name-or-path]` (or `/nf-deps` with no subcommand — `fix` is the default).

Validate that every nanoFramework project's `.nfproj`, `packages.config`, `.nuspec`, and `packages.lock.json` agree on package versions and assembly identities; report drift; offer to apply mechanical fixes. Read-only by default — never edit a file without first reporting findings and asking the user to confirm.

## Inputs

- **Optional project filter** (single positional argument) — a project name (e.g. `CCSWE.nanoFramework.WebServer`) or a path (relative or absolute) to a single `.nfproj`. When supplied, restrict every check to that project. When absent, walk every `.nfproj` under the current working directory recursively (excluding `bin/` and `obj/`).

## Steps

### 1. Discover projects

Enumerate target `.nfproj` files. Per project, locate the sibling `packages.config`, optional `*.nuspec`, and optional `packages.lock.json` in the project directory.

### 2. Build the canonical-identity map

Before checking any individual project, scan every `.nfproj` in the repository (not just the filtered set) for `<Reference>` items whose `Include` already contains `Version=`, `Culture=`, and `PublicKeyToken=` and whose `<HintPath>` resolves into a `packages\<PackageId>.<Version>\` folder. Build a map `packageFolder → canonicalInclude`. The `fix` step for incomplete metadata references this map. See [../references/nuget_quirks.md § Bare Reference Includes](../references/nuget_quirks.md) for the lookup procedure when no sibling project provides the canonical form.

### 3. Run the five checks per project

For each target project, accumulate errors and warnings as described below. Do not stop on the first finding.

#### Check 1 — Bare references

A `<Reference>` that is not a `<ProjectReference>` and has no `<HintPath>` child element. Every package reference must have a `<HintPath>`. Flag each as an error and ask the user to resolve manually; the skill does not auto-add `<HintPath>` elements.

#### Check 2 — Incomplete assembly metadata

A `<Reference>` with a `<HintPath>` pointing into `packages\` whose `Include` attribute lacks `Version=`, `Culture=`, or `PublicKeyToken=`. Look up the canonical form in the map built in Step 2. If a canonical form exists, this is fixable. If no other project in the repo references this package with a complete form, fall back to reading the assembly identity from the DLL itself:

```powershell
[System.Reflection.AssemblyName]::GetAssemblyName('<absolute-path-to-dll>').FullName
```

If the DLL is missing (e.g. `nuget restore` has not been run), report the issue as unfixable until restore happens.

#### Check 3 — packages.config ↔ .nfproj HintPath sync

For every `<Reference>` with a `<HintPath>` pointing into `packages\`:

- Extract `<PackageId>` and `<Version>` from the path (`packages\{PackageId}.{Version}\...`). The version is the longest dotted suffix matching `\d+\.\d+(?:\.\d+){0,2}` — use a non-greedy capture for the package id (`^(.+?)\.(\d+\.\d+(?:\.\d+){0,2})$`) to avoid eating part of the version into the id.
- Verify a matching `<package id="..." version="..." />` entry exists in `packages.config`. Mismatch → error.
- Track which package ids have a corresponding `<Reference>`.

Conversely, every package in `packages.config` (excluding `developmentDependency="true"` entries) that ships a DLL should have a corresponding `<Reference>`. Flag packages without one as a warning (likely transitive-only). Skip the well-known transitive metapackages (`nanoFramework.CoreLibrary`, `Nerdbank.GitVersioning`) from this warning — they almost never have a direct reference.

#### Check 4 — packages.config ↔ .nuspec sync

For every `<dependency id="..." version="..." />` in the project's `.nuspec` whose `id` is not the sibling-package placeholder pattern (`$version$`):

- Look up the same `id` in `packages.config`.
- Compare the `version` attributes exactly. Mismatch → error.

Version ranges (`[1.0,2.0)`) are uncommon for nanoFramework deps; treat them as equal to the lower bound for comparison purposes.

#### Check 5 — `packages.lock.json` presence

If `packages.lock.json` does not exist in the project directory, warn — `nuget restore` needs to be run before the rest of the checks can be trusted (lock-file is the cross-reference for transitive resolution).

### 4. Report findings

Use this output format per project. Skip projects with no findings entirely from the per-project report; include them only in the trailing summary.

```
### <relative-path-to-.nfproj>

WARN: <package-id> in packages.config has no corresponding Reference (transitive-only?)
ERR:  Bare reference: <Reference Include="X" /> (no HintPath)
ERR:  Incomplete metadata for "X": Include="X" — canonical: "X, Version=1.2.3.0, Culture=neutral, PublicKeyToken=abc..."
ERR:  HintPath/packages.config mismatch: X — HintPath has 1.2.3, packages.config has 1.2.4
ERR:  .nuspec/packages.config mismatch: X — nuspec has 1.2.3, packages.config has 1.2.4
```

End with `Summary: <N> errors, <M> warnings across <P> projects`.

### 5. Ask before applying fixes

If any findings exist, ask the user whether to apply automatic fixes. **Do not edit without explicit confirmation.** When confirmed, apply only the categories the skill can fix mechanically and safely:

- **Update HintPath version strings** in `.nfproj` to match the version listed in `packages.config` (Check 3 mismatches).
- **Update `.nuspec` dependency `version` attributes** to match `packages.config` (Check 4 mismatches).
- **Fill missing `Version=` / `Culture=` / `PublicKeyToken=`** in Reference `Include` attributes (Check 2), using either the canonical form from the sibling-project map or the DLL identity.

Do **not** auto-add missing `<Reference>` entries, remove `<package>` entries, fabricate canonical metadata when both sibling and DLL lookups fail, or alter `<HintPath>` elements themselves (only the version embedded in their containing `Include` string). Flag those for the user to handle manually.

All write operations must preserve BOM state on `.nfproj` files. See [../SKILL.md § Cross-cutting policy](../SKILL.md) and [../references/nuget_quirks.md § Private noise + BOM preservation](../references/nuget_quirks.md). When in doubt, defer the edit and ask the user.

## Guardrails

- Read-only until the user confirms fixes.
- Never auto-stage or commit. Leave the working tree changed; the user reviews and commits.
- Never invent assembly identities. If both sibling-project lookup and DLL inspection fail, report the issue as unfixable.
- Do not touch projects outside the supplied filter (when one is given).
- Skip `bin/` and `obj/` directories during discovery.
