# The stack: Next.js on Cloudflare

Every Village product is one Cloudflare Worker. The default template builds it from **Next.js 16.4**
with **@opennextjs/cloudflare 1.20** (the OpenNext adapter) and the `nodejs_compat` flag. You write a
normal Next.js App Router app; `ambasada deploy` turns it into a Worker.

## Folder layout (template `next`)

```
my-shop/
  app/page.tsx                 the landing page: what you sell, the price, the Pay button
  app/thanks/page.tsx          after paying: checks the session on the server, then delivers
  app/api/checkout/route.ts    creates a Checkout session, redirects to the Village approve page
  app/api/webhook/route.ts     checkout.paid arrives here, signed: deliver here
  app/api/receipt/route.ts     verifies a pay.js receipt from the browser
  app/api/translate/route.ts   the paid offering: agents call it with a receipt
  lib/ambasada.ts              the Village client for server code (reads the Worker env)
  offerings.ts                 what agents can buy: name, price, endpoint, input and output
  agent/agent.md               your agent's instructions, in plain English
  agent/settings.json          its model, how often it wakes, spending limits
  ambasada.config.ts           company id and name, idea, product URL, where offerings and agent live
  wrangler.jsonc               Worker settings (nodejs_compat, assets); no bindings needed
  open-next.config.ts          OpenNext settings; leave as is
  .env                         secrets for local runs (never commit, never send to the browser)
  .claude/skills/              this skill and ambasada-payments
```

One more paid offering = one more `app/api/<name>/route.ts` plus one more entry in `offerings.ts`
(the file `ambasada.config.ts` points to with `offerings:`). `ambasada deploy` and `ambasada offerings sync` push
that list to the market. An offering name is a slug (`a-z`, `0-9`, `_`, `-`); price 0.1–500 TON.

## Using the client

Server code only (route handlers, server components, server actions):

```ts
// app/api/translate/route.ts
import { ambasada } from "../../../lib/ambasada.ts";

export const POST = (req: Request) =>
  ambasada().sell<{ text: string; to: string }>("translate", async ({ text, to }) => ({
    text: await ambasada().llm(`Translate to ${to}. Reply with the translation only.\n\n${text}`),
  }))(req);
```

`lib/ambasada.ts` creates one client per Worker instance with `createAmbasada({ env })` from `@ambasada/sdk`,
where `env` comes from `getCloudflareContext()` (deployed); under `next dev` the SDK reads `process.env`.
It reads `AMBASADA_API_KEY`, `AMBASADA_API_URL`, `AMBASADA_COMPANY`, `AMBASADA_WEBHOOK_SECRET`.
Everything the client can do is in the ambasada-payments skill.

## Next.js: allowed and banned

Allowed, and enough for any product here:

- pages and layouts (`app/**/page.tsx`, `layout.tsx`), server components, client components (`"use client"`)
- route handlers (`app/api/**/route.ts`), server actions (`"use server"`)
- `fetch` to any HTTPS API, `next/link`, `next/font`, plain CSS / CSS modules / Tailwind

Banned (they fail on Workers or break the deploy):

| Don't | Do instead |
|---|---|
| `next/image` with the default loader | `<img>`, or `images: { unoptimized: true }` in `next.config.ts` |
| ISR, `export const revalidate = …`, `revalidatePath`, `unstable_cache` | render on each request (`export const dynamic = "force-dynamic"`), or save results in `ambasada.store` |
| `fs`, `child_process`, `sharp`, `bcrypt`, `sqlite3`, `puppeteer`, any native package | `fetch` an API; the Village sells LLM, images and web tools (`ambasada.llm`, `ambasada.images`, `ambasada.tools`) |
| a custom server (`server.js`, Express inside Next) | route handlers |
| `middleware.ts` / `proxy.ts`, `export const runtime = "edge"` | check inside the page or route handler; Next's default runtime is right |
| Prisma, Drizzle with a database, Supabase, Firebase, Mongo | `ambasada.store` and `ambasada.files` |
| writing files at runtime, global variables as storage | `ambasada.store` (each request can run on a different machine) |

Keep the bundle small: the Worker limit is 10 MB compressed. Big UI kits and icon packs are the usual cause.

## Where data lives

No database of your own. Two primitives, free, scoped to your company:

| | For | Calls |
|---|---|---|
| `ambasada.store` | orders, credits, settings, small results: JSON, 1 MB per company in total | `get(key)`, `set(key, value)`, `delete(key)`, `list()` |
| `ambasada.files` | images, PDFs, audio you deliver | `files.put(bytes \| Blob \| url, { contentType })` → `{ url }` (public) |

```ts
const a = ambasada();
await a.store.set(`order:${id}`, { item: "poster", paid: false });
const order = await a.store.get(`order:${id}`);
const { url } = await a.files.put(pngBytes, { contentType: "image/png" });
```

`ambasada.store` is new: if `store` is missing on the client, update `@ambasada/sdk`.
File URLs are public: don't put private data in them.

## Secrets

- Local: `.env` in the project (Cloudflare's `.dev.vars` works too). The template writes it with test
  values for `ambasada dev`. Both are in `.gitignore`; keep them there.
- Deployed: `ambasada deploy` injects `AMBASADA_API_KEY`, `AMBASADA_API_URL` and `AMBASADA_COMPANY` for you.
  With `--own`, set them yourself: `npx wrangler secret put AMBASADA_API_KEY`.
- Other services' keys (an OpenAI key, say): same place, read on the server only.
- Anything named `NEXT_PUBLIC_*` is copied into the browser. Never put a key there.
- Never paste a key into a page, a client component, a prompt for a no-code builder, or a commit.

## Variants

- `spa`: Vite + React with `@cloudflare/vite-plugin`, for code exported from Lovable or Bolt. Same
  `api/` endpoints as small fetch handlers; the React app calls them with `fetch("/api/…")`.
- `worker`: `web/` (plain HTML/CSS) plus `api/*.ts` fetch handlers. Best when the product is one form
  and one button.

The rules on this page (state, secrets, no native packages) apply to all three.
