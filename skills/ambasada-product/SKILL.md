---
name: ambasada-product
description: Build a digital product that sells in the Hack A Ton Village and put it online. Use when the user wants to make a shop, app, tool or website that people or agents pay for in TON (the Village play currency), start one with `npm create ambasada@latest`, run it with `ambasada dev`, publish it with `ambasada deploy`, check it is ready to sell, or bring a site built in ChatGPT, Lovable, Bolt or v0 into the Village with `ambasada link`. Default stack: Next.js on Cloudflare.
---

# Build a product that sells in the Village

The Village is the Hack A Ton market. Everything is paid in **TON**: a play currency held by the Village
bank, **not** the TON blockchain. Visitors spend TON from their Ambasada ID wallet; other teams' agents
buy from you through the Gateway.

A product is **one website** (one Cloudflare Worker) at `https://<your-slug>.hackaton.ambasada.id`, plus
**your agent**, which runs the shop (prices, suppliers, deals) while you sleep.

For payment details (Checkout, receipts, webhooks, `buy`/`need`, LLM and images) use the
**ambasada-payments** skill. This skill is about the product around them.

## Start

```sh
npm create ambasada@latest my-shop          # Next.js (default)
cd my-shop && npm install
npx ambasada dev                            # fake bank + your site, no login, fake money
```

Other templates: add `-- --template spa` (Vite + React; for code exported from Lovable or Bolt) or
`-- --template worker` (plain HTML + small fetch handlers; the smallest possible product).
All three have the same `ambasada.config.ts`, `agent/` folder and four endpoints: `checkout`, `webhook`,
`receipt` and the paid offering `translate`.

## What to read

| The user wants to… | Read |
|---|---|
| know where files go, what Next.js features are allowed, where to keep data and secrets | [reference/stack.md](reference/stack.md) |
| run it locally, put it online, use their own Cloudflare account | [reference/dev-and-deploy.md](reference/dev-and-deploy.md) |
| make it look good and sell (one page, one offer) | [reference/design.md](reference/design.md) |
| build in ChatGPT, Lovable, Bolt or v0 and sell from there | [reference/build-elsewhere.md](reference/build-elsewhere.md) |
| know if it is ready | [checklists/sellable.md](checklists/sellable.md) |

## Rules (break one and the product does not work in the Village)

1. **One product = one Worker.** No separate backend, no own database, no Docker. Data goes in
   `ambasada.store` (small JSON) or `ambasada.files` (images, PDFs). See [reference/stack.md](reference/stack.md#where-data-lives).
2. **Only the allowed Next.js subset**: pages, server components, route handlers, server actions, `fetch`.
   No `next/image` default loader, no ISR/`revalidate`, no `fs`/`child_process`/native packages, no custom
   server, no `middleware.ts`/`proxy.ts`, no `runtime = "edge"`. See [reference/stack.md](reference/stack.md#nextjs-allowed-and-banned).
3. **The API key never reaches the browser.** It lives in `.env` locally (or `.dev.vars`); only server code reads it.
   Never prefix it with `NEXT_PUBLIC_`.
4. **Deliver only after payment is proven on the server**: webhook, `getCheckout(id).status === "paid"`,
   or a verified receipt. Never because the URL says `?session=…`.
5. **Sellable means**: a landing page with the price and a Pay button above the fold, one offering
   with a receipt-checked endpoint, a webhook that delivers, the agent configured. Run
   [checklists/sellable.md](checklists/sellable.md) before saying "done".

## When the user is not a developer

Do the work; don't ask them to edit code. Ask at most: what they sell, to whom, for how many TON.
Then build one page, run `npx ambasada dev`, open it, pay as the fake visitor, and show them it worked.
Explain in one sentence per step, no jargon.
