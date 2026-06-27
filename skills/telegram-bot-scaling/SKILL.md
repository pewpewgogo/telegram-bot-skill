---
name: telegram-bot-scaling
description: >
  Use when a Telegram bot must handle load — concurrent update processing with
  @grammyjs/runner, sequentialize to avoid per-chat race conditions, API rate-limit
  handling (429/flood) with auto-retry and the throttler, and per-user spam limits.
  Triggers: scale bot, concurrency, grammY runner, run(bot), sequentialize, rate limit,
  429, flood control, retry after, throttler, apiThrottler, auto-retry, ratelimiter,
  slow bot, too many requests, parallel updates.
compatibility: grammY v1; `@grammyjs/runner`, `@grammyjs/auto-retry`, `@grammyjs/transformer-throttler`, `@grammyjs/ratelimiter`.
license: MIT
---

# telegram-bot-scaling

Two different limits to respect: **inbound** (process many updates at once without races) and
**outbound** (don't exceed Telegram's send limits). Error types & `bot.catch` →
[telegram-bot-basics](../telegram-bot-basics/SKILL.md).

## Inbound: concurrent processing with the runner

`bot.start()` processes updates **one at a time**. The runner pulls and handles them
concurrently:

```ts
import { run } from "@grammyjs/runner";
run(bot);                       // replaces bot.start() — concurrent long polling
```

### sequentialize — prevent races

Concurrency means two updates from the **same chat** can run at once and clobber shared state
(e.g. `ctx.session`). Force per-key ordering while keeping different chats parallel:

```ts
import { sequentialize } from "@grammyjs/runner";
bot.use(sequentialize((ctx) => ctx.chat?.id.toString())); // same key = ordered; install FIRST
run(bot);
```

Use the **same key you use for sessions** ([telegram-bot-sessions](../telegram-bot-sessions/SKILL.md)),
so updates that share state are serialized. Install `sequentialize` before session/handlers.

## Outbound: respect Telegram's limits

Telegram throttles sends (~30 msg/s globally; ~1 msg/s per chat; ~20 msg/min per group). Over
the limit returns `429` with `retry_after`.

```ts
// Auto-retry: transparently honors retry_after on 429 / server errors.
import { autoRetry } from "@grammyjs/auto-retry";
bot.api.config.use(autoRetry({ maxRetryAttempts: 3, maxDelaySeconds: 5 }));

// Throttler: queues outgoing calls to stay under the limits (prevents the 429 in the first place).
import { apiThrottler } from "@grammyjs/transformer-throttler";
bot.api.config.use(apiThrottler());
```

Use **both**: throttler smooths the rate, auto-retry catches the occasional overflow. For
broadcasts to many users, send in batches well under 30/s; never tight-loop `sendMessage`.

## Inbound spam control — @grammyjs/ratelimiter

Limit how often a *user* can trigger the bot (abuse protection, distinct from outbound limits):

```ts
import { limit } from "@grammyjs/ratelimiter";
bot.use(limit({ timeFrame: 2000, limit: 3, onLimitExceeded: (ctx) => ctx.reply("Slow down") }));
```

## How far one process scales

A single long-polling process handles thousands of users fine. Horizontal scaling means
**webhooks behind a load balancer** ([telegram-bot-deploy](../telegram-bot-deploy/SKILL.md)) with
**shared storage** (Redis) so instances see the same sessions — file/memory storage breaks
multi-instance.

## Common mistakes

1. **Concurrency without `sequentialize`** — `run(bot)` + shared `ctx.session` races and loses writes. Sequentialize on the session key.
2. **`sequentialize` key ≠ session key** — different keys leave the exact races you meant to prevent. Keep them identical.
3. **No outbound limiting** — broadcast loops hit `429` and drop messages. Add the throttler; batch sends.
4. **Catching 429 by hand** — `autoRetry` already honors `retry_after`. Don't sleep/retry manually.
5. **Multi-instance on file/memory storage** — instances diverge. Use Redis (or a DB) when scaling out.
6. **Still calling `bot.start()` with the runner** — `run(bot)` replaces it; don't do both.
7. **Conflating the two ratelimiters** — `@grammyjs/ratelimiter` throttles inbound *users*; the throttler/auto-retry handle outbound *API* limits. You usually want both.

## Quick reference

```ts
run(bot)                                          // @grammyjs/runner — concurrent (replaces bot.start)
bot.use(sequentialize(ctx => ctx.chat?.id.toString()))  // per-key ordering, install first
bot.api.config.use(autoRetry({ maxRetryAttempts, maxDelaySeconds }))  // honor 429 retry_after
bot.api.config.use(apiThrottler())                // queue outbound under limits
bot.use(limit({ timeFrame, limit, onLimitExceeded }))  // @grammyjs/ratelimiter (inbound)
```
