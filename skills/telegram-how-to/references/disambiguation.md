# Competing clusters — Telegram bot skills

Boundary tables for when two skills look alike. Route to the **owner** skill; use the other as secondary only when the table says so.

---

## 1. State: sessions vs conversations

| Need | Owner | Use when |
| --- | --- | --- |
| Long-lived per-key state (settings, counters, a `step` field) | `telegram-bot-sessions` | state read/written across unrelated updates |
| Linear ask → wait → branch flow | `telegram-bot-conversations` | a guided sequence with several waits |
| Manual `Map<chatId, State>` | — (avoid) | throwaway prototypes only — lost on restart, not multi-instance safe |

**Rule of thumb:** 1–2 steps → a session `step` field is lighter. 3+ steps or loops/branches → conversations. Conversations still need the base plugin and `conversation.external()` for side effects.

**Routing examples:**
- "Remember the user's language" → `telegram-bot-sessions`.
- "Sign-up wizard: name, then email, then confirm" → `telegram-bot-conversations`.
- "State lost after restart" → `telegram-bot-sessions` (Redis adapter) + `telegram-bot-deploy`.

---

## 2. UI: building keyboards vs routing callbacks

| Topic | Owner | Secondary |
| --- | --- | --- |
| Build `InlineKeyboard`/`Keyboard`, menus, pagination layout | `telegram-bot-ui` | — |
| Register `bot.callbackQuery(...)` & `answerCallbackQuery` | `telegram-bot-basics` | `telegram-bot-ui` for `callback_data` format |
| `editMessageText` vs sending a new message | `telegram-bot-ui` | `telegram-bot-messages` for formatting |
| Page state | `telegram-bot-ui` | `telegram-bot-sessions` if persisted |

- "Spinner never stops" → `telegram-bot-basics`/`telegram-bot-ui` (`answerCallbackQuery`), not a builder issue.
- "Inline vs reply keyboard?" → `telegram-bot-ui` (inline → `callback_query`; reply → text messages).

---

## 3. Testing: generic vs the agnt-gm harness

| Topic | Owner | Secondary |
| --- | --- | --- |
| Transformer capture + `handleUpdate` (any project) | `telegram-bot-testing` | the domain skill under test |
| BotSpec JSON, coverage gate, harness CLI (agnt-gm) | `telegram-test-specs` | `telegram-bot-testing` for the underlying technique |
| Mock DB/HTTP, 429/blocked-user, payments (agnt-gm) | `telegram-test-advanced` | `telegram-test-specs` |

Both layers use the same core idea (intercept outgoing API calls, feed synthetic updates); `telegram-bot-testing` is the framework-agnostic version, the `telegram-test-*` skills add the platform's spec format and gate.

---

## 4. Deploy: generic vs agnt-gm platform

| Symptom | Owner | Secondary |
| --- | --- | --- |
| Webhook vs polling, hosting, serverless, graceful shutdown | `telegram-bot-deploy` | `telegram-bot-security` (webhook secret) |
| `409 Conflict` (two consumers) | `telegram-bot-deploy` | — |
| `dist/index.js` entry, `.npmrc`, `REDIS_URL`, container crash on agnt-gm | `agntdev-deploy` | `telegram-bot-deploy` for the generic concept |
| Scaling out to many instances | `telegram-bot-scaling` | `telegram-bot-deploy` (webhooks), `telegram-bot-sessions` (shared store) |

---

## 5. Basics vs deploy

| Symptom | Owner |
| --- | --- |
| Wrong handler logic / routing order | `telegram-bot-basics` |
| Builds locally, crashes in prod | `telegram-bot-deploy` (or `agntdev-deploy` on platform) |
| No error boundary, bot dies on a throw | `telegram-bot-basics` (`bot.catch`) |

---

## 6. Pipeline vs implementation (agnt-gm only)

| Topic | Owner | Secondary |
| --- | --- | --- |
| `agnt ready`, claim, PR, payouts | `agnt-cli-builder` | the general skill for the code |
| How to implement the feature | the general skill | `agnt-cli-builder` at session start only |

**Rule:** load the agnt-gm tier only when the user is a builder on that platform. For generic grammY bots, stay in the general core.
