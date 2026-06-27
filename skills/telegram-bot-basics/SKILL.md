---
name: telegram-bot-basics
description: >
  Use when starting or structuring a Telegram bot with grammY (TypeScript/JS) —
  entry point, command and message routing, the context object, middleware, and the
  global error boundary. The foundation every other telegram-bot-* skill builds on.
  Triggers: build telegram bot, create telegram bot, grammY bot, bot entry point,
  bot.command, bot.on, filter query, ctx.reply, middleware, bot.catch, long polling,
  setMyCommands, new Bot.
compatibility: grammY v1 (npm `grammy`), Node 18+ or Deno. Framework-agnostic.
license: MIT
---

# telegram-bot-basics

How a bot receives updates, how grammY routes them, and how to structure the entry point.

> Keyboards/callbacks → [telegram-bot-ui](../telegram-bot-ui/SKILL.md) ·
> per-user state → [telegram-bot-sessions](../telegram-bot-sessions/SKILL.md) ·
> dialogs → [telegram-bot-conversations](../telegram-bot-conversations/SKILL.md) ·
> production → [telegram-bot-deploy](../telegram-bot-deploy/SKILL.md).

## Mental model

A bot is an **HTTP client** of `api.telegram.org/bot<TOKEN>/<METHOD>`. Telegram keeps no
per-user state: each event is an `Update` you receive, each action is one API call. Two
receive modes (chosen at deploy, not in handlers — routing code is identical):

| Mode | grammY | Use when |
|---|---|---|
| **Long polling** | `bot.start()` | dev, single instance, no public URL (default) |
| **Webhook** | `webhookCallback()` | serverless, autoscale → [telegram-bot-deploy](../telegram-bot-deploy/SKILL.md) |

Only **one consumer per token** — a second poller (or poller + webhook) gets `409 Conflict`.

## Bot instance

```ts
import { Bot } from "grammy";

const bot = new Bot(process.env.BOT_TOKEN!); // from @BotFather; never hardcode
bot.command("start", (ctx) => ctx.reply("Hello!"));
bot.start();
```

## Context (`ctx`)

One arg per handler — the `Update` plus chat-bound shortcuts:

```ts
ctx.msg        // the message for ANY update type (use over ctx.message)
ctx.from       // User · ctx.chat / ctx.chatId — Chat
ctx.match      // capture from command arg / hears regex / callbackQuery regex
await ctx.reply("text");                      // respond in this chat
await ctx.api.sendMessage(otherChatId, "hi"); // raw API to any chat / any method
```

## Routing

```ts
bot.command("start", (ctx) => ctx.reply("hi"));      // /start (+ /start@Bot auto-handled)
bot.command(["help", "h"], (ctx) => ctx.reply("…")); // aliases
bot.command("echo", (ctx) => ctx.reply(ctx.match));  // "/echo x" → ctx.match === "x"
bot.hears(/^buy (\d+)/, (ctx) => ctx.reply(ctx.match[1])); // text/caption; regex → ctx.match
bot.on("message:text", (ctx) => ctx.reply(ctx.msg.text));  // filter query (narrows ctx)
bot.on("message:photo", (ctx) => ctx.reply("nice"));
bot.callbackQuery("data", (ctx) => ctx.answerCallbackQuery()); // inline button → telegram-bot-ui
```

**Filter queries** are typed `update:sub:detail` strings (`message:entities:url`,
`edited_message`, …); editor autocompletes them and `ctx` narrows inside the handler.
Handlers run **top-to-bottom** — register specific (commands) before catch-alls (`bot.on("message")`).

## Middleware

```ts
bot.use(async (ctx, next) => { console.log(ctx.update.update_id); await next(); }); // log + continue
bot.use(async (ctx, next) => { if (ctx.from?.id !== ADMIN) return; await next(); }); // guard: no next() = stop
```

Group features with `Composer` and mount in order — keeps `index.ts` small:

```ts
import { Composer } from "grammy";
export const admin = new Composer();
admin.command("ban", (ctx) => {/* … */});
// index.ts: bot.use(admin);
```

## Error boundary

An unhandled throw stops the bot. Always install:

```ts
import { GrammyError, HttpError } from "grammy";
bot.catch(({ ctx, error }) => {
  if (error instanceof GrammyError) console.error("API:", error.description); // Telegram rejected (.error_code)
  else if (error instanceof HttpError) console.error("network:", error);      // never reached Telegram
  else console.error(error);
});
```

Recover within a sub-tree with `bot.errorBoundary(handler, ...mw)`. 429/retry strategy →
[telegram-bot-scaling](../telegram-bot-scaling/SKILL.md).

## Structure & custom context

Split **build** from **run** so tests and webhook deploys can import a wired bot without polling:

```ts
// src/bot.ts
import { Bot, Context, SessionFlavor } from "grammy";
export type MyContext = Context & SessionFlavor<{ count: number }>; // plugins extend ctx via flavors
export function buildBot(token: string) {
  const bot = new Bot<MyContext>(token);
  bot.catch((err) => console.error(err));
  bot.use(/* composers */);
  return bot;                       // NO bot.start() here
}
// src/index.ts
buildBot(process.env.BOT_TOKEN!).start();
```

```
src/  bot.ts (buildBot) · index.ts (run) · commands/*.ts (one Composer each) · types.ts
```

Register the menu once at startup: `await bot.api.setMyCommands([{ command: "start", description: "Start" }])`.

## Common mistakes

1. **Hardcoded token** — read `process.env.BOT_TOKEN`; a leak = full takeover.
2. **Un-awaited API calls** — `ctx.reply(...)` without `await` races the handler and swallows errors. Await every `ctx.*`/`bot.api.*`.
3. **No `bot.catch`** — one throw kills long polling.
4. **Catch-all before specific** — `bot.on("message")` above `bot.command` eats the command.
5. **Two pollers on one token** — `409 Conflict`. One consumer per token.
6. **`ctx.message` for non-messages** — `undefined` on edits/posts/callbacks. Use `ctx.msg`/`ctx.from`/`ctx.chatId`.
7. **`bot.start()` inside the build fn** — un-testable, un-webhookable. Split build from run.

## Quick reference

```ts
new Bot<MyContext>(token)
bot.command("n" | ["a","b"], fn)   // ctx.match = arg string
bot.hears(/re/, fn)                 // ctx.match = RegExpMatch
bot.on("message:text", fn)          // filter query — narrows ctx
bot.callbackQuery("data", fn)       // must answerCallbackQuery → telegram-bot-ui
bot.use(mw | composer) · bot.catch(fn) · bot.api.setMyCommands([...])
ctx.reply / ctx.api.sendMessage · bot.start() · webhookCallback(bot, "express")
```
