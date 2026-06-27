---
name: telegram-bot-security
description: >
  Use when hardening a Telegram bot — bot-token hygiene, webhook secret tokens,
  validating and sanitizing user input, admin/authorization guards, abuse/flood
  rate-limiting, and not leaking secrets in logs or replies.
  Triggers: bot security, secure telegram bot, token leak, webhook secret, validate
  input, sanitize, admin only, authorization, rate limit abuse, prevent spam,
  injection, untrusted input, who can use bot.
compatibility: grammY v1 (`grammy`). Rate-limiting via `@grammyjs/ratelimiter`; initData validation via Node `crypto`.
license: MIT
---

# telegram-bot-security

Everything a bot receives is **attacker-controllable** — chat ids, text, file names, callback
data, deep-link payloads. Treat it as untrusted. Core handlers →
[telegram-bot-basics](../telegram-bot-basics/SKILL.md).

## Secrets

- **Bot token** in env only (`process.env.BOT_TOKEN`); never commit, never log, never echo to a chat. A leak = full bot takeover; rotate via @BotFather if exposed.
- **Webhook secret token** — set `secret_token` on `setWebhook` and the matching `secretToken` on `webhookCallback` so forged POSTs are rejected → [telegram-bot-deploy](../telegram-bot-deploy/SKILL.md).
- **Mini App initData** — validate the HMAC server-side before trusting any user identity → [telegram-bot-mini-apps](../telegram-bot-mini-apps/SKILL.md).
- Scrub tokens/PII from error logs; don't send raw exception text back to users.

## Authorization

Identity is `ctx.from.id` (a stable integer) — **not** username (mutable, spoofable in display).
Gate privileged actions with a guard, ideally as a `Composer`:

```ts
import { Composer } from "grammy";
const ADMINS = new Set(process.env.ADMIN_IDS!.split(",").map(Number));

const admin = new Composer<MyContext>();
admin.use((ctx, next) => (ctx.from && ADMINS.has(ctx.from.id) ? next() : undefined)); // else: silent drop
admin.command("broadcast", (ctx) => { /* … */ });
bot.use(admin);
```

In groups, check actual membership with `ctx.getChatMember(userId)` and the `status` field
(`creator`/`administrator`) rather than assuming.

## Validate & sanitize input

- **Parse, don't assume.** `Number(ctx.match)` → check `Number.isFinite`; bound array indices and page numbers from `callback_data`.
- **Never interpolate user text into SQL / shell / file paths.** Use parameterized queries; reject path separators in file names.
- **Echoing user text?** Send as plain text or escape it for your `parse_mode` — unescaped MarkdownV2/HTML lets users break formatting or inject links → [telegram-bot-messages](../telegram-bot-messages/SKILL.md).
- **Deep-link payloads** (`/start <payload>`) are attacker-set — validate before acting.

## Abuse & flood control

```ts
import { limit } from "@grammyjs/ratelimiter";
bot.use(limit({ timeFrame: 1000, limit: 5, onLimitExceeded: (ctx) => {} })); // per user
```

Drop or rate-limit unknown users on expensive operations; cap pagination/loop sizes so a
crafted update can't make the bot spam itself into a `429` → [telegram-bot-scaling](../telegram-bot-scaling/SKILL.md).

## Common mistakes

1. **Token in repo/logs/replies** — the one unrecoverable leak. Env only; scrub logs; rotate if exposed.
2. **Webhook without `secret_token`** — public URL accepts forged updates. Always set and verify it.
3. **Authorizing by username** — usernames change and aren't unique-stable for trust. Use `ctx.from.id`.
4. **Trusting `callback_data` / deep-link payloads** — both are client-set. Validate bounds and ownership before acting.
5. **Interpolating user input into SQL/shell/paths** — injection. Parameterize; sanitize file names.
6. **Echoing unescaped user text under a `parse_mode`** — formatting injection. Escape or send plain.
7. **No flood limit on heavy actions** — one user can exhaust quota or rack up cost. Rate-limit and cap loop sizes.
8. **Unvalidated Mini App initData** — forged identity. Verify HMAC + `auth_date` server-side.

## Quick reference

```ts
process.env.BOT_TOKEN                         // never commit/log/echo
setWebhook(url, { secret_token }) + webhookCallback(bot, adapter, { secretToken })
ctx.from.id (Set<number> allowlist)           // authorize by id, not username
Number.isFinite(Number(x)) · bound indices · parameterized queries
bot.use(limit({ timeFrame, limit }))          // @grammyjs/ratelimiter
// validate Mini App initData HMAC server-side → telegram-bot-mini-apps
```
