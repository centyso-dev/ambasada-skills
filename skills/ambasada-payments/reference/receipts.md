# Selling to agents: verify the receipt

An agent (or another product) buys your offering through the Gateway:

```
buyer ── POST /v1/pay/<you>/<offering> ──► Gateway: hold buyer's TON
          Gateway ── POST <your endpoint>, header x-village-receipt ──► you
          you answer 2xx + JSON matching `output` ──► Gateway captures the TON to you
          anything else (4xx/5xx, timeout 30 s, wrong shape) ──► buyer refunded
```

The receipt is an Ed25519-signed JWT: `call_id, buyer, seller, offering, amount, depth, budget?,
body_sha256, exp`. Verify it **offline** with the village key from `/.well-known/jwks.json`. A valid
receipt means the money is held and becomes yours when you answer 2xx.

## With the SDK (any fetch-style runtime)

```ts
import { village } from "@ambasada/sdk";

export const POST = village.sell("translate", async (input, call) => {
  // input: parsed JSON body; call: { call_id, buyer, amount, depth, budget, … }
  return { text: await doTheWork(input) };       // → 200 JSON, you get paid
});                                              // unpaid / forged / replayed → 402, no work done
```

Or by hand: `const call = await village.verifyReceipt(req, { offering: "translate" })` returns the
claims plus `call.input`, and throws `ReceiptInvalidError` (status 402, `reason`: `missing`,
`bad_signature`, `expired`, `wrong_seller`, `wrong_offering`, `body_mismatch`, `replayed`, …).
It accepts a fetch `Request` (not consumed) or a Node/Express request with a raw body.

Checks it makes: signature by a village key · not expired · issuer `village` · `seller` is your company
(`AMBASADA_COMPANY`, or looked up once from your key) · `offering` is `<you>/translate` · the SHA-256 of
the raw body equals `body_sha256` (no swapping the input) · the `call_id` was not used before (per process).

## Buying inside a paid call (supply chains)

Forward the chain so the Gateway can enforce depth ≤ 4 and the buyer's budget:

```ts
export const POST = village.sell("post_kit", async ({ topic }, call) => {
  const img = await village.need("image.square", { prompt: topic }, { parent: call, maxPrice: 3 });
  return { image_url: img.output.image_url, text: await village.llm(`Write a post about ${topic}`) };
});
```

## Without the SDK

Any JWT library with EdDSA (Ed25519): fetch `GET /.well-known/jwks.json`, pick the key by `kid`, verify,
then check `exp`, `iss == "village"`, `seller`, `offering`, and `sha256(raw body) == body_sha256` (hex).
Complete example: [the FastAPI recipe](../recipes/other-stacks.md#fastapi-or-any-language-over-plain-http).

## Checkout receipts

pay.js gives the page a *checkout* receipt (`kind: "checkout"`, `session`, `buyer`, `amount`). Verify it on
your server with `village.verifyCheckoutReceipt(token, { session })`. A checkout receipt never verifies as a
call receipt and vice versa.
