# Astro, or any static page: pay.js (no backend)

pay.js is served by the village (`https://hackaton.ambasada.id/pay.js`; `ambasada dev` serves the same script at
`http://127.0.0.1:8788/pay.js`, and it talks to whichever origin it was loaded from). It turns a button into
"Pay with TON" using a **payment link**, so the page never needs your key.

1. Create a link once, from your terminal:

```sh
ambasada links create --amount 3 --description "A coffee for the team" --success-url https://my.site/thanks
# → pl_… (or POST /v1/checkout/links with curl)
```

2. On the page (`src/pages/index.astro` or plain HTML):

```html
<script src="https://hackaton.ambasada.id/pay.js" defer></script>

<button data-village-pay="pl_…" data-ref="table-4">Pay 3 TON</button>
<!-- data-mode="popup" keeps the person on the page (desktop); redirect works best on phones -->

<script>
  // Fired on the button (bubbles) in popup mode, and on document when the person comes back from a redirect.
  document.addEventListener("village:paid", (e) => {
    // e.detail.session = "cs_…", e.detail.receipt = signed checkout receipt (JWT)
    document.querySelector("#thanks").hidden = false;
  });
  document.addEventListener("village:cancelled", (e) => console.log("not paid:", e.detail.reason)); // "cancelled" | "closed"
</script>
<p id="thanks" hidden>Mulțumim!</p>
```

How it flows: the button goes to `https://hackaton.ambasada.id/pay/pl_…` (`?ref=` from `data-ref`). After approving,
the person returns to the link's `success_url` with `?session=cs_…`; pay.js on that page reads
`GET /v1/web/checkout/cs_…` (public) and fires `village:paid`. In popup mode the approve page posts the result
to your page's origin and closes. Each click is a new payment; a double tap is not.

From code: `const { session, receipt } = await Village.pay("pl_…", { ref: "order-42", mode: "popup" })` (rejects
with `cancelled` or `closed`), and `await Village.session("cs_…")` → `{ status, amount, description, receipt, … }`.
`ambasada links create` makes a link; `POST /v1/checkout/links/<id>/deactivate` turns one off.

**Anything unlocked purely in the browser can be unlocked by anyone who reads your JavaScript.** If the
purchase matters (a download, credits), send `e.detail.receipt` to a server or edge function and verify it:

```ts
// e.g. an Astro endpoint src/pages/api/unlock.ts (output: "server"), or a Worker
import { village } from "@ambasada/sdk";
export const POST = async ({ request }: { request: Request }) => {
  const { receipt } = await request.json();
  const claims = await village.verifyCheckoutReceipt(receipt); // throws if forged / for another company
  return Response.json({ ok: true, amount: claims.amount });
};
```

Or set up the `checkout.paid` webhook (see reference/checkout.md).
