# Test mode: `ambasada dev`

A local mock of the whole Payments API on `http://127.0.0.1:8788`. No login, fake money, same routes and
signatures as production, so your code does not change between test and live: only `AMBASADA_API_URL` and
`AMBASADA_API_KEY` do.

```sh
npx ambasada dev                      # in your project: the mock, your offerings registered, and your site's dev server
npx ambasada dev buy translate '{"text":"Bună!","to":"en"}'                     # a fake agent pays you 1 TON
```

A project made by `npm create ambasada@latest` already has `AMBASADA_API_URL=http://127.0.0.1:8788` and
`AMBASADA_API_KEY=sk_test_dev` in its local env file. Running your site yourself? `npx ambasada dev --mock-only`,
and start it with those two variables set.

What it gives you:

- **Fake bank**: your company starts with 500 TON; `fake-agent` (key `sk_test_agent`) has 100; the fake
  visitor `p_visitor` has 100. `POST /dev/mint { to?, amount }` adds more. `ambasada wallet`, `ambasada ledger` work.
- **Real Pay proxy**: holds, calls your endpoint with a signed `x-village-receipt` (keys at
  `/.well-known/jwks.json`), captures on 2xx, refunds on failure, checks `input`/`output` schemas.
- **Checkout**: `village.checkout()` returns a `/pay/cs_…` page where the fake visitor approves; the signed
  `checkout.paid` webhook goes to your webhook URL (secret `AMBASADA_WEBHOOK_SECRET` from the local env file).
  `/pay.js` is the production script itself, so buttons, redirect and popup mode, `village:paid` /
  `village:cancelled` and `GET /v1/web/checkout/:id` behave as on the real site. Webhook test, retry and
  link / session management routes are there too.
- **Canned utilities**: `/v1/llm/chat` answers `[mock] <your last message>` (0.01 TON), `/v1/images` returns
  a placeholder image (2 TON), `/v1/tools` has `web.search`, `web.news`, `web.extract`, `voice.speech`.
- **Needs**: `image.square` is routed to the mock supplier `mockco/square-image`; `POST /dev/route
  { need, offering }` routes others.
- **Agent**: `/v1/agent/config`, `/v1/agent/steer` (canned reply), `/v1/agent/transcript`.
- **Dashboard** at `http://127.0.0.1:8788/`: wallets, your offerings with a "Buy" button, recent calls.

Any bearer key is accepted as your company except `sk_test_agent`. Going live: `ambasada login`, then
`ambasada deploy` (Village hosting sets the real key and URL for you). Hosting it yourself: put your `sk_live_…`
key in the host's env, leave `AMBASADA_API_URL` unset, then `ambasada link https://your.product`.
