---
name: telegram-bot-mini-apps
description: >
  Use when a Telegram bot pairs with a Web App / Mini App (TMA) or answers inline
  queries — opening a web app from a button, receiving web_app_data, and the
  CRITICAL server-side validation of Telegram initData (HMAC-SHA256) before trusting it.
  Triggers: mini app, TMA, web app, WebApp, initData, validate initData, web_app_data,
  webApp button, openTelegramLink, inline mode, inline query, answerInlineQuery,
  telegram web app auth.
compatibility: grammY v1 (`grammy`) for the bot side; the Mini App is any web frontend. initData validation is server-side crypto (Node `crypto`).
license: MIT
---

# telegram-bot-mini-apps

A **Mini App** is a web page opened inside Telegram. The bot opens it and may receive data
back; the web app authenticates its user via **initData** that your backend must verify.
Buttons → [telegram-bot-ui](../telegram-bot-ui/SKILL.md).

## Open a Mini App

```ts
import { InlineKeyboard } from "grammy";
const kb = new InlineKeyboard().webApp("Open app", "https://your.app");
await ctx.reply("Launch:", { reply_markup: kb });
```

Also openable from a reply-keyboard `Keyboard().webApp(...)`, the chat menu button
(`setChatMenuButton`), or an inline-mode result.

## Receiving data back

A Web App can `Telegram.WebApp.sendData(str)` (only from a reply-keyboard web app), which
arrives as:

```ts
bot.on("message:web_app_data", (ctx) => {
  const data = ctx.msg.web_app_data.data;   // arbitrary string your app sent
});
```

For richer flows the app usually calls **your own HTTPS backend** directly (not via the bot),
sending `initData` for auth.

## ⚠️ Validate initData — the security core

`window.Telegram.WebApp.initData` is a query string the app sends to your backend. **Never
trust it unvalidated** — anyone can forge a user id otherwise. Verify the HMAC, server-side:

```ts
import { createHmac } from "node:crypto";

function checkInitData(initData: string, botToken: string): boolean {
  const p = new URLSearchParams(initData);
  const hash = p.get("hash"); p.delete("hash");
  const dataCheck = [...p.entries()].map(([k, v]) => `${k}=${v}`).sort().join("\n");
  const secret = createHmac("sha256", "WebAppData").update(botToken).digest();
  const calc = createHmac("sha256", secret).update(dataCheck).digest("hex");
  return calc === hash;                       // also check auth_date freshness (e.g. < 24h)
}
```

Key points: secret key is `HMAC_SHA256("WebAppData", botToken)`; data-check string is the
sorted `key=value` pairs joined by `\n`, excluding `hash`. Reject stale `auth_date`. Libraries
like `@telegram-apps/init-data-node` do this for you — prefer them over hand-rolling.

## Inline mode (bonus)

Answer `@yourbot query` from any chat:

```ts
import { InlineQueryResultBuilder } from "grammy";
bot.inlineQuery(/.*/, async (ctx) => {
  const results = [InlineQueryResultBuilder.article("1", "Send hi").text("hi")];
  await ctx.answerInlineQuery(results, { cache_time: 0 });
});
```

Enable inline mode in @BotFather first.

## Common mistakes

1. **Trusting initData without HMAC check** — the entire auth model is the signature. Validate every request server-side; never trust client-sent user ids.
2. **Validating on the client** — meaningless; do it on your backend with the bot token (which must stay server-side).
3. **Ignoring `auth_date`** — without a freshness window, a captured initData replays forever. Reject old ones.
4. **Expecting `web_app_data` from inline web apps** — `sendData` only fires from a *reply-keyboard* web app; inline/menu apps talk to your backend instead.
5. **Inline mode not enabled** — `answerInlineQuery` does nothing until you turn on inline mode in @BotFather.
6. **Putting the bot token in the frontend** — it's needed only to *verify* initData, on the server. Never ship it to the browser.

## Quick reference

```ts
new InlineKeyboard().webApp("label", "https://app")     // open Mini App
bot.on("message:web_app_data", ctx => ctx.msg.web_app_data.data)
// server: secret = HMAC_SHA256("WebAppData", botToken); hash over sorted key=value\n…; check auth_date
bot.inlineQuery(/re/, ctx => ctx.answerInlineQuery([InlineQueryResultBuilder.article(id, title).text(t)]))
```
