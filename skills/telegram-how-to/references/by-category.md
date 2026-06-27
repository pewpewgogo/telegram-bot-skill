# Telegram bot skills — by category

Short identifiers match directory names under `skills/`. Load paths are sibling `../<name>/SKILL.md` from this orchestrator.

---

## Core

| Skill | Use when |
| --- | --- |
| `telegram-bot-basics` | Bot API mental model, `new Bot()`, command/`hears`/filter-query routing, middleware & `Composer`, `bot.catch`, `setMyCommands`, build/run split, project structure |

## UI & messages

| Skill | Use when |
| --- | --- |
| `telegram-bot-ui` | `InlineKeyboard`/`Keyboard`, callback routing & `answerCallbackQuery`, `@grammyjs/menu`, pagination, confirm dialogs, edit vs new message |
| `telegram-bot-messages` | HTML/MarkdownV2 + `@grammyjs/parse-mode` `fmt`, entities, edit/delete, media (`InputFile`, photo/doc), `file_id` reuse, downloads (`@grammyjs/files`) |

## State & flows

| Skill | Use when |
| --- | --- |
| `telegram-bot-sessions` | `ctx.session`, session key/scope, storage adapters (memory/Redis/file/free), `lazySession`, shape design, migrations |
| `telegram-bot-conversations` | `@grammyjs/conversations` multi-step dialogs, `wait`/`waitFor`/forms, the replay model + `conversation.external()` |

## Production

| Skill | Use when |
| --- | --- |
| `telegram-bot-deploy` | polling vs webhook, `webhookCallback` adapters, `setWebhook` + secret token, serverless/edge, graceful shutdown |
| `telegram-bot-scaling` | `@grammyjs/runner` `run()`, `sequentialize`, `apiThrottler`/`auto-retry` for 429, `@grammyjs/ratelimiter` |
| `telegram-bot-security` | token hygiene, webhook secret, input validation, authz by `ctx.from.id`, abuse/flood limits |

## Advanced

| Skill | Use when |
| --- | --- |
| `telegram-bot-payments` | `sendInvoice`, Telegram Stars (`XTR`), `pre_checkout_query`, `successful_payment`, refunds |
| `telegram-bot-mini-apps` | Web App buttons, `web_app_data`, **initData HMAC validation**, inline mode |

## Testing

| Skill | Use when |
| --- | --- |
| `telegram-bot-testing` | transformer to capture API calls, synthetic `Update` + `bot.handleUpdate`, vitest/jest |

---

## Typical multi-skill combos

| Task | Load together |
| --- | --- |
| Booking flow with date-picker buttons | `telegram-bot-conversations` + `telegram-bot-ui` |
| New `/settings` command with inline menu | `telegram-bot-basics` + `telegram-bot-ui` |
| Sell a digital plan for Stars | `telegram-bot-payments` + `telegram-bot-sessions` |
| Mini App with authenticated backend | `telegram-bot-mini-apps` + `telegram-bot-security` |
| High-traffic broadcast bot | `telegram-bot-scaling` + `telegram-bot-deploy` + `telegram-bot-sessions` |
| Bot exits immediately in production | `telegram-bot-deploy` + `telegram-bot-basics` |
