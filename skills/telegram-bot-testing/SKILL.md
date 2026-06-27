---
name: telegram-bot-testing
description: >
  Use when testing a grammY Telegram bot without hitting Telegram — intercept outgoing
  API calls with a transformer, feed synthetic Update objects through bot.handleUpdate,
  and assert on what the bot tried to send (vitest/jest). The general testing layer.
  Triggers: test telegram bot, mock telegram api, bot.handleUpdate, transformer mock,
  synthetic update, assert sendMessage, vitest bot, unit test handler, fake update,
  test without token.
compatibility: grammY v1 (`grammy`) + any test runner (vitest/jest).
license: MIT
---

# telegram-bot-testing

Bots are pure-ish functions: an `Update` goes in, API calls come out. Test by **capturing the
outgoing calls** and feeding **synthetic updates** — no token, no network. Build/run split →
[telegram-bot-basics](../telegram-bot-basics/SKILL.md).

## Capture outgoing calls with a transformer

An API transformer sits in front of every `bot.api` call. Stub the network and record what was
sent:

```ts
import { Bot } from "grammy";

function makeTestBot() {
  const bot = new Bot("test-token", { botInfo: FAKE_BOT_INFO }); // botInfo skips getMe()
  registerHandlers(bot);
  const calls: { method: string; payload: any }[] = [];
  bot.api.config.use((prev, method, payload, signal) => {
    calls.push({ method, payload });
    return Promise.resolve({ ok: true, result: {} } as any); // never hits Telegram
  });
  return { bot, calls };
}
```

`botInfo` must be supplied (any `id`/`username`/`is_bot:true` shape) so the bot doesn't call
`getMe` at startup.

## Feed a synthetic update

```ts
import { test, expect } from "vitest";

function textUpdate(text: string): Update {
  return {
    update_id: 1,
    message: {
      message_id: 1, date: 0, text,
      chat: { id: 1, type: "private" },
      from: { id: 1, is_bot: false, first_name: "T" },
    },
  } as Update;
}

test("/start replies", async () => {
  const { bot, calls } = makeTestBot();
  await bot.handleUpdate(textUpdate("/start"));
  const sent = calls.find((c) => c.method === "sendMessage");
  expect(sent?.payload.text).toContain("Hello");
});
```

`bot.handleUpdate(update)` runs the full middleware stack exactly as in production, then
resolves — so you assert *after* it returns.

## Simulating failures

Return a Telegram-style error from the transformer to exercise error paths:

```ts
bot.api.config.use((prev, method) => {
  if (method === "sendMessage") return Promise.reject(new GrammyError("Forbidden", { ok: false, error_code: 403, description: "bot was blocked" } as any, method, {}));
  return Promise.resolve({ ok: true, result: {} } as any);
});
```

Assert your `bot.catch` / `errorBoundary` handles `403`/`429` correctly →
[telegram-bot-scaling](../telegram-bot-scaling/SKILL.md).

## What to test

- **Routing**: the right command/filter handler fires.
- **Output**: text, `reply_markup`, `parse_mode` of the captured call.
- **State**: `ctx.session` transitions (inject a fresh in-memory store per test).
- **Branches**: callback queries (`answerCallbackQuery` was called), error paths.

Use a **fresh bot + fresh storage per test** so state never leaks between cases.

## Common mistakes

1. **No `botInfo`** — the bot calls `getMe` on first use and your test makes a real network call. Pass `botInfo`.
2. **Asserting before `await handleUpdate`** — handlers are async; await it, then check `calls`.
3. **Shared bot/state across tests** — leaks sessions between cases. Rebuild per test.
4. **Transformer returning the wrong shape** — return `{ ok: true, result: … }` (or reject with `GrammyError`) so grammY parses it.
5. **Testing through Telegram for real** — flaky, rate-limited, needs a token. Capture via transformer instead.
6. **Forgetting callback assertions** — verify `answerCallbackQuery` fired, not just the edit.

## Quick reference

```ts
new Bot("test", { botInfo })                  // no getMe()
bot.api.config.use((prev, method, payload) => { record(); return Promise.resolve({ ok:true, result:{} }) })
await bot.handleUpdate(update)                // run full stack, then assert
reject(new GrammyError(msg, { ok:false, error_code, description }, method, payload))  // failure path
// fresh bot + fresh session storage per test
```
