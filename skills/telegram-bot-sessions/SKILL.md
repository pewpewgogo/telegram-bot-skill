---
name: telegram-bot-sessions
description: >
  Use when a Telegram bot needs per-user or per-chat state that survives across
  messages — grammY's session plugin, the session key, storage adapters
  (memory/Redis/file/free), lazy sessions, session-shape design, and migrations.
  Triggers: session, persist state, user state, ctx.session, SessionFlavor,
  storage adapter, RedisAdapter, session key, getSessionKey, lazySession,
  remember user, counter, save state.
compatibility: grammY v1 (`grammy`); adapters via `@grammyjs/storage-*`. Node 18+ or Deno.
license: MIT
---

# telegram-bot-sessions

The Bot API is stateless — Telegram remembers nothing about a user between updates. The
**session plugin** attaches a per-key object (`ctx.session`) that you read/write freely and
that persists to storage. Context type setup → [telegram-bot-basics](../telegram-bot-basics/SKILL.md).

## Wire it up

```ts
import { Bot, Context, session, SessionFlavor } from "grammy";

interface SessionData { count: number }
type MyContext = Context & SessionFlavor<SessionData>;

const bot = new Bot<MyContext>(process.env.BOT_TOKEN!);
bot.use(session({ initial: (): SessionData => ({ count: 0 }) })); // initial() runs per new key

bot.command("inc", async (ctx) => {
  ctx.session.count++;                      // typed; auto-persists after the handler
  await ctx.reply(`count: ${ctx.session.count}`);
});
```

`initial` is **required** and must return a *fresh* object each call (never share a reference).
Default storage is in-memory (lost on restart) — fine for dev, not production.

## The session key — what "per" means

`getSessionKey(ctx)` decides scope. Default is **per chat** (`ctx.chat.id`):

```ts
session({ initial, getSessionKey: (ctx) => ctx.from?.id.toString() });        // per user (any chat)
session({ initial, getSessionKey: (ctx) => ctx.from && ctx.chat && `${ctx.chat.id}:${ctx.from.id}` }); // per user per chat
```

If the key is `undefined`, that update has **no session** (e.g. a user-less update under a
per-user key) — guard before writing.

## Storage adapters (production)

```ts
import { RedisAdapter } from "@grammyjs/storage-redis";
import { Redis } from "ioredis";
const storage = new RedisAdapter({ instance: new Redis(process.env.REDIS_URL!) });
bot.use(session({ initial, storage }));
```

| Adapter | Package | Use |
|---|---|---|
| Memory (default) | built-in | dev only — lost on restart |
| Redis | `@grammyjs/storage-redis` | production default (fast, shared across instances) |
| File | `@grammyjs/storage-file` | single-host, low volume |
| free | `@grammyjs/storage-free` | zero-setup hobby (grammY-hosted) |

Also published: PostgreSQL, MongoDB, Supabase adapters. All implement the same
`StorageAdapter` interface, so swapping is a one-line change.

### lazySession

`lazySession` reads storage only when you actually touch `ctx.session` (await it), saving a
fetch on updates that ignore state:

```ts
import { lazySession, LazySessionFlavor } from "grammy";
type MyContext = Context & LazySessionFlavor<SessionData>;
bot.use(lazySession({ initial, storage }));
bot.command("inc", async (ctx) => { const s = await ctx.session; s.count++; });
```

## Designing the shape

- **Small & serializable.** Sessions are JSON round-tripped to storage. Store IDs and
  primitives, not class instances, big blobs, or secrets.
- **State machine for flows.** A `step` field (`"idle" | "awaiting_name"`) drives manual
  multi-step input. For anything beyond ~2 steps prefer
  [telegram-bot-conversations](../telegram-bot-conversations/SKILL.md).
- **Migrations.** Old stored sessions won't have new fields. Default safely in code
  (`ctx.session.foo ??= []`) rather than assuming the shape.

## Common mistakes

1. **No persistent storage in prod** — default memory storage drops all state on restart/redeploy. Set a `storage` adapter.
2. **Shared `initial` reference** — `initial: () => SHARED_OBJECT` leaks state across users. Return a new object literal each call.
3. **Wrong key scope** — defaulting to per-chat when you meant per-user (or vice versa) in groups corrupts who-owns-what. Set `getSessionKey` deliberately.
4. **Writing on an undefined key** — under a per-user key, user-less updates have no session. Guard `ctx.from`.
5. **Storing secrets/large blobs** — sessions are plain JSON in shared storage; keep them small and non-sensitive.
6. **Assuming the shape exists** — after a schema change, old sessions lack new fields. Default with `??=`.
7. **Multi-instance with file/memory** — only Redis (or a DB) is safe when several bot instances share a token's load → [telegram-bot-scaling](../telegram-bot-scaling/SKILL.md).

## Quick reference

```ts
type MyContext = Context & SessionFlavor<SessionData>
session({ initial: () => ({...}), storage?, getSessionKey?, prefix? })
ctx.session.<field>                          // read/write, auto-persists after handler
lazySession({ initial, storage }) + await ctx.session     // deferred read
new RedisAdapter({ instance: redis })        // @grammyjs/storage-redis
```
