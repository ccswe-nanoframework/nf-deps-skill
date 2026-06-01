# nf-deps skill

A Claude Code skill that keeps .NET nanoFramework project dependencies consistent across `.nfproj`, `packages.config`, `.nuspec`, and `packages.lock.json` — and runs the `nuget update` flow with the safety steps that aren't documented anywhere else.

## Subcommands at a glance

| Subcommand | What it does |
|---|---|
| `/nf-deps fix [project-name-or-path?]` | Validate dep sync across `.nfproj` / `packages.config` / `.nuspec` / lock; report drift; offer to repair the categories it can fix mechanically. Default when no subcommand is given. |
| `/nf-deps update` | Solution-level. `nuget restore` + `nuget update -FileConflictAction Ignore`, strip the `<Private>True</Private>` noise NuGet inserts, restore deleted `nano.runsettings` (in a git repo), then run `fix` for final validation. Ends with `git diff --stat`; no auto-commit. |

## Prerequisites

- **`nuget.exe` on `PATH`** — used by `update`.
- **A nanoFramework solution** in the current working directory for `update` (exactly one `*.sln`).
- **Git** for the `nano.runsettings` restore step, the semantic HEAD-vs-working-tree check on `<Private>` cleanup, and the final `git diff --stat`. Strongly recommended; the `update` subcommand degrades gracefully outside git but prompts more.
- **PowerShell or Python** — only if the `<Private>` cleanup step needs to script the edit across many files. For small change sets Claude can use the Edit tool directly. The skill makes no specific shell assumption; reference implementations are provided in both languages under `references/nuget_quirks.md`.

`msbuild` and `vstest.console.exe` are not required by this skill itself — they're used to verify the result after the skill finishes.

## Installation

Clone the repository directly into `~/.claude/skills/nf-deps` — the directory name **must** be exactly `nf-deps` because Claude Code derives the slash command (`/nf-deps`) from the install directory.

```bash
git clone https://github.com/CCSWE-nanoFramework/nf-deps-skill.git ~/.claude/skills/nf-deps
```

> **Do not** clone into `~/.claude/skills/nf-deps-skill/` or any other path — the skill name won't resolve.

Start a new Claude Code session. `/nf-deps` should appear in the available skills list and be invokable immediately.

### Verifying the install

```bash
ls ~/.claude/skills/nf-deps/
# Expected: README.md  CLAUDE.md  SKILL.md  subcommands/  references/
```

In a Claude Code session, from a nanoFramework repo root:

```
/nf-deps fix
```

The skill should walk every `.nfproj` and report findings (or `0 errors / 0 warnings` on a clean tree).

## Repository layout

```
.
├── README.md                 # this file
├── CLAUDE.md                 # guidance to Claude when editing the skill itself
├── SKILL.md                  # dispatcher + cross-cutting policy
├── subcommands/
│   ├── fix.md                # /nf-deps fix — validation + repair
│   └── update.md             # /nf-deps update — nuget update flow with cleanup
└── references/
    └── nuget_quirks.md       # canonical write-up of the five nuget update pitfalls
                              # (includes reference PowerShell and Python implementations
                              #  for the BOM-preserving <Private> cleanup)
```

`SKILL.md` is the entry point. It dispatches to the right file under `subcommands/` based on the first argument, and `subcommands/` files link into `references/nuget_quirks.md` for the *why* behind each step. There are no executable scripts — the skill is pure markdown prompts plus reference code examples Claude executes via whatever tools are available.

## What the `update` subcommand actually does

The hand flow it replaces, distilled from real dep-bump sessions:

1. Locate the `.sln`.
2. `nuget restore <sln>`.
3. `nuget update <sln> -FileConflictAction Ignore` — **the flag is mandatory**; without it, NuGet prompts to overwrite `nano.runsettings` files and hangs forever in a Claude Code session.
4. Strip the `<Private>True</Private>` lines NuGet inserted into modified `.nfproj` files — only the lines that are new vs HEAD; any `<Private>` already in HEAD is preserved. The cleanup preserves the file's original BOM state (naive UTF-8 read/write strips BOMs and pollutes the diff).
5. `git restore -- "**/test/**/nano.runsettings"` — NuGet uninstalled the runsettings files when it bumped `nanoFramework.TestFramework`; restore them from HEAD.
6. Run the `fix` subcommand to catch any `.nuspec` drift (NuGet bumps `packages.config` but not `.nuspec`) and any pre-existing metadata issues.
7. Print `git diff --stat`. Stop. The user reviews and commits.

All five steps have caused real incidents — see [`references/nuget_quirks.md`](references/nuget_quirks.md) for the postmortems.

## Update cadence

```bash
git -C ~/.claude/skills/nf-deps pull
```

The skill is markdown plus one PowerShell helper, so updates take effect on the next `/nf-deps ...` invocation. No Claude Code restart needed.

## Contributing

Open a PR against `main`. Please keep:

- `SKILL.md` lean — it's the dispatcher plus the cross-cutting policy block, nothing more.
- Subcommand-specific detail inside `subcommands/<name>.md`.
- Quirks and reasoning inside `references/*.md`, linked from subcommand files rather than duplicated.
- The `<Private>` cleanup procedure described in `subcommands/update.md` (procedure) + `references/nuget_quirks.md` (reference implementations and failure-mode write-up) — do not inline the regex into other files.

See [`CLAUDE.md`](CLAUDE.md) for the load-bearing layout invariants and the sync-risk hotspots.
