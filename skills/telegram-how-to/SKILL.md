---
name: telegram-how-to
description: >
  Telegram bot skills orchestrator — load the right skill(s) for any bot task.
  Routes intent to telegram-bot-basics, sessions, ui, test-specs, test-advanced,
  deploy, and agnt-cli-builder. Disambiguates overlapping topics (sessions vs
  conversations, inline vs reply keyboards, declarative vs programmatic tests).
  Triggers: telegram bot, how to build telegram bot, which telegram skill,
  telegram bot help, bot skill routing, /telegram-how-to, organize bot work.
compatibility: Works with grammY and @agntdev/bot-toolkit. agnt-cli-builder needs Node 18+, gh CLI.
license: MIT
user-invocable: true
---

**Persona:** You are a Telegram bot skills orchestrator. For every bot task, identify all relevant skills and load them together — a task rarely belongs to a single skill.

**Modes:**

- **Orchestrate** — for any bot coding, review, test, or deploy task, load the primary skill plus all applicable secondary skills at the start.
- **Disambiguate** — when two skills seem to overlap, show the boundary table. See [disambiguation.md](references/disambiguation.md).
- **Configure** — add a `## Required Telegram bot skills` block to the project's `CLAUDE.md` or `AGENTS.md`. Follow [project-config.md](references/project-config.md).

## Skill loading

For each task, load the **primary skill** and all applicable **secondary skills** at the same time. Do not wait — load them together at the start.

| Intent | Primary | Also load |
| --- | --- | --- |
| Find paid work / claim a task / ship PR | [agnt-cli-builder](../agnt-cli-builder/SKILL.md) | Skill for the task type (basics, ui, sessions, tests, deploy) |
| New bot / entry point / commands / callbacks | [telegram-bot-basics](../telegram-bot-basics/SKILL.md) | [telegram-test-specs](../telegram-test-specs/SKILL.md) if adding commands |
| Multi-step dialog / user state / booking flow | [telegram-bot-sessions](../telegram-bot-sessions/SKILL.md) | [telegram-bot-ui](../telegram-bot-ui/SKILL.md) if menus/buttons drive the flow |
| Inline buttons / menus / pagination / confirm | [telegram-bot-ui](../telegram-bot-ui/SKILL.md) | [telegram-bot-basics](../telegram-bot-basics/SKILL.md) for callback routing |
| Write dialog tests / BotSpec JSON / coverage | [telegram-test-specs](../telegram-test-specs/SKILL.md) | [telegram-bot-basics](../telegram-bot-basics/SKILL.md) (`makeBot()` contract) |
| Mock DB/HTTP / API failures / payment tests | [telegram-test-advanced](../telegram-test-advanced/SKILL.md) | [telegram-test-specs](../telegram-test-specs/SKILL.md) |
| Deploy / container crash / dist/index.js / Redis | [telegram-bot-deploy](../telegram-bot-deploy/SKILL.md) | [telegram-bot-sessions](../telegram-bot-sessions/SKILL.md) if session storage |
| Review bot PR / audit patterns | [telegram-bot-basics](../telegram-bot-basics/SKILL.md) | [telegram-test-specs](../telegram-test-specs/SKILL.md), domain skill for changed area |

## Cold-start decision tree

```
User mentions agntdev / TON / claim / paid task?
  YES → agnt-cli-builder first, then domain skill for the task
  NO  ↓

Changing handlers, routing, makeBot(), project layout?
  YES → telegram-bot-basics (+ test-specs if commands added)

Building keyboards, pagination, confirm dialogs?
  YES → telegram-bot-ui (+ sessions if flow has steps)

Persisting user state across messages?
  YES → telegram-bot-sessions (+ ui if buttons advance steps)

Writing or fixing tests?
  YES → test-specs (declarative JSON)
        test-advanced if mocks, 429, blocked user, payments

Bot won't start / deploy / Redis / .npmrc?
  YES → telegram-bot-deploy
```

## Categories at a glance

Full catalog with "use when" hooks: [by-category.md](references/by-category.md)

| Category | Skills |
| --- | --- |
| Pipeline | `agnt-cli-builder` |
| Core bot | `telegram-bot-basics` |
| State & flows | `telegram-bot-sessions` |
| UI | `telegram-bot-ui` |
| Testing | `telegram-test-specs` `telegram-test-advanced` |
| Production | `telegram-bot-deploy` |

## Competing clusters — boundary lines

Full boundary tables with routing examples: [disambiguation.md](references/disambiguation.md)

Key clusters:

- **State**: `telegram-bot-sessions` (per-chat `ctx.session`) · manual DB/Map (avoid unless skill says otherwise) · grammY conversations plugin (not in toolkit MVP — use sessions skill)
- **UI**: `telegram-bot-ui` (keyboards, builders) · `telegram-bot-basics` (raw `callbackQuery` routing)
- **Tests**: `telegram-test-specs` (BotSpec JSON, coverage gate) · `telegram-test-advanced` (mocks, error paths, `handleUpdate`)
- **Toolkit layers**: Bot API concept → grammY → `@agntdev/bot-toolkit` — every domain skill follows this order; load basics if the agent skips a layer

## Universal rules (every bot task)

These apply regardless of which skill loads:

1. **`makeBot()` returns a fresh bot** — no module-level singleton; harness needs isolation.
2. **`await ctx.answerCallbackQuery()`** in every callback handler — or the spinner never stops.
3. **`await` all API calls** — `ctx.reply`, `editMessageText`, etc.
4. **Never commit `BOT_TOKEN`** — use `process.env.BOT_TOKEN`.
5. **Tests gate publish** — all BotSpec specs pass + declared command coverage (agntdev pipeline).
6. **Canonical entry** — `dist/index.js` from `src/index.ts`; see deploy skill for legacy fallbacks.

## Configure mode

Force-trigger specific skills in a project's `CLAUDE.md` or `AGENTS.md` so they always load.

When invoked as `/telegram-how-to configure`, follow [project-config.md](references/project-config.md).

---

This skill routes to the domain skills in this bundle. Read the linked `SKILL.md` files for implementation detail — do not improvise patterns that contradict them.