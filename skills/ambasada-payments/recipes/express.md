# Express

`npm i @ambasada/sdk express` · env: `AMBASADA_API_KEY`, `AMBASADA_WEBHOOK_SECRET` (+ `AMBASADA_API_URL=http://127.0.0.1:8788` on `ambasada dev`)

```ts
import express from "express";
import { isVillageError, village } from "@ambasada/sdk";

const app = express();
const raw = express.raw({ type: "*/*", limit: "1mb" }); // receipts and webhooks are signed over the raw bytes

// An offering agents pay for.
app.post("/api/translate", raw, async (req, res) => {
  try {
    const { input, buyer, amount } = await village.verifyReceipt(req, { offering: "translate" });
    const text = await village.llm(`Translate to ${input.to}. Reply with the translation only.\n\n${input.text}`);
    res.json({ text }); // 2xx = you get paid; any error status = buyer refunded
  } catch (e) {
    if (isVillageError(e)) return res.status(e.status || 500).json(e.toJSON()); // 402 when unpaid
    throw e;
  }
});

// People pay.
app.post("/buy", express.json(), async (req, res) => {
  const session = await village.checkout({
    amount: 3,
    description: "Coffee for the team",
    success_url: "https://my.app/thanks?session={CHECKOUT_SESSION_ID}",
  });
  res.redirect(303, session.url);
});

app.post("/village-webhook", raw, async (req, res) => {
  try {
    const event = await village.verifyWebhook(req);
    if (event.type === "checkout.paid") await fulfil(event.data.session, event.id);
    res.sendStatus(200);
  } catch {
    res.sendStatus(400);
  }
});

app.listen(3000);
```

Don't put `app.use(express.json())` in front of the paid routes: once the body is parsed the bytes are gone
and `verifyReceipt` refuses with a `CONFIG` error. Full example: `examples/express-seller` in the village repo.
