# Lovable, Bolt, v0 and other no-code builders

These tools build from a prompt, so give them the docs instead of code.

**1. Static "Pay with TON" button (works everywhere, no backend):** create a payment link with
`ambasada links create …` (or in the village app → Payments), then paste this into the builder's chat:

> Add a "Pay 3 TON" button. Include `<script src="https://hackaton.ambasada.id/pay.js" defer></script>` in the
> page head and render `<button data-village-pay="pl_XXXX">Pay 3 TON</button>` (add `data-mode="popup"` to
> stay on the page). Listen for the `village:paid` DOM event on `document`; `event.detail` has `session` and
> `receipt`. When it fires, show a thank-you message; on `village:cancelled` do nothing. The page that the
> payment returns to must also load pay.js. Do not add any other payment provider.

**2. Paid features that matter (credits, downloads, a paid API) need a server.** Lovable/Bolt apps usually have
Supabase Edge Functions or similar; paste this:

> Create an edge function `village-webhook` that verifies Standard Webhooks signatures (library
> `standardwebhooks`, secret from env `AMBASADA_WEBHOOK_SECRET`) over the raw request body, and when
> `type == "checkout.paid"` marks the order `data.session.client_reference_id` as paid (dedupe on the
> `webhook-id` header). Create an edge function `buy` that POSTs to
> `https://hackaton.ambasada.id/v1/checkout/sessions` with header `Authorization: Bearer ${AMBASADA_API_KEY}`
> and JSON `{ amount, description, success_url, client_reference_id }`, then returns the `url` from the
> response for the browser to redirect to. Never expose AMBASADA_API_KEY to the browser.

Then register the webhook URL: `ambasada webhook set https://…/functions/v1/village-webhook`.

**3. Selling to agents** needs an endpoint that verifies `x-village-receipt` (reference/receipts.md). In an
edge function with npm support: `import { village } from "npm:@ambasada/sdk"` and
`Deno.serve(village.sell("translate", async (input) => ({ … })))`.

**4. MCP:** the village MCP server (`https://hackaton.ambasada.id/mcp`) works as a Lovable custom chat
connector while building; a published app calls the HTTP API with its key or uses pay.js.

If the builder supports custom knowledge files, upload this skill's `SKILL.md` and `reference/*.md`.
