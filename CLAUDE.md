# CLAUDE.md

Guidance to Claude when working on the `nf-deps` skill itself (this repo). For consumer-facing docs see [README.md](README.md); for skill behavior see [SKILL.md](SKILL.md).

## Repository shape

This repo is a Claude Code skill — markdown prompts plus one PowerShell helper. There is no build, lint, or test runner. "Testing" a change means cloning the repo into `~/.claude/skills/nf-deps/`, starting a fresh Claude Code session, and running the affected subcommand against a real nanoFramework solution. The canonical end-to-end test is `/nf-deps update` from a CCSWE.nanoFramework or Emily.Clock working tree — both have been used as the dogfood targets.

The skill name `/nf-deps` is derived from the install directory; the README's Installation section makes this explicit. Do not rename the directory.

## Three-tier layout (load-bearing)

- **`SKILL.md`** — dispatcher + cross-cutting policy block. Keep lean. New cross-cutting rules go here only when both subcommands consume them; rules used by exactly one subcommand belong in that subcommand file.
- **`subcommands/<name>.md`** — full prompt per subcommand (Usage, Inputs, Steps, Guardrails). Loaded on demand by the dispatcher.
- **`references/<topic>.md`** — canonical knowledge subcommands link into rather than duplicate. Currently only `nuget_quirks.md`. New domain knowledge (e.g. nanoFramework's project-file schema, Nerdbank.GitVersioning quirks) should land here as new files rather than expanding `nuget_quirks.md` beyond its current five sections.

When adding a subcommand: extend the dispatch table in `SKILL.md`, add `subcommands/<name>.md`, and link to relevant `references/` sections rather than restating them.

## Implementation philosophy: prompt-driven, environment-agnostic

This skill does **not** ship executable scripts. The `<Private>True</Private>` cleanup step in `update` is described in the subcommand as a procedure with a semantic check (compare HEAD vs working tree, strip only newly-added entries) and the `references/nuget_quirks.md` file carries reference implementations in PowerShell and Python. Claude picks whichever fits the current environment, or uses the Edit tool directly when the edit volume is small.

The reasoning: the skill's audience runs Claude Code on Windows (the common case) and occasionally on macOS/Linux, and committing to one shell would either force pwsh-on-Unix or sed-on-Windows. Both work, neither is universal, and Claude is perfectly capable of translating between them given the reference implementations as a baseline. Keeping the canonical logic in prose + worked examples — rather than as a single mandatory script — also makes the semantic check (HEAD vs working tree) easy to express without contorting it into a shell pipeline.

If a future need genuinely requires deterministic helper code (e.g. a fast canonical-identity extractor that runs against thousands of references), add it under a new `scripts/` directory and document the same way: the script is the *reference*, not the *only* implementation, and the subcommand explains the procedure regardless.

## Cross-skill coupling

None today. This skill consumes no other skill, and no other skill consumes its surface. If that changes (e.g. a release-prep skill that calls `/nf-deps fix` as a gate), document the contract here.

## Writing style

Imperative / infinitive form throughout the skill body — "Strip the Private elements" not "You should strip the Private elements". Frontmatter description is third-person ("This skill should be used when...") — the existing description follows the pattern, preserve it on edits.

Keep `SKILL.md` under ~2000 words. The body currently sits well under that; keep it lean. Push detail into subcommands and references.

## Sync risks (#1 drift risk in the repo)

- `nuget update` flag list. `-FileConflictAction Ignore` is mentioned in three places: `SKILL.md § Cross-cutting policy`, `subcommands/update.md § Step 4`, and `references/nuget_quirks.md § runsettings prompt`. If the flag set changes (e.g. NuGet introduces a non-interactive global flag that supersedes it), update all three. The `references/` entry is the canonical explanation; the two other mentions are short cross-references back to it.
- The `<Private>` cleanup procedure is described in `subcommands/update.md § Step 6`. Reference implementations live in `references/nuget_quirks.md § Private noise + BOM preservation`. If the procedure changes (e.g. a new failure mode is discovered), update both: the procedural description in the subcommand and the reference write-up.

## Commit / PR conventions

Plain conventional-ish messages — "Add update subcommand", "Fix BOM handling in strip-private.ps1", "Document nuspec drift root cause". No Co-Authored-By or AI-attribution lines per the user's standing rule across CCSWE-nanoFramework repos. No issue-tracker prefix is required; the repo has no issue-tracker convention yet.

## Publishing

This is the canonical repo at https://github.com/CCSWE-nanoFramework/nf-deps-skill. Merge to `main` is the release. Consumers pull updates with `git -C ~/.claude/skills/nf-deps pull` and the new behavior takes effect on the next `/nf-deps ...` invocation — no Claude Code restart required.
