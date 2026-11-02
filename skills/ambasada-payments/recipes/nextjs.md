# Next.js on Cloudflare (the default template)

`npm create ambasada@latest my-shop` gives you all of this already: Next.js 16.4 App Router, built into one
Cloudflare Worker by `@opennextjs/cloudflare` 1.20 with `nodejs_compat`. Local secrets in `.env`
(`AMBASADA_API_KEY`, `AMBASADA_API_URL`, `AMBASADA_COMPANY`, `AMBASADA_WEBHOOK_SECRET`); `.dev.vars` works too. What runs on
Workers and what doesn't: the ambasada-product skill, `reference/stack.md`.

## The client: `lib/ambasada.ts`

Server code only. One client per Worker instance: it caches the receipt keys and remembers used receipts.

```ts
// lib/ambasada.ts (the template ships it; this is the core)
import { createAmbasada, type Village } from "@ambasada/sdk";
import { getCloudflareContext } from "@opennextjs/cloudflare";

function cloudflareEnv(): Record<string, unknown> | undefined {
  try {
    return getCloudflareContext().env as unknown as Record<string, unknown>;
  } catch {
    return undefined; // `next dev`: the SDK reads process.env
  }
}

let client: Village | undefined;

export function ambasada(): Village {
  return (client ??= createAmbasada({ env: cloudflareEnv() }));
}
```

It also exports `env(name)` (a variable from the Worker env, else `process.env`) and `villageUrl()`
(the Village, or the mock under `ambasada dev`).

## An offering agents pay for

```ts
// app/api/translate/route.ts   (offering endpoint "/api/translate")
import { ambasada } from "../../../lib/ambasada.ts";

export const POST = (req: Request) =>
  ambasada().sell<{ text: string; to: string }>("translate", async ({ text, to }) => ({
    text: await ambasada().llm(`Translate to ${to}. Reply with the translation only.\n\n${text}`),
  }))(req);
```

`sell` answers `402` without a valid receipt, runs your handler, returns its JSON. Throw a `VillageError`
(or let it fail) and the buyer is refunded.

## People pay: Checkout

```ts
// app/api/checkout/route.ts   (the landing page: <form action="/api/checkout" method="post">)
import { ambasada } from "../../../lib/ambasada.ts";

export async function POST(req: Request) {
  const origin = new URL(req.url).origin;
  const id = crypto.randomUUID();
  const a = ambasada();
  await a.store.set(`order:${id}`, { paid: false });
  const session = await a.checkout(
    {
      amount: 2,
      description: "Your name as a folk poem",            // public in the ledger: no personal data
      success_url: `${origin}/thanks?session={CHECKOUT_SESSION_ID}`,
      cancel_url: `${origin}/`,
      client_reference_id: id,
    },
    { idempotencyKey: id },
  );
  return Response.redirect(session.url, 303);
}
```

Or as a server action:

```ts
"use server";
import { redirect } from "next/navigation";
import { ambasada } from "../../../lib/ambasada.ts";

export async function buy() {
  const session = await ambasada().checkout({ amount: 2, description: "Your name as a folk poem", success_url: "…" });
  redirect(session.url);
}
```

## Deliver: the webhook

```ts
// app/api/webhook/route.ts   (`ambasada webhook set https://<slug>.hackaton.ambasada.id/api/webhook`)
import { ambasada } from "../../../lib/ambasada.ts";

export async function POST(req: Request) {
  const a = ambasada();
  let event;
  try {
    event = await a.verifyWebhook(req);                    // reads the raw body itself
  } catch {
    return new Response("bad signature", { status: 400 });
  }
  if (event.type === "checkout.paid") {
    const key = `order:${event.data.session.client_reference_id}`;
    const order = (await a.store.get(key)) as { paid: boolean } | null;
    if (order && !order.paid) await a.store.set(key, { ...order, paid: true });   // retries reuse event.id
  }
  return new Response("ok");                               // 2xx within 10 s, or it is retried
}
```

## The thanks page: check on the server

```tsx
// app/thanks/page.tsx
import { ambasada } from "../../lib/ambasada.ts";

export default async function Thanks({ searchParams }: { searchParams: Promise<{ session?: string }> }) {
  const { session } = await searchParams;
  const s = session ? await ambasada().getCheckout(session) : null;
  if (s?.status !== "paid") return <h1>Payment pending… refresh in a moment.</h1>;
  return <h1>Thank you! Paid {s.amount} TON.</h1>;
}
```

## pay.js receipts from the browser

```ts
// app/api/receipt/route.ts   (POST { receipt } from a `village:paid` event)
import { ambasada } from "../../../lib/ambasada.ts";
import { isVillageError } from "@ambasada/sdk";

export async function POST(req: Request) {
  try {
    const { receipt } = (await req.json()) as { receipt?: string };
    const claims = await ambasada().verifyCheckoutReceipt(receipt);
    return Response.json({ ok: true, amount: claims.amount, session: claims.session });
  } catch (error) {
    if (isVillageError(error)) return error.toResponse();
    return Response.json({ error: { code: "INVALID_REQUEST", message: "body must be { receipt }" } }, { status: 400 });
  }
}
```

The button side is in [static-payjs.md](static-payjs.md). In Next, the template's `app/page.tsx` loads it
with `<Script src={`${villageUrl()}/pay.js`} />` (`next/script`), so the same page works against the mock.

## Not on Cloudflare

Next on Vercel or a Node server works with the same route handlers: replace `ambasada()` with the
default client, `import { village } from "@ambasada/sdk"`, which reads `process.env`. Then `ambasada link <url>`.
