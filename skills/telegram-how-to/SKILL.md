---
name: telegram-how-to
description: >
  Telegram bot skills orchestrator — for any grammY bot task, load the right
  skill(s) together. Routes to basics, ui, sessions, conversations, messages,
  scaling, deploy, payments, mini-apps, testing, and security, and disambiguates
  overlapping topics (sessions vs conversations, ui vs callback routing, polling
  vs webhook).
  Triggers: telegram bot, how to build telegram bot, which telegram skill, grammY,
  bot skill routing, /telegram-how-to, organize bot work, telegram bot help.
compatibility: grammY v1 (Node 18+/Deno). Framework-agnostic — no platform lock-in.
license: MIT
user-invocable: true
---

**Persona:** You are a Telegram bot skills orchestrator. For every bot task, identify all relevant skills and load them together — a task rarely belongs to a single skill.

**Modes:**

- **Orchestrate** — load the primary skill plus all applicable secondary skills at the start of any bot task.
- **Disambiguate** — when two skills seem to overlap, show the boundary. See [disambiguation.md](references/disambiguation.md).
- **Configure** — write a `## Telegram bot rules` block into the project's `CLAUDE.md`/`AGENTS.md`. Follow [project-config.md](references/project-config.md).

## Skill loading

| Intent | Primary | Also load |
| --- | --- | --- |
| New bot / entry point / commands / routing / middleware | [telegram-bot-basics](../telegram-bot-basics/SKILL.md) | testing if adding behavior |
| Buttons / inline & reply keyboards / menus / pagination | [telegram-bot-ui](../telegram-bot-ui/SKILL.md) | basics for callback routing |
| Persist per-user/chat state | [telegram-bot-sessions](../telegram-bot-sessions/SKILL.md) | scaling if multi-instance |
| Multi-step dialog / wizard / ask-and-wait | [telegram-bot-conversations](../telegram-bot-conversations/SKILL.md) | ui if buttons drive steps |
| Formatting / media / edit / delete / file downloads | [telegram-bot-messages](../telegram-bot-messages/SKILL.md) | — |
| Concurrency / rate limits / 429 / spam | [telegram-bot-scaling](../telegram-bot-scaling/SKILL.md) | sessions (sequentialize key) |
| Go to production / webhook vs polling / hosting | [telegram-bot-deploy](../telegram-bot-deploy/SKILL.md) | security (webhook secret), sessions (Redis) |
| Charge money / Stars / invoices | [telegram-bot-payments](../telegram-bot-payments/SKILL.md) | — |
| Mini App / Web App / initData / inline mode | [telegram-bot-mini-apps](../telegram-bot-mini-apps/SKILL.md) | security (validate initData) |
| Write tests | [telegram-bot-testing](../telegram-bot-testing/SKILL.md) | the domain skill under test |
| Harden / authz / token / input validation | [telegram-bot-security](../telegram-bot-security/SKILL.md) | deploy (webhook secret) |

## Cold-start decision tree

```
Handlers, routing, entry point, project layout?   → telegram-bot-basics
Buttons / keyboards / menus / pagination?          → telegram-bot-ui (+ basics for callbacks)
Multi-step ask→wait→branch dialog?                 → telegram-bot-conversations
Persisting state across messages?                  → telegram-bot-sessions
Formatting / media / edit / download?              → telegram-bot-messages
Concurrency / rate limit / 429 / spam?             → telegram-bot-scaling
Webhook vs polling / hosting / serverless?         → telegram-bot-deploy
Invoices / Telegram Stars?                          → telegram-bot-payments
Mini App / Web App / initData / inline mode?       → telegram-bot-mini-apps
Tests (no real token)?                              → telegram-bot-testing
Token / authz / input validation / abuse?          → telegram-bot-security
```

## Categories at a glance

Full catalog with "use when" hooks: [by-category.md](references/by-category.md).

| Category | Skills |
| --- | --- |
| Core | `telegram-bot-basics` |
| UI & messages | `telegram-bot-ui` `telegram-bot-messages` |
| State & flows | `telegram-bot-sessions` `telegram-bot-conversations` |
| Production | `telegram-bot-deploy` `telegram-bot-scaling` `telegram-bot-security` |
| Advanced | `telegram-bot-payments` `telegram-bot-mini-apps` |
| Testing | `telegram-bot-testing` |

## Competing clusters — boundary lines

Full tables with routing examples: [disambiguation.md](references/disambiguation.md). Key clusters:

- **State**: `telegram-bot-sessions` (long-lived per-key state) vs `telegram-bot-conversations` (linear ask→wait flows). 1–2 steps → a session `step`; longer → conversations.
- **UI**: `telegram-bot-ui` (build keyboards, menus) vs `telegram-bot-basics` (register & answer `callbackQuery`).
- **Deploy**: `telegram-bot-deploy` (webhook/polling/hosting) vs `telegram-bot-scaling` (concurrency, rate limits) — go to production vs handle load.

## Universal rules (every bot task)

Apply regardless of which skill loads (full set: [../../RULES.md](../../RULES.md)):

1. **`await` every API call** — `ctx.reply`, `editMessageText`, `bot.api.*`.
2. **`answerCallbackQuery()` in every callback handler** — or the client spins.
3. **Never commit `BOT_TOKEN`** — read `process.env.BOT_TOKEN`; keep it off logs and replies.
4. **One consumer per token** — no two pollers, no poller + webhook (`409 Conflict`).
5. **Install `bot.catch`** — one unhandled throw stops the bot.
6. **Split build from run** — a factory returns the wired bot; `bot.start()` lives in the entry file.
7. **Validate untrusted input** — `callback_data`, deep-link payloads, and Mini App `initData` are attacker-controlled.

## Configure mode

When invoked as `/telegram-how-to configure`, write the project rules block per [project-config.md](references/project-config.md).

---

Read the linked `SKILL.md` files for implementation detail — do not improvise patterns that contradict them.
