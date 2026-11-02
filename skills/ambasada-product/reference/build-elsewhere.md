# Built in ChatGPT, Lovable, Bolt or v0

Three ways, easiest first. Replace `pl_XXXX`, prices and names with yours.

## A. Keep it hosted there, add a Pay button

Good for: a page that sells something simple (a poster, a reading, a coffee for the team).

1. On a laptop with Node 22+, once:

   ```sh
   npm create ambasada@latest my-shop      # you need its ambasada.config.ts and agent/ folder
   cd my-shop && npm install
   npx ambasada login
   npx ambasada links create --amount 2 --description "Your name as a folk poem" --success-url https://YOUR-SITE/thanks
   # → pl_XXXX
   ```

2. Paste this into the builder's chat:

   > Add a "Pay 2 TON" button to the top of the home page, visible without scrolling on a phone.
   > Load `<script src="https://hackaton.ambasada.id/pay.js" defer></script>` on every page (in the HTML head
   > or index.html). Render the button as `<button data-village-pay="pl_XXXX">Pay 2 TON</button>`.
   > Make a `/thanks` page that also loads pay.js. Listen for the `village:paid` event on `document`
   > (`event.detail` has `session` and `receipt`): when it fires, show the result. On `village:cancelled`,
   > do nothing. Do not add Stripe or any other payment provider. Do not add login.

3. Register the site with the Village:

   ```sh
   npx ambasada link https://YOUR-SITE
   ```

   It warns if `https://YOUR-SITE/health` doesn't answer; for a page with only a Pay button that's fine.
   It also lists the template's `translate` offering at your site, which has no such endpoint: delete it
   from the offerings list first, or add `--no-sync`.

Limit: anything unlocked only in the browser can be unlocked by anyone who reads the page's code. Fine for
a thank-you and a small gift; for downloads, credits or paid AI work, use B or C.

## B. Bring the code home (recommended for Lovable and Bolt)

Good for: anything with real work behind the button. You get Village hosting and the full rules.

1. Export the project to GitHub (Lovable: GitHub sync; Bolt: download or GitHub) and clone it.
2. `npm create ambasada@latest my-shop -- --template spa` and copy the exported `src/` (components,
   pages, styles) into the new project's `src/`, replacing the starter UI. Keep the template's `api/`,
   `agent/`, `ambasada.config.ts`, `wrangler.jsonc` and `vite.config.ts`.
3. Open the project in Claude Code, Cursor or Codex and say:

   > This is a Village product (see .claude/skills/ambasada-product). Wire the Pay button in my UI to the
   > template's checkout endpoint, remove Supabase/Firebase and any login, store data with ambasada.store,
   > then run the sellable checklist.

4. `npx ambasada dev`, test a payment, then `npx ambasada deploy`.

## C. ChatGPT (or any chat) writes the code, you paste it

1. `npm create ambasada@latest my-shop`, `cd my-shop && npm install`, `npx ambasada dev`.
2. Paste this into ChatGPT, then the two files it mentions, then your idea:

   > I have a Next.js 16 App Router project that runs on Cloudflare Workers (OpenNext). Rules: no
   > database, no `fs`, no native packages, no `next/image` (use `<img>`), no middleware, no `revalidate`,
   > no login. Server code gets the payments client from `lib/ambasada.ts`; it never runs in the browser.
   > Below are my `app/page.tsx` and `app/api/checkout/route.ts`. Keep the checkout route and call it the
   > way the current page does. Rewrite `app/page.tsx` for my idea: what I sell, the price in TON and
   > the Pay button, all visible on a phone without scrolling. After paying, the buyer lands on
   > `app/thanks/page.tsx`; keep its check that the session is `"paid"` before showing the result.
   > Write complete files, one code block per file, with the file path above each.
   > My idea: …

3. Paste each file where ChatGPT says, keep `npx ambasada dev` running, check the page, pay as the fake
   visitor. When it works: `npx ambasada deploy`.

Use ChatGPT to write files, not to decide the stack: if its answer adds Prisma, Supabase, NextAuth,
`next/image` or a `middleware.ts`, tell it "that's banned in my project, do it without".

## Connect your AI tool to the Village (optional)

The Village MCP server (`https://hackaton.ambasada.id/mcp`) lets Claude, ChatGPT, Cursor or Lovable read
your wallet, create offerings and set up your agent while you build. Setup per tool: `docs/CONNECT.md`
in the Village repo. It is for building only; your published site never uses it.
