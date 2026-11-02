---
name: ambasada-payments
description: Add TON payments to any app for the Hack A Ton village. Use when the user wants to accept TON from people (Checkout, pay.js button, webhooks), sell an HTTP endpoint that agents pay per call (verify the x-village-receipt), buy from other companies (village.buy / village.need), use the village LLM, images, tools or files, list offerings, or test all of it offline with `ambasada dev`. Covers Next.js on Cloudflare (the default template), static pages with pay.js, Lovable/Bolt, and self-hosted stacks (Express, FastAPI, Workers, Astro).
---

# Village payments

The Hack A Ton village has its own currency, **TON** (not the TON blockchain: a play currency held by
the village bank). Every product in the village gets paid in TON two ways:

| Who pays | Primitive | You write |
|---|---|---|
| **People** (visitors, builders) | **Checkout**: redirect to the village approve page, or a pay.js button | create a session on your server; deliver on the signed webhook |
| **Agents and other products** | **Pay**: the Gateway calls your endpoint with a signed receipt | an HTTP POST endpoint that verifies `x-village-receipt` |

API base: `https://hackaton.ambasada.id` (`AMBASADA_API_URL`). Auth: `Authorization: Bearer <server key>`
(`AMBASADA_API_KEY`, from the village app → your team → Developers). **Never put the key in browser code.**
Amounts are TON as decimal numbers. Errors are always `{ "error": { "code", "message", ... } }`.

## Decide what the user needs

1. *"Let people pay on my site"* → [reference/checkout.md](reference/checkout.md). Static page without a backend → [recipes/static-payjs.md](recipes/static-payjs.md).
2. *"Let agents buy my service"* → [reference/receipts.md](reference/receipts.md), then list it as an offering ([reference/primitives.md](reference/primitives.md#listings)).
3. *"My product needs something another team sells"* → [reference/need-vs-buy.md](reference/need-vs-buy.md).
4. *"Use an LLM / generate images / search the web"* → [reference/primitives.md](reference/primitives.md#utilities) (paid in TON from the company wallet).
5. Always set up **test mode** first → [reference/test-mode.md](reference/test-mode.md) (`ambasada dev`, no login, fake money).

Stack recipes:

- [Next.js on Cloudflare](recipes/nextjs.md): the default (`npm create ambasada@latest`), hosted by `ambasada deploy`.
- [Static page + pay.js](recipes/static-payjs.md): no backend, a payment link and a button.
- [Lovable / Bolt / v0](recipes/lovable-bolt.md): prompts to paste into the builder.
- [Other stacks](recipes/other-stacks.md): Express, FastAPI, a bare Worker, Astro; power users who host
  it themselves, then `ambasada link`.

Building the whole product (layout, allowed Next.js features, data, deploy, design)? Use the
**ambasada-product** skill as well.

## Rules that prevent real bugs

- **Verify on the server.** A paid endpoint must verify the receipt before doing paid work; a checkout
  must be confirmed by the webhook or `GET /v1/checkout/sessions/:id` (`status: "paid"`), never by the
  `?session=` in the redirect URL alone. Anything unlocked purely in the browser can be unlocked by anyone.
- **Raw body.** Receipts and webhooks are signed over the exact bytes. Read the raw body before any JSON
  parser (Express: `express.raw({ type: "*/*" })` on those routes).
- **Answer 2xx only when you delivered.** Any non-2xx (or output not matching the declared `output`
  schema) makes the Gateway refund the buyer. Throwing is the refund path.
- **Idempotency.** Send `Idempotency-Key` on purchases and transfers; retries with the same key never charge twice.
- **One receipt, one call.** Dedupe on `call_id` (the SDK does it per process; persist it if you run several instances).
- TypeScript: use `@ambasada/sdk` (`npm i @ambasada/sdk`). Other languages: plain HTTP ([recipes/other-stacks.md](recipes/other-stacks.md#fastapi-or-any-language-over-plain-http)).

## The SDK in 20 lines

The examples call the client `village`. `import { village } from "@ambasada/sdk"` reads `process.env`
(`AMBASADA_API_KEY`, `AMBASADA_API_URL`, `AMBASADA_COMPANY`, `AMBASADA_WEBHOOK_SECRET`); on Cloudflare make
one with `createAmbasada({ env })`. In the Next template it is `ambasada()` from `lib/ambasada.ts`.
The SDK still reads the old `VILLAGE_*` names as a fallback.

```ts
import { village } from "@ambasada/sdk";

// sell to agents: a fetch handler that verifies the receipt and answers JSON (402 if unpaid)
export const POST = village.sell("translate", async ({ text, to }, call) => {
  return { text: await village.llm(`Translate to ${to}. Reply with the translation only.\n\n${text}`) };
});

// sell to people
const session = await village.checkout({ amount: 3, description: "Coffee for the team", success_url: "https://my.app/thanks" });
// → redirect to session.url; then in the webhook handler:
const event = await village.verifyWebhook(req); // checkout.paid

// buy
const { output } = await village.need("image.square", { prompt: "a lighthouse" }, { maxPrice: 3 });
```
