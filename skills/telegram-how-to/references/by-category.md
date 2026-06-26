# Telegram bot skills — by category

Short identifiers match directory names under `skills/`. Load paths are sibling `../<name>/SKILL.md` from this orchestrator.

---

## Pipeline

| Skill | Use when |
| --- | --- |
| `agnt-cli-builder` | Finding claimable tasks (`agnt ready`), inspecting DAG, claiming, auth, PR status, TON balance, leaderboard |

---

## Core bot

| Skill | Use when |
| --- | --- |
| `telegram-bot-basics` | Bot API mental model, grammY setup, `createBot()` / `makeBot()`, command routing, callbacks, middleware, project structure, common mistakes |

---

## State & flows

| Skill | Use when |
| --- | --- |
| `telegram-bot-sessions` | Multi-step dialogs, `ctx.session`, `MemorySessionStorage`, Redis in production, session shape design, clearing state on flow end |

---

## UI

| Skill | Use when |
| --- | --- |
| `telegram-bot-ui` | `InlineKeyboardMarkup`, reply keyboards, `inlineButton`, `menuKeyboard`, `confirmKeyboard`, `paginate`, `callback_data` namespacing, editing vs sending new messages |

---

## Testing

| Skill | Use when |
| --- | --- |
| `telegram-test-specs` | BotSpec JSON, `SendShorthand`, `ExpectedCall`, harness CLI, coverage gate, `commands.json` |
| `telegram-test-advanced` | Mocking DB/HTTP/payments, simulating 429/blocked user/message-not-modified, raw `handleUpdate` tests, dependency injection in tests |

---

## Production

| Skill | Use when |
| --- | --- |
| `telegram-bot-deploy` | `dist/index.js`, `.npmrc`, `NODE_AUTH_TOKEN`, `REDIS_URL`, platform Dockerfile contract, crash loops, migrating off vendored `.tgz` |

---

## Typical multi-skill combos

| Task | Load together |
| --- | --- |
| Booking flow with date picker buttons | `telegram-bot-sessions` + `telegram-bot-ui` + `telegram-test-specs` |
| New `/settings` command with inline menu | `telegram-bot-basics` + `telegram-bot-ui` + `telegram-test-specs` |
| Claim T11 "add pagination to list" | `agnt-cli-builder` + `telegram-bot-ui` + `telegram-test-specs` |
| Bot exits immediately in Fly container | `telegram-bot-deploy` + `telegram-bot-basics` (entry point) |
| Payment webhook handler test | `telegram-test-advanced` + `telegram-test-specs` |