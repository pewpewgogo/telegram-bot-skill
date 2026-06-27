---
name: telegram-bot-ui
description: >
  Use when building Telegram bot UI with grammY — inline keyboards (callback
  buttons), reply keyboards, url/webApp/switchInline buttons, callback-query
  routing, the @grammyjs/menu plugin, pagination, and confirm dialogs.
  Triggers: inline keyboard, reply keyboard, InlineKeyboard, Keyboard, callback
  button, callback_data, answerCallbackQuery, menu, pagination, confirm dialog,
  bot.callbackQuery, web app button, edit message.
compatibility: grammY v1 (`grammy`); menus need `@grammyjs/menu`. Node 18+ or Deno.
license: MIT
---

# telegram-bot-ui

Buttons, keyboards, and how to route the presses. Routing/`ctx` basics →
[telegram-bot-basics](../telegram-bot-basics/SKILL.md).

## Two keyboard kinds

| Kind | Builder | Where it shows | Press produces |
|---|---|---|---|
| **Inline** | `InlineKeyboard` | attached under the message | a `callback_query` (or opens URL/web app) |
| **Reply** | `Keyboard` | replaces the user's keyboard | a normal text message |

```ts
import { InlineKeyboard, Keyboard } from "grammy";

const inline = new InlineKeyboard()
  .text("Like", "like").text("Share", "share").row() // callback buttons
  .url("Docs", "https://grammy.dev")                 // opens URL, no callback
  .webApp("Open app", "https://example.com/app");    // Mini App → telegram-bot-mini-apps
await ctx.reply("Pick:", { reply_markup: inline });

const reply = new Keyboard()
  .text("Yes").text("No").row()
  .requestContact("Share contact")
  .resized().oneTime();          // shrink height; hide after one press
await ctx.reply("Confirm?", { reply_markup: reply });
```

Remove a reply keyboard with `{ reply_markup: { remove_keyboard: true } }`.

## Handle inline presses — and always answer

A pressed inline button fires a `callback_query`. **Always `answerCallbackQuery()`** or the
client shows a spinner for ~30s.

```ts
bot.callbackQuery("like", async (ctx) => {
  await ctx.answerCallbackQuery();                       // silent ack
  await ctx.editMessageText("Liked ❤️");                 // update in place
});
bot.callbackQuery("share", (ctx) => ctx.answerCallbackQuery({ text: "Shared!" })); // toast
// alert dialog: ctx.answerCallbackQuery({ text: "Nope", show_alert: true });
```

### Namespaced callback data (routing by prefix)

`callback_data` is a string ≤ 64 bytes. Encode an action + arg, route by prefix:

```ts
new InlineKeyboard().text("Next", `page:${n + 1}`);      // build: `page:2`, `buy:42`

bot.callbackQuery(/^page:(\d+)$/, async (ctx) => {
  const page = Number(ctx.match[1]);                     // regex capture → ctx.match
  await ctx.answerCallbackQuery();
  await ctx.editMessageReplyMarkup({ reply_markup: pageKb(page, last) });
});
```

For complex payloads use the [callback-data plugin](https://grammy.dev/plugins/callback-data)
instead of hand-packing strings.

## Stateful menus — @grammyjs/menu

For multi-button menus that update themselves (toggles, sub-menus), the menu plugin tracks
state and wires callbacks for you — no manual `callback_data`:

```ts
import { Menu } from "@grammyjs/menu";

const menu = new Menu("settings")
  .text("➕", (ctx) => ctx.reply("added")).row()
  .submenu("Advanced", "settings-adv")
  .url("Help", "https://grammy.dev");

bot.use(menu);                                  // register BEFORE handlers that send it
bot.command("settings", (ctx) => ctx.reply("Settings", { reply_markup: menu }));
```

Dynamic labels: pass a function — `.text((ctx) => label, handler)`. For free-form multi-step
input (not buttons), use [telegram-bot-conversations](../telegram-bot-conversations/SKILL.md).

## Pagination & confirm (common patterns)

```ts
function pageKb(n: number, last: number) {          // prev/next in callback_data
  const kb = new InlineKeyboard();
  if (n > 0) kb.text("‹ Prev", `page:${n - 1}`);
  if (n < last) kb.text("Next ›", `page:${n + 1}`);
  return kb;                                          // on press → editMessageReplyMarkup
}
const confirm = new InlineKeyboard().text("✅ Yes", "do:yes").text("❌ No", "do:no");
```

## Common mistakes

1. **Missing `answerCallbackQuery()`** — every `callback_query` handler must call it (spinner otherwise). Pass `text`/`show_alert` for feedback.
2. **`callback_data` > 64 bytes** — Telegram rejects it. Keep payloads short; store big state in [sessions](../telegram-bot-sessions/SKILL.md).
3. **Inline vs reply confusion** — inline → `callback_query`; reply → plain text messages you match with `bot.hears`.
4. **Editing with no change** — `editMessageText` with identical text/markup throws `message is not modified`. Guard it or change the content.
5. **Menu registered after the sender** — `bot.use(menu)` must come before the handler that replies with it, or its buttons are dead.
6. **Re-sending instead of editing** — for pagination/toggles use `editMessageText`/`editMessageReplyMarkup`, not a new message.

## Quick reference

```ts
new InlineKeyboard().text(label, data).url(l, u).webApp(l, u).switchInline(l).row()
new Keyboard().text(l).requestContact(l).requestLocation(l).resized().oneTime().persistent()
{ reply_markup: kb }                         // attach · { remove_keyboard: true } to clear
bot.callbackQuery("data" | /re/, fn)         // ctx.match = capture
ctx.answerCallbackQuery({ text?, show_alert? })
ctx.editMessageText(text, { reply_markup }) · ctx.editMessageReplyMarkup({ reply_markup })
new Menu(id).text(l, fn).submenu(l, id).row() // @grammyjs/menu — bot.use(menu) first
```
