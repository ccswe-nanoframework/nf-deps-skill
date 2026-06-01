# Subcommand: `update`

**Usage:** `/nf-deps update`

Run the solution-level dep-update flow: restore packages, run `nuget update` with the safety flags, clean up the noise `nuget update` introduces, restore files it deletes, then delegate to `fix` for final validation. End with a diff stat for human review — no auto-commit.

## Inputs

None. The current working directory must contain exactly one `*.sln` file. If it contains zero or multiple, stop and ask the user to clarify.

## Steps

### 1. Locate the solution

Find `*.sln` in the cwd. Error if zero or multiple matches. Store the absolute path as `$sln`.

### 2. Detect git context

Run `git rev-parse --is-inside-work-tree 2>/dev/null`. If exit code 0 and output is `true`, set `$inGit = true`. Otherwise `$inGit = false`. The result drives later branches (file-set determination in Step 5, `nano.runsettings` restore in Step 7, final diff in Step 9).

### 3. Restore packages

Run `nuget restore "$sln"` in the foreground. Required before `update` so that `update` has the current resolved state to compare against. A clean restore that prints "All packages listed in packages.config are already installed." is the expected state.

### 4. Run nuget update

Run, in the foreground:

```
nuget update "$sln" -FileConflictAction Ignore
```

**Never omit `-FileConflictAction Ignore`.** Without it, `nuget update` prompts interactively when an installed `nano.runsettings` file already exists, and the prompt hangs forever in Claude's Bash environment with no way to answer. See [../references/nuget_quirks.md § runsettings prompt](../references/nuget_quirks.md) for the postmortem.

**Never background this call.** A backgrounded hang is invisible until the user notices.

### 5. Determine the modified `.nfproj` set

- If `$inGit`: `git diff --name-only -- '*.nfproj'`.
- Otherwise: every `.nfproj` under the solution directory recursively (excluding `bin/` and `obj/`).

Pass the absolute paths to Step 6. If the set is empty, skip Step 6 and continue.

### 6. Strip newly-added `<Private>True</Private>` noise

`nuget update` adds a `<Private>True</Private>` child to every `<Reference>` it touches. These pollute the diff and are not the convention in any of the CCSWE.nanoFramework repos. **Strip only the entries `nuget update` just added — never the ones that were already in HEAD.** A project author may have set `<Private>` intentionally.

For each modified `.nfproj`, compare its working-tree contents against `HEAD:<path>`:

- If HEAD contains zero `<Private>` lines and the working tree contains some, every working-tree `<Private>` entry is new noise — strip them all.
- If HEAD contains some `<Private>` lines and the working tree contains more, only the newly-added entries are noise. Compute the set of new entries (line-by-line diff scoped to `<Private>` lines is sufficient) and strip only those. The cleanest way to do this in practice is to revert each `<Reference>` block to its HEAD form when the only change is the addition of `<Private>True</Private>` — `git diff` shows exactly those blocks.
- If not in a git repo (no HEAD baseline available), prompt the user before stripping anything: list every `<Private>` entry in the modified files and ask whether to remove them.

**BOM preservation is mandatory** on every write. See [../references/nuget_quirks.md § Private noise + BOM preservation](../references/nuget_quirks.md) for the failure mode and reference implementations in PowerShell and Python. Pick the implementation matching the current environment, adapt as needed, or use the Edit tool directly when each `.nfproj` only has a few `<Private>` lines to remove and the line-anchored edits are tractable.

### 7. Restore deleted `nano.runsettings`

`nuget update` deletes the `nano.runsettings` file from every test project whose `nanoFramework.TestFramework` package was touched (the package ships one as content; updating the package causes uninstall→reinstall, and the install step is what `-FileConflictAction Ignore` skipped).

- If `$inGit`: `git restore -- "**/test/**/nano.runsettings"` and `git restore -- "test/**/nano.runsettings"`. Either may report "no such path" if the project layout differs — tolerate it. Verify with `git status` that no `D ...nano.runsettings` entries remain.
- Otherwise: print a warning that deleted `nano.runsettings` files cannot be auto-restored without git, and list the files that would need manual restoration from a backup or fresh package install.

### 8. Delegate to `fix`

Read [fix.md](fix.md) and follow it against the solution directory (no project filter). This catches:

- New `.nuspec` drift (`nuget update` bumps `packages.config` but not `.nuspec` — see [../references/nuget_quirks.md § nuspec drift](../references/nuget_quirks.md)).
- Any pre-existing bare references or incomplete metadata.
- Lock-file presence.

Apply the user-confirmed fixes the `fix` subcommand offers.

### 9. Show the result

If `$inGit`, print `git diff --stat`. Do not stage or commit — that's the user's call. End with a one-line summary:

> Done. <N> .nfproj files modified, <M> .nuspec files modified, <K> nano.runsettings restored. Review the diff and commit when ready.

## Guardrails

- **Foreground only** for `nuget restore` and `nuget update`. Do not background them.
- **`-FileConflictAction Ignore` is mandatory** on `nuget update`. Skipping it has caused multi-minute hangs that required killing the process.
- **BOM-preserving writes only.** See the reference implementations in `references/nuget_quirks.md`.
- **Strip only newly-added `<Private>` entries**, never ones present in HEAD. Outside a git repo, prompt before stripping anything.
- **No auto-commit.** Stop at `git diff --stat`.
- **Single-solution scope.** Refuse to operate when zero or multiple `.sln` files are present in the cwd; ask the user to disambiguate.
- **Trust `fix` for repair.** Do not implement repair logic in this subcommand; delegate.
