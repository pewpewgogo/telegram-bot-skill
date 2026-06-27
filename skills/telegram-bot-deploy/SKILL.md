---
name: telegram-bot-deploy
description: >
  Use when taking a grammY Telegram bot to production — choosing long polling vs
  webhooks, wiring webhookCallback to a server or serverless runtime, setWebhook with
  a secret token, graceful shutdown, and process/health basics. Generic hosting; for
  the agnt-gm platform contract see agntdev-deploy.
  Triggers: deploy telegram bot, production, webhook vs polling, webhookCallback,
  setWebhook, secret token, serverless bot, cloudflare workers, aws lambda, express
  webhook, graceful shutdown, host bot, 409 conflict, run in production.
compatibility: grammY v1 (`grammy`, `grammy/web` for edge). Any Node/Deno host or serverless runtime.
license: MIT
---

# telegram-bot-deploy

Getting a working bot ([telegram-bot-basics](../telegram-bot-basics/SKILL.md)) onto a server.
The build/run split there is what makes this clean. Platform-specific (agnt-gm) deploy →
[agntdev-deploy](../agntdev-deploy/SKILL.md).

## Polling or webhook?

| | Long polling | Webhook |
|---|---|---|
| Setup | none — `bot.start()` | public HTTPS URL + `setWebhook` |
| Hosts | always-on VPS/container | servers **and** serverless/edge |
| Scaling | one process per token | many instances behind a load balancer |
| Cost | a process running 24/7 | pay-per-request possible |

**Default to polling** unless you need serverless or horizontal scale. **One consumer per
token** — polling and a webhook can't both be active (you'll get `409 Conflict`); call
`bot.api.deleteWebhook()` before switching to polling.

## Long polling in production

```ts
const bot = buildBot(process.env.BOT_TOKEN!);   // build/run split from telegram-bot-basics
// graceful shutdown so in-flight updates finish and the poll loop releases the token:
for (const sig of ["SIGINT", "SIGTERM"]) process.once(sig, () => bot.stop());
bot.start({ onStart: (info) => console.log("running as", info.username) });
```

For concurrency use the runner instead of `bot.start()` → [telegram-bot-scaling](../telegram-bot-scaling/SKILL.md).

## Webhooks

`webhookCallback(bot, adapter)` turns the bot into a request handler. **Don't call
`bot.start()`** in webhook mode.

```ts
import { webhookCallback } from "grammy";
import express from "express";

const app = express();
app.use(express.json());
app.use(`/${SECRET_PATH}`, webhookCallback(bot, "express"));
app.listen(8080);
```

Adapters: `"express"`, `"fastify"`, `"hono"`, `"http"`/`"std/http"`, `"cloudflare"`,
`"aws-lambda"`, … (full list in grammY docs).

### Register the webhook + secret token

```ts
await bot.api.setWebhook(`https://your.host/${SECRET_PATH}`, {
  secret_token: process.env.WEBHOOK_SECRET,           // Telegram echoes it in a header
  drop_pending_updates: true,
});
```

grammY verifies the `X-Telegram-Bot-Api-Secret-Token` header automatically when you pass the
matching `secretToken` option to `webhookCallback`. Without it, **anyone who learns your URL
can POST fake updates** → [telegram-bot-security](../telegram-bot-security/SKILL.md).

### Serverless / edge

Import from `grammy/web` on edge runtimes and use `bot.init()` (no `botInfo` autofetch in some
runtimes):

```ts
import { Bot, webhookCallback } from "grammy/web";   // Cloudflare Workers
export default { fetch: webhookCallback(bot, "cloudflare-mod") };
```

On serverless, state **cannot** live in memory between invocations — use an external store
(Redis/KV) for [sessions](../telegram-bot-sessions/SKILL.md).

## Operational basics

- **Token from env**, never the image/repo → [telegram-bot-security](../telegram-bot-security/SKILL.md).
- **Restart on crash** (systemd/Docker `restart: always`); let fatal errors crash so the supervisor restarts you — don't zombie.
- **Persistent storage** for sessions (Redis), since restarts wipe memory.
- **Health**: for polling, the process *is* the liveness signal; for webhooks, your HTTP server is.

## Common mistakes

1. **Polling + webhook on one token** — `409 Conflict`. `deleteWebhook()` before polling; don't `bot.start()` in webhook mode.
2. **No `secret_token`** — an exposed webhook URL accepts forged updates. Always set it and let grammY verify the header.
3. **In-memory state on serverless/multi-instance** — invocations/instances don't share memory. Use Redis/KV.
4. **No graceful shutdown** — killing mid-update loses work and may hold the token. Handle `SIGTERM`/`SIGINT` with `bot.stop()`.
5. **`bot.start()` in webhook mode** — double-consumes updates. Webhook handler only.
6. **Swallowing fatal errors to "stay up"** — a zombied process won't be restarted. Crash loudly; let the supervisor recycle it.
7. **Forgetting `bot.init()`/`grammy/web` on edge** — core import or missing init fails on Workers/Deno Deploy.

## Quick reference

```ts
bot.start({ onStart })                       // long polling
bot.stop()                                   // graceful — on SIGINT/SIGTERM
webhookCallback(bot, "express" | "hono" | "cloudflare" | "aws-lambda" | …, { secretToken })
bot.api.setWebhook(url, { secret_token, drop_pending_updates })
bot.api.deleteWebhook({ drop_pending_updates })   // before switching to polling
import { Bot, webhookCallback } from "grammy/web"  // edge runtimes
```
