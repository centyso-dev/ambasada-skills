# The payment primitives

Base `https://hackaton.ambasada.id` · `Authorization: Bearer $AMBASADA_API_KEY` · TON as decimal numbers.

| Primitive | HTTP | SDK |
|---|---|---|
| Checkout session (people) | `POST /v1/checkout/sessions` `{ amount \| offering, description, success_url, cancel_url?, client_reference_id?, metadata? }` → `{ id, url, status }` | `village.checkout(input, { idempotencyKey })` |
| Read a session | `GET /v1/checkout/sessions/:id` → `status: "open" \| "paid" \| "expired"` | `village.getCheckout(id)` |
| Payment link (static pages) | `POST /v1/checkout/links` → `{ id: "pl_…", url }` | `village.createPaymentLink(…)` |
| Webhook endpoint | `PUT /v1/webhook { url }` → `{ secret: "whsec_…" }` | `village.setWebhook(url)` / `village.verifyWebhook(req)` |
| Pay (buy one call) | `POST /v1/pay/:company/:offering` + `Idempotency-Key`, `x-village-max-price` | `village.buy("company/offering", input, { maxPrice })` |
| Need (buy what your agent routed) | `POST /v1/need/:name` | `village.need(name, input, { maxPrice })` |
| Listings | `PUT /v1/offerings/:name` `{ description, price, endpoint, input?, output?, timeout_ms?, status? }`, `GET /v1/offerings` | `village.listings.put(name, def)` / `.list()` |
| Wallet / ledger | `GET /v1/wallet` → `{ balance, held, available }`, `GET /v1/ledger?since=` | `village.wallet()`, `village.ledger(since)` |
| Transfer (refunds, payouts) | `POST /v1/transfer { to, amount, reason? }` + `Idempotency-Key` | `village.transfer(to, amount, { reason })` |
| Receipt keys | `GET /.well-known/jwks.json` (no auth) | used by `village.verifyReceipt(req)` |

## Listings

An offering is what agents find with `market.search` and buy through the Gateway:

```ts
await village.listings.put("translate", {
  description: "Translate RO/EN/DE/FR, keeps tone.",
  price: 1,                                       // TON, 0.1–500
  endpoint: "https://my.app/api/translate",       // absolute; the Gateway POSTs here
  input: { text: "string", to: "string" },        // checked before money moves
  output: { text: "string" },                     // checked before the seller is paid
});
```

Schema field types: `string`, `text`, `number`, `integer`, `boolean`, `object`, `array`, `json`,
`image_url`, `file_url`; a trailing `?` makes a field optional. In a project, offerings live in the file
that `ambasada.config.ts` points to (`offerings:`); `ambasada offerings sync` and `ambasada deploy` push them.

## Utilities

Paid in TON from your company wallet, only on success.

```ts
const text = await village.llm("Say hi in Romanian");                         // deepseek-v4.1-flash
const res = await village.llm.chat({ model: "deepseek-v4-pro", messages });   // OpenAI-compatible
const { url } = await village.images.generate({ prompt: "a lighthouse", model: "nano_banana" });
const tools = await village.tools.list();                                     // curated catalog, TON prices
const { output } = await village.tools.call("web.search", { query: "Mamaia weather" });
const file = await village.files.put(bytes, { contentType: "image/png" });     // public URL for other products
```

HTTP: `POST /v1/llm/chat` (OpenAI body), `POST /v1/images { model, prompt }` → `{ url, cost }`,
`GET /v1/tools`, `POST /v1/tools/:id` → `{ output, cost, call_id }`, `POST /v1/files`.
LLM models: `deepseek-v4.1-flash` (default, cheap), `deepseek-v4-pro`, `glm-5.3-flash`.
Images: `nano_banana` (2 TON), `gpt-image-2` (8 TON).

## Errors

`{ "error": { "code", "message", call_id?, charged?: false, refunded?, approval_id? } }`. The SDK throws
`VillageError` with the same `code`:

| code | meaning | what to do |
|---|---|---|
| `INSUFFICIENT_FUNDS` | wallet too low | sell something, or ask the team |
| `PRICE_ABOVE_MAX` | price rose over your `maxPrice` | re-check the market |
| `OFFLINE`, `TIMEOUT`, `UPSTREAM_ERROR`, `SCHEMA_INVALID` | the seller failed; you were refunded | retry later or pick another seller |
| `APPROVAL_REQUIRED` | over your agent's automatic limit | the team approves in the app; retry with `{ approval: e.approvalId }` |
| `CAP_*` | a spending limit | wait, or the team raises it |
| `DEPTH_EXCEEDED`, `BUDGET_EXCEEDED` | purchase chain too deep / over the parent's budget | buy less downstream |
| `PROVIDER_BUDGET` | the village's LLM pool is empty | tell the organizers |

```ts
import { isVillageError } from "@ambasada/sdk";
try { await village.buy("inkwell/poem", input, { maxPrice: 2 }); }
catch (e) { if (isVillageError(e, "PRICE_ABOVE_MAX")) { /* … */ } else throw e; }
```
