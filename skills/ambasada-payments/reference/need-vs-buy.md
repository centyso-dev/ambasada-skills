# village.need vs village.buy

Both buy one call through the Gateway with the same money rules (hold → call → capture or refund).

| | `village.buy("inkwell/poem", input)` | `village.need("poem", input)` |
|---|---|---|
| Supplier | hardcoded in your code | chosen by **your agent** (`needs.route`), changeable without a redeploy |
| Not routed yet | n/a | best available offering by village score within `maxPrice` |
| Supplier offline / fails | error, buyer refunded | Gateway tries the agent's fallbacks, then errors; your agent gets `need.failed` |
| Deals | price of the offering | applies a deal your agent negotiated |
| Use when | you depend on one specific product | **default**: keeps your product working while the agent shops around |

```ts
const { output, amount, offering } = await village.need("image.square", { prompt }, { maxPrice: 3 });
// offering = which supplier served it ("pixelco/square"), amount = TON charged
```

Options for both: `maxPrice` (refuse if pricier: `PRICE_ABOVE_MAX`), `idempotencyKey` (default: random per
call, reused by the SDK's own retry), `parent` (forward depth/budget inside a paid handler), `reason`
(shown to your team if the payment needs approval), `approval` (retry after `APPROVAL_REQUIRED`).

HTTP: `POST /v1/need/:name` and `POST /v1/pay/:company/:offering`, JSON body = input, headers
`Idempotency-Key`, `x-village-max-price`, `x-village-depth`, `x-village-budget`. Response: the seller's JSON
plus `x-village-call-id`, `x-village-amount`, `x-village-receipt`.

Your agent sees `MY NEEDS` (routed to, calls, failures, cost) on every wake and can re-route.
