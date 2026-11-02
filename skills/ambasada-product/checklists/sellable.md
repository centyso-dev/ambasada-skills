# Sellable checklist

Run it before saying "done". Check each line yourself; don't assume. Every line needs `npx ambasada dev`
running, except the last section.

## The page

- [ ] `/` on a phone-sized window (375 px wide) shows, without scrolling: what it is, the price in TON, the Pay button.
- [ ] The price on the page equals the price charged (the Checkout `amount`, or the offering's `price`).
- [ ] One offer. Any second offer has its own price next to its own button.
- [ ] No login, no signup, no "coming soon", no lorem ipsum, no broken images.

## People can pay

- [ ] Pressing Pay opens the Village approve page (`/pay/cs_…` or `/pay/pl_…`); approving as the fake visitor works.
- [ ] After paying, `/thanks` shows the result, or "working on it" and then the result.
- [ ] `/thanks?session=` with a made-up or unpaid session does **not** show the result.
- [ ] The webhook route verifies the signature (`verifyWebhook`) and delivers on `checkout.paid`; a
      repeated event (same `webhook-id`) does not deliver twice.
- [ ] The Checkout `description` has no personal data (it is public in the ledger).

## Agents can pay

- [ ] At least one offering in the offerings list that `ambasada.config.ts` points to, with
      `description`, `price`, `endpoint`, `input` and `output`.
- [ ] Its route uses `sell(...)` or `verifyReceipt(...)`: a plain `curl -X POST` to it gets `402`.
- [ ] `npx ambasada dev buy <offering> '<json input>'` succeeds and the output matches `output`.
- [ ] On failure it answers non-2xx (the buyer is refunded), never 200 with an error inside.

## Safe

- [ ] `grep -rn "sk_live_\|sk_test_\|whsec_" app components lib public src web` finds nothing
      (keys live only in `.env` / `.dev.vars`), and nothing is named `NEXT_PUBLIC_*KEY*`.
- [ ] `.dev.vars` and `.env*` are in `.gitignore`.
- [ ] No banned Next.js features: `grep -rn "next/image\|revalidate\|runtime = \"edge\"\|from \"fs\"\|child_process" app lib`
      finds nothing (or only `images: { unoptimized: true }` in `next.config.ts`).

## The agent

- [ ] `agent/agent.md` says what you sell, your price range (minimum and maximum) and what the agent may buy.
- [ ] `agent/settings.json` `limits` are sane for your wallet (defaults: 10 TON per payment, 100 per day,
      5 automatic per payment).

## Online (after `npx ambasada deploy`)

- [ ] `https://<slug>.hackaton.ambasada.id` loads on a phone.
- [ ] `npx ambasada offerings list`: every endpoint starts with your public URL, none with `localhost`.
- [ ] `npx ambasada webhook test` prints `delivered`.
- [ ] One real payment by someone outside your team (members can't buy from their own company) shows up
      in `npx ambasada ledger`, and they got what they paid for.
- [ ] `npx ambasada logs --agent` shows the agent woke up.
