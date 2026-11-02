# Cloudflare Workers

`npm i @ambasada/sdk`, `wrangler secret put AMBASADA_API_KEY` (and `AMBASADA_WEBHOOK_SECRET`). Workers have no
`process.env`, so create the client from `env`:

```ts
import { createAmbasada } from "@ambasada/sdk";

export default {
  async fetch(req: Request, env: Record<string, string>): Promise<Response> {
    const village = createAmbasada({ env }); // AMBASADA_API_KEY, AMBASADA_API_URL, AMBASADA_COMPANY, AMBASADA_WEBHOOK_SECRET
    const { pathname } = new URL(req.url);
    if (pathname === "/api/translate") {
      return village.sell("translate", async ({ text, to }) => ({ text: await village.llm(`Translate to ${to}: ${text}`) }))(req);
    }
    if (pathname === "/village-webhook") {
      const event = await village.verifyWebhook(req);
      if (event.type === "checkout.paid") { /* fulfil event.data.session */ }
      return new Response("ok");
    }
    return new Response("not found", { status: 404 });
  },
};
```

Set `AMBASADA_COMPANY` to your company id to skip one API call per isolate for receipt checks. Receipt keys
are cached per isolate for 10 minutes. Bun, Deno and Next.js route handlers use the same fetch-style code.
