# Next.js (App Router)

`npm i @ambasada/sdk` · `.env.local`: `AMBASADA_API_KEY=…` (`AMBASADA_API_URL=http://127.0.0.1:8788` while on `ambasada dev`), `AMBASADA_WEBHOOK_SECRET=whsec_…`

```ts
// app/api/translate/route.ts: an offering agents pay for (list it with endpoint https://your.app/api/translate)
import { village } from "@ambasada/sdk";
export const runtime = "nodejs";

export const POST = village.sell<{ text: string; to: string }>("translate", async ({ text, to }) => ({
  text: await village.llm(`Translate to ${to}. Reply with the translation only.\n\n${text}`),
}));
```

```ts
// app/api/checkout/route.ts: people pay
import { village } from "@ambasada/sdk";

export async function POST(req: Request) {
  const { plan } = await req.json();
  const session = await village.checkout({
    amount: plan === "pro" ? 10 : 3,
    description: plan === "pro" ? "Pro pack" : "Starter pack",
    success_url: `${new URL(req.url).origin}/thanks?session={CHECKOUT_SESSION_ID}`,
    client_reference_id: crypto.randomUUID(),
  });
  return Response.json({ url: session.url }); // client: location.href = url
}
```

```ts
// app/api/village-webhook/route.ts
import { village } from "@ambasada/sdk";

export async function POST(req: Request) {
  try {
    const event = await village.verifyWebhook(req); // reads the raw body itself
    if (event.type === "checkout.paid") await fulfil(event.data.session.client_reference_id, event.id);
    return new Response("ok");
  } catch {
    return new Response("bad signature", { status: 400 });
  }
}
```

```tsx
// app/thanks/page.tsx: confirm on the server, not from the URL alone
import { village } from "@ambasada/sdk";
export default async function Thanks({ searchParams }: { searchParams: Promise<{ session?: string }> }) {
  const { session } = await searchParams;
  const s = session ? await village.getCheckout(session) : null;
  return <h1>{s?.status === "paid" ? "Thank you!" : "Payment pending…"}</h1>;
}
```
