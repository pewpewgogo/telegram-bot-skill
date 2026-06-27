---
name: telegram-bot-payments
description: >
  Use when a Telegram bot charges money — Telegram Stars (XTR digital goods) or
  provider payments — covering sendInvoice, the pre_checkout_query you MUST answer,
  successful_payment handling, and Stars refunds.
  Triggers: telegram payments, sendInvoice, invoice, Telegram Stars, XTR, pre_checkout_query,
  answerPreCheckoutQuery, successful_payment, refund stars, paywall, sell digital goods,
  donate, buy button, pay button.
compatibility: grammY v1 (`grammy`). Stars (XTR) need no provider token; fiat needs a BotFather-connected provider.
license: MIT
---

# telegram-bot-payments

Two rails: **Telegram Stars** (`XTR`) for digital goods/services — no provider, no token — and
**provider payments** (fiat via Stripe etc., connected in @BotFather). Stars is the modern
default and what Telegram requires for in-app digital goods. Sending/`ctx` →
[telegram-bot-basics](../telegram-bot-basics/SKILL.md).

## The flow

```
sendInvoice → user pays → pre_checkout_query (you MUST answer within 10s)
            → successful_payment (deliver the goods)
```

## 1. Send an invoice (Stars)

```ts
await ctx.replyWithInvoice(
  "Pro plan",                 // title
  "Unlocks everything",       // description
  JSON.stringify({ userId: ctx.from!.id, sku: "pro" }), // payload — your internal order ref
  "XTR",                      // currency: Stars
  [{ label: "Pro", amount: 100 }], // prices; for XTR amount = number of Stars (no decimals)
);
// provider payments: pass a provider_token and use fiat currency + cents in `amount`.
```

The **payload** is echoed back to you on success — pack the order id / user / sku there so you
know what was bought. Don't trust the client for that.

## 2. Answer the pre-checkout (required)

```ts
bot.on("pre_checkout_query", async (ctx) => {
  const ok = await stillAvailable(ctx.preCheckoutQuery.invoice_payload);
  await ctx.answerPreCheckoutQuery(ok, ok ? undefined : "Sold out");
});
```

You have ~10 seconds. **Not answering = the payment fails.** Answer `false` with a reason to
reject (out of stock, invalid order).

## 3. Fulfill on success

```ts
bot.on("message:successful_payment", async (ctx) => {
  const p = ctx.msg.successful_payment;
  const order = JSON.parse(p.invoice_payload);
  await grantAccess(order);                       // idempotent: deliver exactly once
  // keep p.telegram_payment_charge_id — required for refunds
  await ctx.reply("Thanks! Access granted.");
});
```

## Refunds (Stars)

```ts
await ctx.api.refundStarPayment(userId, telegramPaymentChargeId);
```

## Common mistakes

1. **Not answering `pre_checkout_query`** — silence (or > 10s) fails the charge. Always `answerPreCheckoutQuery`.
2. **Trusting the client for what was bought** — derive it from `invoice_payload` you set, not from user input.
3. **Non-idempotent fulfillment** — duplicate `successful_payment` (retries) double-grants. Make delivery idempotent on `telegram_payment_charge_id`.
4. **Dropping the charge id** — you can't refund Stars without `telegram_payment_charge_id`. Persist it.
5. **Wrong amount units** — for `XTR`, `amount` is a whole number of Stars; for fiat it's the smallest unit (cents).
6. **Using a provider token for Stars** — Stars (`XTR`) need none; provider tokens are for fiat only.

## Quick reference

```ts
ctx.replyWithInvoice(title, description, payload, "XTR", [{ label, amount }])
ctx.api.sendInvoice({ chat_id, title, description, payload, currency, prices, provider_token? })
bot.on("pre_checkout_query", …) → ctx.answerPreCheckoutQuery(ok, errorMessage?)   // ≤10s, required
bot.on("message:successful_payment", …) → ctx.msg.successful_payment.{invoice_payload, telegram_payment_charge_id}
ctx.api.refundStarPayment(userId, chargeId)
```
