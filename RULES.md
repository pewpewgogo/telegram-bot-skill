# Telegram bot rules (grammY)

The cross-cutting RULE set for building Telegram bots with grammY — the rules that
hold regardless of which feature you're working on. Each domain skill under `skills/`
carries its own `## Common mistakes`; this file is the consolidated, project-level
version. Inject a condensed copy into a repo's `CLAUDE.md`/`AGENTS.md` with
`/telegram-how-to configure`.

> Format: **Rule** — why it bites — *owning skill*.

## Correctness

1. **`await` every API call.** `ctx.reply`, `editMessageText`, `answerCallbackQuery`,
   `bot.api.*` are all async; an un-awaited call races the handler's return and silently
   swallows errors. — *telegram-bot-basics*
2. **`answerCallbackQuery()` in every callback handler.** Until you answer, the client
   shows a loading spinner for ~30s. Answer even when there's nothing to say. — *telegram-bot-ui*
3. **Install `bot.catch`.** One unhandled throw stops long polling. Distinguish
   `GrammyError` (Telegram rejected) from `HttpError` (never reached Telegram). — *telegram-bot-basics*
4. **Register specific handlers before catch-alls.** Handlers run top-to-bottom; a
   `bot.on("message")` above `bot.command("start")` eats the command. — *telegram-bot-basics*
5. **Use `ctx.msg`/`ctx.from`/`ctx.chatId`, not `ctx.message`.** The latter is `undefined`
   for edits, channel posts, and callback queries. — *telegram-bot-basics*

## Architecture

6. **Split build from run.** A factory (`createBot`) returns the wired bot; `bot.start()`
   lives only in the entry file. This is what tests feed updates to and what webhook
   deploys import. — *telegram-bot-basics, telegram-bot-testing*
7. **One consumer per token.** Two pollers — or a poller plus a webhook — on the same
   token give `409 Conflict`. `deleteWebhook()` before switching to polling. — *telegram-bot-deploy*
8. **Persistent, shared storage in production.** Default in-memory sessions vanish on
   restart and diverge across instances. Use Redis (or a DB) for any multi-instance or
   serverless deploy. — *telegram-bot-sessions, telegram-bot-scaling*
9. **Group features into `Composer`s.** Keep the entry file small; mount modules in order. — *telegram-bot-basics*

## State & flows

10. **`initial()` returns a fresh object.** A shared reference leaks state across users. — *telegram-bot-sessions*
11. **Set the session key deliberately.** Per-chat vs per-user changes who owns the state;
    the default is per-chat. — *telegram-bot-sessions*
12. **Pick sessions vs conversations by length.** 1–2 steps → a session `step` field;
    longer/branching → the conversations plugin. — *telegram-bot-sessions, telegram-bot-conversations*
13. **Wrap side effects in `conversation.external()`.** Conversation functions replay from
    the top each update; raw `fetch`/DB/`Date.now()`/random re-run and corrupt the flow. — *telegram-bot-conversations*

## Messages & UI

14. **`callback_data` ≤ 64 bytes.** Telegram rejects longer; store real state in a session
    and keep an id in the button. — *telegram-bot-ui*
15. **Prefer HTML or `@grammyjs/parse-mode` `fmt` over hand-written MarkdownV2.** One
    unescaped `.`/`-`/`!` 400s the send. — *telegram-bot-messages*
16. **Edit, don't re-send, for updates;** but never edit to identical content
    (`message is not modified`). — *telegram-bot-ui, telegram-bot-messages*
17. **Reuse `file_id`** instead of re-uploading the same media. — *telegram-bot-messages*

## Scale

18. **`sequentialize` on the same key as sessions** when using the runner, or concurrent
    updates from one chat race shared state. — *telegram-bot-scaling*
19. **Respect outbound limits.** Use `apiThrottler` to stay under (~30 msg/s, ~1/s per
    chat) and `autoRetry` to honor `429 retry_after`; batch broadcasts. — *telegram-bot-scaling*

## Security (everything inbound is untrusted)

20. **`BOT_TOKEN` from env only.** Never commit, log, or echo it; rotate via @BotFather if
    exposed. — *telegram-bot-security*
21. **Set a webhook `secret_token`.** Without it, anyone who learns the URL can POST forged
    updates. — *telegram-bot-deploy, telegram-bot-security*
22. **Validate Mini App `initData` server-side** (HMAC + `auth_date` freshness) before
    trusting any user identity. — *telegram-bot-mini-apps, telegram-bot-security*
23. **Authorize by `ctx.from.id`, not username.** Usernames are mutable/spoofable. — *telegram-bot-security*
24. **Validate `callback_data` and deep-link payloads;** never interpolate user text into
    SQL/shell/paths; escape user text under a `parse_mode`. — *telegram-bot-security*

## Payments

25. **Answer `pre_checkout_query` within ~10s,** or the charge fails. — *telegram-bot-payments*
26. **Fulfill idempotently and keep `telegram_payment_charge_id`** (retries can re-deliver;
    refunds need the id). Derive what was bought from your `invoice_payload`, not the client. — *telegram-bot-payments*

---

See `skills/telegram-how-to/SKILL.md` for routing and `skills/<name>/SKILL.md` for the
detail behind each rule.
