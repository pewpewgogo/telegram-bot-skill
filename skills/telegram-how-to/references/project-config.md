# Configure mode — force-trigger Telegram bot skills in a project

Adds a `## Telegram bot rules` block to the project's agent config so the bot conventions and the right skills always load. This is how the repo's RULE set ([../../../RULES.md](../../../RULES.md)) gets injected into a consuming project.

## When to use

- A bot repo should always enforce the cross-cutting rules (await calls, answer callbacks, token hygiene).
- The team wants consistent skill routing without retyping triggers.
- Agents keep missing rules without explicit reinforcement.

## Step 1 — Detect the project config file

Check in precedence order (use `Glob` at the project root; update all that apply; if none, ask which to create):

```
1. AGENTS.md
2. CLAUDE.md
3. .cursor/rules/  (Cursor)
4. .github/copilot-instructions.md  (Copilot)
```

## Step 2 — Idempotency check

```bash
grep -n "## Telegram bot rules" AGENTS.md CLAUDE.md 2>/dev/null
```

If present, confirm whether to replace the block or skip.

## Step 3 — Pick the skill set

**Recommended default (any grammY bot):**

```
telegram-how-to
telegram-bot-basics
```

**Add based on signals in the codebase:**

| Signal | Add skill |
| --- | --- |
| `InlineKeyboard` / `callback_data` / `reply_markup` | `telegram-bot-ui` |
| `ctx.session` / `SessionFlavor` | `telegram-bot-sessions` |
| `@grammyjs/conversations` / multi-step input | `telegram-bot-conversations` |
| `parse_mode` / `InputFile` / media | `telegram-bot-messages` |
| `@grammyjs/runner` / rate limits / 429 | `telegram-bot-scaling` |
| `webhookCallback` / `setWebhook` / hosting | `telegram-bot-deploy` |
| `sendInvoice` / `XTR` / Stars | `telegram-bot-payments` |
| `web_app` / `initData` / inline mode | `telegram-bot-mini-apps` |
| any test files | `telegram-bot-testing` |
| auth / token / untrusted input handling | `telegram-bot-security` |
| `agnt` CLI / BotSpec / `dist/index.js` on agnt-gm | `agnt-cli-builder`, `telegram-test-specs`, `agntdev-deploy` |

Remind the user: each always-loaded skill adds ~80–150 description tokens to every session.

## Step 4 — Write the block

```markdown
## Telegram bot rules

Load these skills at the start of every bot task in this repo:

- `telegram-how-to` — route to the right domain skill(s)
- `telegram-bot-basics` — entry point, routing, error boundary
- <add the skills selected in Step 3>

### Non-negotiables

- `await` every API call (`ctx.reply`, `editMessageText`, `bot.api.*`)
- `answerCallbackQuery()` in every callback handler
- Never commit `BOT_TOKEN`; read `process.env.BOT_TOKEN`; keep it out of logs/replies
- One consumer per token (no double poller, no poller + webhook)
- Install `bot.catch`; split build (`createBot`) from run (`bot.start()`)
- Validate untrusted input: `callback_data`, deep-link payloads, Mini App `initData`
```

Adjust the skill list to Step 3. The full rationale for each rule is in [../../../RULES.md](../../../RULES.md).

### Insertion point

- Empty file → block at top.
- Existing content → append after the last section with a blank line.
- Existing `## Telegram bot rules` → replace the block.

## Step 5 — Confirm to the user

Summarize which file(s) were updated, which skills are now always-loaded, the approximate token cost, and point to `telegram-how-to` for per-task routing.
