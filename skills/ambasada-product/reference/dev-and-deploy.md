# Run it, put it online

## `ambasada dev`: everything on your laptop, fake money

```sh
npx ambasada dev
```

Starts two things:

1. a **mock Village** on `http://127.0.0.1:8788`: same routes and signatures as the real one, a fake bank
   (your company 500 TON, the fake visitor `p_visitor` 100, `fake-agent` 100), the real `pay.js`, and an
   LLM that answers `[mock] …`. Open it in a browser for a dashboard: wallets, offerings, calls.
2. **your site** with its normal dev server (`next dev` for the `next` template), pointed at the mock.

It registers your offerings with the mock. No login, no real TON.

Try a sale:

- **as a person**: open your site, press Pay, approve as the fake visitor, land on `/thanks`.
- **as an agent**: `npx ambasada dev buy translate '{"text":"Bună!","to":"en"}'` (the fake agent pays 1 TON).

`npx ambasada dev --mock-only` starts only the mock, if you run your site yourself. The mock's keys:
`AMBASADA_API_URL=http://127.0.0.1:8788`, `AMBASADA_API_KEY=sk_test_dev` (already in the project's `.env`).
More on the mock: the ambasada-payments skill, `reference/test-mode.md`.

## `ambasada deploy`: online at `<slug>.hackaton.ambasada.id`

Once, per laptop:

```sh
npx ambasada login        # opens a page: sign in with Ambasada ID, pick your company
```

(If it asks for a key instead: the Village app → your team → Developers → Server keys, `sk_live_…`.)

Then, every time you want the new version online:

```sh
npx ambasada deploy
```

It does, in order:

1. **build**: `opennextjs-cloudflare build` (Next), `vite build` (spa) or a bundle of the Worker (worker);
2. **smoke test**: runs the built Worker locally in workerd and requests `/` and `/api/receipt`;
   a crash stops the deploy (skip with `--no-smoke` only if you know why it fails);
3. **upload** to the Village's hosting: the URL is `https://<your-company-slug>.hackaton.ambasada.id`,
   with `AMBASADA_API_KEY`, `AMBASADA_API_URL` and `AMBASADA_COMPANY` set for you;
4. **agent and offerings**: pushes `agent/agent.md` + `agent/settings.json`, and lists your offerings at the new URL.

No Cloudflare account needed. Options: `--agent-only` (push only `agent/`), `--offerings-only` (only the
offerings list), `--dry-run` (print what it would push).

After the first deploy, point the webhook at it and deploy again so the Worker gets the secret:

```sh
npx ambasada webhook set https://<slug>.hackaton.ambasada.id/api/webhook
npx ambasada webhook test        # should print "delivered"
```

Limits on Village hosting: 10 MB compressed bundle, 1000 static files, short CPU time per request
(waiting on `fetch` or the LLM does not count). Heavy work belongs in an API you call, not in the Worker.

## `ambasada deploy --own`: your own Cloudflare account

Runs `wrangler deploy` into your account (you need `npx wrangler login` first), then
`ambasada link <your workers.dev URL>`. Set the secrets yourself:

```sh
npx wrangler secret put AMBASADA_API_KEY
npx wrangler secret put AMBASADA_COMPANY
npx wrangler secret put AMBASADA_WEBHOOK_SECRET
```

## `ambasada link`: the product is hosted somewhere else

```sh
npx ambasada link https://my-thing.lovable.app
```

Sets `product.url` in `ambasada.config.ts`, checks that `<url>/health` answers, and lists your offerings
at that URL (relative endpoints like `/api/translate` become `<url>/api/translate`). Use it for Lovable,
Bolt, v0, Vercel, your own server. See [build-elsewhere.md](build-elsewhere.md).

## Watch it run

```sh
npx ambasada wallet                       # balance, held, available
npx ambasada ledger                       # every TON in and out
npx ambasada logs --agent --follow        # what your agent thinks and does
npx ambasada steer "raise translate to 1.2 TON for 2 hours"
```

## When it goes wrong

| Symptom | Fix |
|---|---|
| `deploy` prints "Product hosting is not part of `ambasada deploy` yet" | your CLI is old: `npm i -D @ambasada/cli@latest` |
| build fails on `fs`, `child_process`, a `.node` file, `sharp` | a banned package: see [stack.md](stack.md#nextjs-allowed-and-banned) |
| smoke test: `/` answers 500 | open `npx ambasada dev`, load `/`, read the terminal error |
| `CONFIG: no API key` | missing `AMBASADA_API_KEY`: `.env` locally, `ambasada login` before deploying |
| Pay works, nothing delivered | webhook not set or wrong secret: `ambasada webhook test`, then deploy again |
| agents can't buy | `ambasada offerings list`: endpoint must be your public URL, not `localhost` |
| `402` when you call your own offering | right: offerings only answer calls that come through the Gateway with a receipt; test with `ambasada dev buy` |
