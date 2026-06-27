# AGENTS.md

## Project Type

A grammY Telegram-bot **skill collection** (modeled on samber/cc-skills-golang).
Each `skills/<name>/` directory is a self-contained skill consumable by any agent
runtime that supports the `name` + `description` frontmatter protocol (Claude,
Cursor, Copilot, etc.).

## Structure

- `skills/telegram-how-to/` — orchestrator; routes intent to the domain skills; `references/` hold the catalog, disambiguation tables, and the configure template
- `skills/telegram-bot-{basics,ui,sessions,conversations,messages,scaling,deploy,payments,mini-apps,testing,security}/` — the domain skills
- `RULES.md` (repo root) — consolidated cross-cutting rule set

## Authoring conventions

- **Pure grammY only** — no framework or platform lock-in in the snippets. Verify APIs
  against grammy.dev before writing — wrong API names are what make a skill "wrong."
- Keep each `SKILL.md` to the lean bar set by `telegram-bot-basics`: concept → API →
  minimal snippet, a `## Common mistakes` block, sibling cross-refs, a quick reference.
- Cross-reference sibling skills by relative path (`../<name>/SKILL.md`); link the rule
  set as `../../RULES.md` from a `SKILL.md` and `../../../RULES.md` from a `references/` file.

## SKILL.md conventions

- `name:` matches the directory name.
- `description:` is a multi-line YAML `>` block. The first sentence is the trigger; the
  rest is context. It must include a `Triggers:` line — a comma-separated list of phrases
  the agent runtime matches against user input.
- `compatibility:` documents required tools / env (grammY version, Node version, plugins).
- `license:` is an SPDX id from the set the validator allows (use `MIT`).
- A copy-pasteable **Quick reference** block at the bottom.

Run `node scripts/validate-skills.mjs` before committing — it checks frontmatter and that
every `references/` link resolves.
