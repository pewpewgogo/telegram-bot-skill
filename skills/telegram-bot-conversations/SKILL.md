---
name: telegram-bot-conversations
description: >
  Use when a Telegram bot needs multi-step dialogs — ask a question, wait for the
  reply, branch, loop — with the @grammyjs/conversations plugin (v2). Covers
  enter/wait/waitFor, forms, the replay model and conversation.external() for side
  effects, plus when to use a session state machine instead.
  Triggers: conversation, multi-step dialog, wizard, ask and wait, conversation.wait,
  waitFor, createConversation, ctx.conversation.enter, form, booking flow, signup flow,
  step by step input.
compatibility: grammY v1 + `@grammyjs/conversations` v2. Node 18+ or Deno.
license: MIT
---

# telegram-bot-conversations

When a flow is "ask → wait for answer → ask again," the conversations plugin lets you write
it as **linear `async` code** instead of a session state machine. Simple per-user state →
[telegram-bot-sessions](../telegram-bot-sessions/SKILL.md); buttons → [telegram-bot-ui](../telegram-bot-ui/SKILL.md).

## Install & enter

```ts
import { Bot, Context } from "grammy";
import {
  conversations, createConversation,
  type Conversation, type ConversationFlavor,
} from "@grammyjs/conversations";

type MyContext = ConversationFlavor<Context>;
const bot = new Bot<MyContext>(process.env.BOT_TOKEN!);
bot.use(conversations());                          // install the plugin

async function greet(conversation: Conversation, ctx: Context) {
  await ctx.reply("What's your name?");
  const { message } = await conversation.waitFor("message:text"); // pause until text arrives
  await ctx.reply(`Hello, ${message.text}!`);
}

bot.use(createConversation(greet));                // register by function name
bot.command("greet", (ctx) => ctx.conversation.enter("greet")); // start it
```

`enter()` launches the conversation; control returns to your normal handlers when the
function ends (or you `halt()`).

## Waiting

```ts
const ctx2  = await conversation.wait();                  // any update
const txt   = await conversation.waitFor("message:text"); // a filter query
const cb    = await conversation.waitFor("callback_query:data");
const cmd   = await conversation.waitForHears(/^\d+$/);   // pattern
// loop until valid:
let age: number;
do { const m = await conversation.waitFor("message:text"); age = Number(m.message.text); }
while (Number.isNaN(age));
```

## Forms (built-in validation)

```ts
const name = await conversation.form.text();          // waits for text, returns it
const n    = await conversation.form.number();        // re-prompts until numeric
const photo = await conversation.form.photo();
```

## The replay model — the one thing to get right

The plugin runs your function by **replaying it from the top** on each new update until the
next `wait`. So the function must be **deterministic**: anything with side effects or
randomness (DB reads, `fetch`, `Date.now()`, `Math.random()`) must be wrapped so it runs once
and its result is cached:

```ts
const user = await conversation.external(() => db.getUser(ctx.from!.id)); // runs once, replay-safe
const now  = await conversation.now();                                    // replay-safe time
```

Plain `await db.getUser(...)` inside a conversation will re-run on every replay — wrap it in
`conversation.external()`.

## When NOT to use conversations

- **One-shot commands** — no waiting needed; just a handler.
- **Button-driven menus** — use [telegram-bot-ui](../telegram-bot-ui/SKILL.md) `@grammyjs/menu`.
- **Long-lived/global state** (settings, counters) — that's a [session](../telegram-bot-sessions/SKILL.md), not a conversation.

## Common mistakes

1. **Side effects not wrapped** — `fetch`/DB/`Date.now()`/random run on every replay and corrupt the flow. Wrap in `conversation.external()` (or `conversation.now()`/`conversation.random()`).
2. **Forgetting `bot.use(conversations())`** — `ctx.conversation` is undefined without the base plugin, before `createConversation`.
3. **Name mismatch** — the string in `enter("greet")` must match the registered function's name (or the id you pass to `createConversation`).
4. **Reading `ctx` mutations across a `wait`** — capture what you need from the returned update; the outer `ctx` is the entering update.
5. **Heavy logic where a state machine fits** — for 1–2 steps a session `step` field is lighter than pulling in the plugin.
6. **Sending from inside `external`** — only put pure side-effecting data fetches there; do `ctx.reply` in the conversation body.

## Quick reference

```ts
bot.use(conversations())                          // base plugin (once)
bot.use(createConversation(fn, id?))              // register
ctx.conversation.enter(id) / .exit(id)            // start / stop
conversation.wait() / .waitFor(query) / .waitForHears(re)
conversation.form.text() / .number() / .photo()
conversation.external(() => sideEffect())         // replay-safe I/O
conversation.now() / .random() / .halt()
```
