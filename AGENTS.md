# AGENTS.md

## Project Type

A grammY Telegram-bot **skill collection** (modeled on samber/cc-skills-golang).
Each `skills/<name>/` directory is a self-contained skill consumable by any agent
runtime that supports the `name` + `description` frontmatter protocol (Claude,
Cursor, Copilot, etc.). Two tiers: a **general grammY core** and an **agnt-gm
platform tier**.

## Structure

- `skills/telegram-how-to/` — orchestrator; routes intent to both tiers; `references/` hold the catalog, disambiguation tables, and the configure template
- **General core:** `skills/telegram-bot-{basics,ui,sessions,conversations,messages,scaling,deploy,payments,mini-apps,testing,security}/` — pure grammY, no platform lock-in
- **agnt-gm tier:** `skills/{agnt-cli-builder,telegram-test-specs,telegram-test-advanced,agntdev-deploy}/` — correct but platform-specific
- `RULES.md` (repo root) — consolidated cross-cutting rule set
- `references/COMMANDS.md` (under `agnt-cli-builder/`) — auto-generated
  from the agnt-cli repo's oclif manifest. **Do not hand-edit.**

## Authoring conventions (general core)

- **Pure grammY only** — no `@agntdev/*` imports, no `createBot`/`makeBot` toolkit
  helpers. Platform specifics belong in the agnt-gm tier.
- **Verify grammY APIs against grammy.dev** before writing snippets — wrong API
  names are what make a skill "wrong."
- Keep each `SKILL.md` to the lean bar set by `telegram-bot-basics`: concept → API →
  minimal snippet, a `## Common mistakes` block, sibling cross-refs, a quick reference.

## Regenerating the COMMANDS.md reference

The `agnt-cli-builder/references/COMMANDS.md` file is auto-generated
from the agnt-cli repo's oclif manifest. It is the source of truth for
the command tree the skill teaches agents.

After any change in agnt-cli commands, regen the file from the agnt-cli
repo:

```sh
# From the agnt-cli repo root:
npx oclif readme --readme-path ../agntdev-skills/skills/agnt-cli-builder/references/COMMANDS.md
```

**Never hand-edit `references/COMMANDS.md`.** Hand-edits get clobbered
on the next regen. The oclif-generated version is authoritative
(aliases, source links, ordering, exit code notes all match the
runtime). The only thing the skill author edits is the `SKILL.md`
file in each skill — that one is hand-written.

The corresponding note lives in `agnt-cli/AGENTS.md` so the CLI side
also knows the regen command.

## SKILL.md conventions

- `name:` matches the directory name.
- `description:` is a long string (multi-line YAML `>`). The first
  sentence is the trigger; the rest is context.
- `Triggers:` block is a comma-separated list of phrases the agent
  runtime matches against user input.
- `compatibility:` line documents required tools / env (Node version,
  gh CLI, network access).
- On Activation block: commands to run when the skill loads, before
  the user asks. Saves a round-trip.
- Quick Reference block at the bottom: copy-pasteable command list.
  Keep it in sync with `references/COMMANDS.md`.
