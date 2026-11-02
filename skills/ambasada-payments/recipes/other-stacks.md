# Other stacks: host it yourself, then `ambasada link`

For power users who already have a server. The Village does not host these: run them anywhere with a
public HTTPS URL, then `ambasada link https://your.server` (it rewrites `product.url` in
`ambasada.config.ts` and lists your offerings there). Agents and people pay the same way; only the code differs.

Every stack needs the same four things: a paid endpoint that verifies `x-village-receipt` on the **raw
body**, a Checkout route, a webhook route that verifies the signature on the **raw body**, and the env
`AMBASADA_API_KEY`, `AMBASADA_COMPANY`, `AMBASADA_WEBHOOK_SECRET` (plus
`AMBASADA_API_URL=http://127.0.0.1:8788` against `ambasada dev --mock-only`).

## Express (Node)

`npm i @ambasada/sdk express`. The default `village` client reads `process.env`.

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

## Cloudflare Worker without a framework

The `worker` template (`npm create ambasada@latest my-shop -- --template worker`) is this, with Village
hosting by `ambasada deploy`. By hand, in your own account (`wrangler secret put AMBASADA_API_KEY`, …):

```ts
import { createAmbasada, type Village } from "@ambasada/sdk";

let village: Village | undefined; // one per isolate: it caches receipt keys and remembers used receipts

export default {
  async fetch(req: Request, env: Record<string, string>): Promise<Response> {
    village ??= createAmbasada({ env }); // AMBASADA_API_KEY, AMBASADA_API_URL, AMBASADA_COMPANY, AMBASADA_WEBHOOK_SECRET
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

Workers have no `process.env` by default, hence `createAmbasada({ env })`. Bun, Deno, Hono, Astro endpoints
(`output: "server"`) and SvelteKit use the same fetch-style handlers: `village.sell(name, handler)` is a
`(req: Request) => Promise<Response>`.

## FastAPI (or any language over plain HTTP)

`pip install fastapi uvicorn httpx pyjwt[crypto] standardwebhooks`

```python
import hashlib, json, os, httpx, jwt
from fastapi import FastAPI, Request, HTTPException
from standardwebhooks.webhooks import Webhook

API = os.environ.get("AMBASADA_API_URL", "https://hackaton.ambasada.id")
KEY = os.environ["AMBASADA_API_KEY"]
COMPANY = os.environ["AMBASADA_COMPANY"]          # your company id
HEADERS = {"Authorization": f"Bearer {KEY}"}
app = FastAPI()
_keys = {k["kid"]: jwt.PyJWK(k) for k in httpx.get(f"{API}/.well-known/jwks.json").json()["keys"]}
_seen: set[str] = set()

def verify_receipt(token: str | None, body: bytes, offering: str) -> dict:
    if not token:
        raise HTTPException(402, "payment required")
    key = _keys.get(jwt.get_unverified_header(token).get("kid"))
    if not key:
        raise HTTPException(402, "unknown village key")
    try:
        c = jwt.decode(token, key, algorithms=["EdDSA"], issuer="village")  # checks signature + exp
    except jwt.PyJWTError as e:
        raise HTTPException(402, f"bad receipt: {e}")
    if c.get("kind") or c["seller"] != COMPANY or c["offering"] != f"{COMPANY}/{offering}":
        raise HTTPException(402, "receipt is for another offering")
    if hashlib.sha256(body).hexdigest() != c["body_sha256"] or c["call_id"] in _seen:
        raise HTTPException(402, "receipt does not match this request")
    _seen.add(c["call_id"])
    return c

def llm(prompt: str) -> str:
    r = httpx.post(f"{API}/v1/llm/chat", headers=HEADERS, timeout=60,
                   json={"model": "deepseek-v4.1-flash", "messages": [{"role": "user", "content": prompt}]})
    r.raise_for_status()
    return r.json()["choices"][0]["message"]["content"]

@app.post("/api/translate")                       # offering endpoint agents pay for
async def translate(request: Request):
    body = await request.body()                   # raw bytes first
    verify_receipt(request.headers.get("x-village-receipt"), body, "translate")
    data = json.loads(body)
    return {"text": llm(f"Translate to {data['to']}. Reply with the translation only.\n\n{data['text']}")}

@app.post("/buy")                                 # people pay
def buy():
    s = httpx.post(f"{API}/v1/checkout/sessions", headers=HEADERS, json={
        "amount": 3, "description": "Coffee for the team",
        "success_url": "https://my.app/thanks?session={CHECKOUT_SESSION_ID}"}).json()
    return {"url": s["url"]}

@app.post("/village-webhook")
async def webhook(request: Request):
    body = await request.body()
    try:
        event = Webhook(os.environ["AMBASADA_WEBHOOK_SECRET"]).verify(body, dict(request.headers))
    except Exception:
        raise HTTPException(400, "bad signature")
    if event["type"] == "checkout.paid":
        fulfil(event["data"]["session"], request.headers["webhook-id"])
    return "ok"
```

Buy from another company: `httpx.post(f"{API}/v1/pay/inkwell/poem", headers={**HEADERS, "Idempotency-Key": str(uuid4()), "x-village-max-price": "2"}, json=input)`.
Errors come back as `{"error": {"code": …}}` with the HTTP status. Go, PHP, Ruby: same calls.
