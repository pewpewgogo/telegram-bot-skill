# Competing clusters — Telegram bot skills

Boundary tables for when two skills (or patterns) look alike. Route to the **owner** skill; use the other as secondary only when the table says so.

---

## 1. Session state cluster

| Approach | Owner skill | Use when |
| --- | --- | --- |
| `ctx.session` + grammY `session()` plugin | `telegram-bot-sessions` | Default for agntdev bots; toolkit wires via `createBot()` |
| Manual `Map<chatId, State>` | — (avoid) | Only for throwaway prototypes; not harness-friendly |
| SQLite / custom DB per message | `telegram-bot-sessions` | Post-MVP or preview patterns documented in skill |
| grammY **conversations** plugin | — (not MVP) | Multi-step plugin with its own middleware; not in bot-starter template — prefer sessions skill patterns |

**Routing examples:**

- "Remember which step the user is on" → `telegram-bot-sessions`.
- "User sent text outside the expected step" → `telegram-bot-sessions` (guard + reset) + maybe `telegram-bot-ui` (offer menu).
- "Session lost after container restart" → `telegram-bot-deploy` (Redis) + `telegram-bot-sessions`.

---

## 2. UI cluster

| Topic | Owner | Secondary |
| --- | --- | --- |
| Building keyboard JSON / toolkit builders | `telegram-bot-ui` | — |
| Registering `bot.callbackQuery("prefix:", …)` handlers | `telegram-bot-basics` | `telegram-bot-ui` for payload format |
| Pagination / confirm / menu layout | `telegram-bot-ui` | `telegram-bot-sessions` if page state in session |
| `editMessageText` vs new message | `telegram-bot-ui` | — |

**Routing examples:**

- "Add Yes/No buttons under the booking summary" → `telegram-bot-ui` (`confirmKeyboard`).
- "Callback spinner never stops" → `telegram-bot-basics` (`answerCallbackQuery`) — not a UI-builder issue.
- "Inline vs reply keyboard?" → `telegram-bot-ui` (inline = callbacks on message; reply = sends text).

---

## 3. Testing cluster

| Topic | Owner | Secondary |
| --- | --- | --- |
| BotSpec JSON, coverage, harness gate | `telegram-test-specs` | `telegram-bot-basics` for `makeBot()` |
| Mock fetch/DB, 429, blocked user, payments | `telegram-test-advanced` | `telegram-test-specs` for happy-path specs |
| "Do I need a real bot token?" | `telegram-test-specs` | No — tokenless harness |

**Routing examples:**

- "Add test for /start" → `telegram-test-specs` only.
- "Test that API 429 shows retry message" → `telegram-test-advanced` + keep existing specs green.
- "Harness says command not covered" → `telegram-test-specs` (`commands.json` + spec file).

---

## 4. Basics vs deploy

| Symptom | Owner | Secondary |
| --- | --- | --- |
| Wrong handler logic, bad routing | `telegram-bot-basics` | — |
| `no bot entry point found` | `telegram-bot-deploy` | `telegram-bot-basics` |
| Build passes locally, container crash loop | `telegram-bot-deploy` | `telegram-bot-sessions` if Redis/session |
| Missing `@agntdev/bot-toolkit` in Docker build | `telegram-bot-deploy` | — |

---

## 5. Pipeline vs implementation

| Topic | Owner | Secondary |
| --- | --- | --- |
| `agnt ready`, claim, PR, payouts | `agnt-cli-builder` | Domain skill for the task code |
| How to implement the feature | Domain skill | `agnt-cli-builder` only at session start |

**Rule:** `agnt-cli-builder` On Activation runs CLI commands — load it when the user is a **builder on agntdev**, not for generic grammY bots outside the pipeline.

---

## 6. Three-layer teaching (all domain skills)

Every `telegram-bot-*` skill teaches in order:

1. **Bot API** — raw HTTP / JSON shapes
2. **grammY** — `ctx`, plugins, routing
3. **@agntdev/bot-toolkit** — `createBot`, harness-ready defaults

If the agent jumps straight to toolkit helpers without understanding grammY routing, load `telegram-bot-basics` as secondary even when the primary is `telegram-bot-ui` or `telegram-bot-sessions`.