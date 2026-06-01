---
name: nf-deps
description: Multi-subcommand skill for keeping .NET nanoFramework project dependencies consistent across .nfproj, packages.config, .nuspec, and packages.lock.json. First argument is a literal subcommand. Subcommands — `fix [project-name-or-path?]` (validate dep sync and offer to repair drift; the default when no subcommand is given), `update` (run nuget restore + nuget update on the solution in the current directory with the safety flags learned the hard way, strip <Private>True</Private> noise, restore deleted nano.runsettings, then delegate to fix). Use when the user invokes `/nf-deps ...`, says "validate nanoFramework dep sync", "check .nfproj / .nuspec / packages.config alignment", "fix dependency drift", "run nuget update on this solution", "update nanoFramework packages", "bump CCSWE packages", or has just run `nuget update` and wants the cleanup pass.
version: 0.1.0
---

# nf-deps

Dispatcher skill. Keep `.nfproj`, `packages.config`, `.nuspec`, and `packages.lock.json` consistent across a nanoFramework solution, and run the `nuget update` flow with the safety steps that aren't documented anywhere else.

| Subcommand | File | Purpose |
|---|---|---|
| `fix` | [subcommands/fix.md](subcommands/fix.md) | Validate dep sync across `.nfproj` / `packages.config` / `.nuspec` / lock; offer to repair drift. Optional positional project-name-or-path filter. **Default when no subcommand is given.** |
| `update` | [subcommands/update.md](subcommands/update.md) | Solution-level. `nuget restore` + `nuget update -FileConflictAction Ignore`, strip `<Private>True</Private>` noise (BOM-preserving), restore deleted `nano.runsettings`, then run `fix`. |

## Dispatch

1. Parse the first argument as the subcommand.
2. If it is `fix`, **or empty/missing**, read [subcommands/fix.md](subcommands/fix.md) and follow it. Any second argument is the optional project-name-or-path filter passed through.
3. If it is `update`, read [subcommands/update.md](subcommands/update.md) and follow it. The subcommand takes no further arguments.
4. If the first argument is anything else, stop and list the supported subcommands from the table above.

Subcommands link into [references/nuget_quirks.md](references/nuget_quirks.md) for the canonical write-up of the `nuget update` pitfalls that motivated this skill. Read that reference once when working on this skill; do not duplicate its content into individual subcommands.

## Cross-cutting policy (consumed by both subcommands)

These rules apply to every code path in this skill. They came from real incidents during the CCSWE.nanoFramework dep-bump session that produced this skill.

- **Preserve BOM state when editing `.nfproj` files.** Some nanoFramework projects ship with a UTF-8 BOM, others don't, and it varies by file even within one repo. Naive UTF-8 read/write strips BOMs from BOM files and pollutes the diff with encoding noise. See [references/nuget_quirks.md § Private noise + BOM preservation](references/nuget_quirks.md) for the failure mode and reference implementations in PowerShell and Python.
- **Distinguish new `<Private>` entries from pre-existing ones.** Do not unconditionally strip `<Private>True</Private>` from `.nfproj` files. A project author may have set `<Private>` intentionally — what the skill targets is the noise `nuget update` *adds*, not anything that was already in HEAD. The diff against HEAD is the source of truth for what is new.
- **Never auto-stage or auto-commit.** Both subcommands end at `git diff --stat`. The user reviews and commits manually.
- **Foreground only for `nuget` commands.** `nuget update` can prompt interactively and hang silently in the background. Run it in the foreground with `-FileConflictAction Ignore` — see [references/nuget_quirks.md § runsettings prompt](references/nuget_quirks.md).
- **Use whatever shell is available.** This skill makes no PowerShell-vs-Bash assumption. PowerShell-only constructs and Bash-only constructs are both fine; pick the one matching the current environment. Reference implementations in `references/` are illustrative, not mandatory invocations.

## How this skill is organized

- **`SKILL.md`** (this file) — dispatcher + cross-cutting policy. Keep lean.
- **`subcommands/<name>.md`** — the full prompt per subcommand (Usage, Inputs, Steps, Guardrails). Loaded on demand by the dispatcher.
- **`references/<topic>.md`** — canonical reference data: the catalog of `nuget update` quirks the subcommands link into.
- **`scripts/<name>.ps1`** — deterministic helpers invoked by subcommands rather than inlined into prompts.

## Source

This skill is versioned at https://github.com/CCSWE-nanoFramework/nf-deps-skill.

Pull updates with:

```
git -C ~/.claude/skills/nf-deps pull
```

Open issues or PRs against `main`. Updates take effect on the next `/nf-deps ...` invocation — no restart required.
