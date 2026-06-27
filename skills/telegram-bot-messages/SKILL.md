---
name: telegram-bot-messages
description: >
  Use when sending or editing rich Telegram messages with grammY — text formatting
  (HTML / MarkdownV2 / the parse-mode plugin's fmt), entities, editing and deleting,
  and sending/receiving media (photos, documents, InputFile) plus downloading files.
  Triggers: format message, bold italic, MarkdownV2, HTML parse_mode, fmt, escape,
  sendPhoto, send document, InputFile, caption, edit message, delete message,
  download file, getFile, media group, link preview.
compatibility: grammY v1 (`grammy`); nicer formatting via `@grammyjs/parse-mode`, downloads via `@grammyjs/files`.
license: MIT
---

# telegram-bot-messages

Formatting text, editing/deleting, and moving media. Sending basics & `ctx` →
[telegram-bot-basics](../telegram-bot-basics/SKILL.md).

## Formatting text

Telegram renders **HTML** or **MarkdownV2**, never both in one message. Pick a `parse_mode`:

```ts
await ctx.reply("<b>bold</b> <i>italic</i> <a href='https://x.com'>link</a>", { parse_mode: "HTML" });
await ctx.reply("*bold* _italic_ `code`", { parse_mode: "MarkdownV2" });
```

**MarkdownV2 requires escaping** `_ * [ ] ( ) ~ \` > # + - = | { } . !` in any literal text —
miss one and the whole send 400s. **HTML only needs `< > &` escaped**, so HTML is safer for
dynamic content.

### Safer: the parse-mode plugin

`@grammyjs/parse-mode`'s `fmt` builds text + entities with **no manual escaping** — user
input is interpolated literally:

```ts
import { fmt, bold, link } from "@grammyjs/parse-mode";
const msg = fmt`Hi ${bold(userName)}, see ${link("docs", "https://grammy.dev")}`;
await ctx.reply(msg.text, { entities: msg.entities });   // no parse_mode needed
```

Set a default parse mode for every send via the plugin's transformer if you prefer raw HTML
throughout. Disable link previews with `{ link_preview_options: { is_disabled: true } }`.

## Editing & deleting

```ts
await ctx.api.editMessageText(chatId, messageId, "new text");
await ctx.editMessageText("new text");          // shortcut in a callback handler
await ctx.deleteMessage();                       // the message in ctx
await ctx.api.deleteMessage(chatId, messageId);
```

Editing to **identical** content throws `message is not modified` — guard it. You can only
delete messages within Telegram's time window (≈48h, or your own if you're admin).

## Sending media

```ts
import { InputFile } from "grammy";

await ctx.replyWithPhoto("https://example.com/cat.jpg", { caption: "A cat" }); // by URL
await ctx.replyWithPhoto(new InputFile("./local.jpg"));                        // local file/Buffer/stream
await ctx.replyWithDocument(new InputFile(buffer, "report.pdf"));
await ctx.replyWithPhoto(existingFileId);                                      // reuse Telegram's file_id (fastest)
```

`file_id` is returned on every uploaded media and can be re-sent forever without re-upload —
cache it. Captions accept the same `parse_mode`/`entities` as text. Group media with
`ctx.replyWithMediaGroup([...])`.

## Receiving & downloading files

```ts
bot.on("message:photo", async (ctx) => {
  const file = await ctx.getFile();              // file_path + metadata (≤ 20 MB)
  const url  = `https://api.telegram.org/file/bot${process.env.BOT_TOKEN}/${file.file_path}`;
  // or the files plugin for a one-liner download:
});
```

With `@grammyjs/files` (`bot.api.config.use(hydrateFiles(token))`), `await file.download()`
saves to a temp path and `file.getUrl()` returns the URL.

## Common mistakes

1. **Unescaped MarkdownV2** — one stray `.`/`-`/`!` in dynamic text 400s the send. Use HTML, or `@grammyjs/parse-mode` `fmt`.
2. **Mixing entities and `parse_mode`** — supply one or the other for a message, not both.
3. **Editing to identical content** — `message is not modified`. Compare before editing.
4. **Re-uploading the same media** — reuse the returned `file_id` instead of re-sending bytes.
5. **Assuming any file size downloads** — the Bot API download path caps at ~20 MB; larger files can't be fetched this way.
6. **Forgetting captions take formatting** — pass `parse_mode`/`entities` to media sends, not just text.
7. **Noisy link previews** — disable with `link_preview_options.is_disabled` when they clutter.

## Quick reference

```ts
ctx.reply(text, { parse_mode: "HTML" | "MarkdownV2", link_preview_options })
fmt`… ${bold(x)} …` → { text, entities }          // @grammyjs/parse-mode (no escaping)
ctx.editMessageText / ctx.api.editMessageText(chatId, msgId, text)
ctx.deleteMessage() / ctx.api.deleteMessage(chatId, msgId)
ctx.replyWithPhoto / replyWithDocument / replyWithMediaGroup(InputFile | url | file_id, { caption })
new InputFile(path | Buffer | stream, name?) · ctx.getFile() · hydrateFiles(token) → file.download()
```
