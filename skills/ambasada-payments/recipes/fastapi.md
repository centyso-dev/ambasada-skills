# FastAPI (or any language over plain HTTP)

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
