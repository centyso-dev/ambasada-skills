# Checkout: people pay with a redirect, you deliver on the webhook

Same shape as Stripe Checkout.

1. **Your server** creates a session (never the browser: it needs your key).
2. Redirect the person to `session.url`. They approve with their Ambasada ID wallet on hackaton.ambasada.id.
3. They come back to `success_url` with `?session=cs_…` (or wherever you put `{CHECKOUT_SESSION_ID}`).
4. The village POSTs a signed `checkout.paid` webhook to your webhook URL. **Deliver there**, or after
   `GET /v1/checkout/sessions/:id` says `"status": "paid"`. Never on the redirect alone.

```ts
import { village } from "@ambasada/sdk";

// 1–2: POST /buy
export async function buy(req: Request) {
  const order = await db.orders.create({ item: "post-kit" });
  const session = await village.checkout(
    {
      amount: 12,                                   // or { offering: "post_kit" } to charge an offering's price
      description: "Launch post kit: 3 posts + images", // shown to the person AND public in the ledger: no personal data
      success_url: "https://my.app/thanks?session={CHECKOUT_SESSION_ID}",
      cancel_url: "https://my.app/cart",
      client_reference_id: order.id,                // comes back to you, never shown
    },
    { idempotencyKey: order.id },                   // a double click returns the same session
  );
  return Response.redirect(session.url, 303);
}

// 4: POST /village-webhook  (register once: `ambasada webhook set https://my.app/village-webhook`)
export async function webhook(req: Request) {
  const event = await village.verifyWebhook(req); // AMBASADA_WEBHOOK_SECRET; throws if forged or older than 5 min
  if (event.type === "checkout.paid") {
    const s = event.data.session;                  // same shape as GET /v1/checkout/sessions/:id
    await db.orders.markPaidOnce(s.client_reference_id, event.id); // dedupe on event.id: retries reuse it
  }
  return new Response("ok");                       // 2xx within 10 s, or it is retried (1 min … 6 h)
}

// 3: GET /thanks
export async function thanks(req: Request) {
  const s = await village.getCheckout(new URL(req.url).searchParams.get("session")!);
  return s.status === "paid" ? deliver(s.client_reference_id) : pending();
}
```

## Webhook signature (any language)

[Standard Webhooks](https://www.standardwebhooks.com): headers `webhook-id`, `webhook-timestamp`,
`webhook-signature: v1,<base64>`, where the signature is HMAC-SHA256 with the bytes of the secret after
`whsec_` (base64-decoded) over `"<webhook-id>.<webhook-timestamp>.<raw body>"`. Off-the-shelf
libraries verify it: `npm i standardwebhooks`, `pip install standardwebhooks`, Go, PHP, Ruby.
Reject timestamps more than 5 minutes off.

## Managing it

| | HTTP | SDK |
|---|---|---|
| cancel an open session | `POST /v1/checkout/sessions/:id/expire` | `village.expireCheckout(id)` |
| payment links | `POST /v1/checkout/links`, `GET /v1/checkout/links`, `POST /v1/checkout/links/:id/deactivate` | `createPaymentLink`, `listPaymentLinks`, `deactivatePaymentLink` |
| webhook URL + secret | `PUT /v1/webhook { url, rotate_secret? }` → `{ url, secret }`, `GET /v1/webhook` (+ last deliveries), `DELETE /v1/webhook` | `setWebhook`, `getWebhook`, `deleteWebhook` |
| send a test event | `POST /v1/webhook/test` (event `webhook.test`) | `village.testWebhook()` |
| resend a delivery | `POST /v1/webhook/deliveries/:id/retry` (same `webhook-id`) | `village.retryWebhookDelivery(id)` |

Events: `{ "type": "checkout.paid", "timestamp": ISO, "data": { "session": <the session> } }` and
`{ "type": "webhook.test", "data": { "company", "message" } }`.

## Rules

- `amount` 0.1–500 TON. A person can spend at most 30 TON a day at one team.
- `metadata`: at most 20 string values. `expires_in`: 300–86400 seconds.
- No automatic refunds: refund with `village.transfer("p_…", amount, { reason })` (the buyer id is `session.buyer`).
- Sessions expire after `expires_in` seconds (default 3600).
- Full API guide: `docs/PAYMENTS.md` in the village repo. Without the SDK, `@ambasada/protocol` exports the same
  `verifyWebhook(secret, headers, rawBody)` and `verifyCheckoutReceipt(token, { keys, seller, session })`.
