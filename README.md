# Ping Me If You Can

> **Webhook‑driven API challenge for intern applicants at *SMITH\&VISION* — ping us if you can.**

![GitHub repo size](https://img.shields.io/github/repo-size/SMITH-VISION/ping-me-if-you-can)
![License](https://img.shields.io/github/license/SMITH-VISION/ping-me-if-you-can)

---

## TL;DR (30 seconds)

1. **Spin‑up a public webhook** with any free service (Cloudflare Workers, Cloudflare Tunnel, Deno Deploy … your choice).
2. **Kick‑off the handshake**:

   ```bash
   curl -X POST \
        -H "Content-Type: application/json" \
        -d '{"callbackUrl": "https://<your‑endpoint>/webhook"}' \
        https://apply.smith.vision/init
   ```
3. **Answer the challenge** our server sends to *your* endpoint within **5 minutes** using HMAC‑SHA256.
4. **Store the `registrationKey` we return** — include it as `X-Registration-Key` in every request from **Stage 2 onward**.
5. Follow the `links[].rel == "next"` field in each response until you reach `/accept`.
6. Survive every stage → automatic calendar invite for a casual chat with us!

If any step looks cryptic, keep reading — the full walkthrough (and free tooling tips) is below. 🪄

---

## About SMITH\&VISION

We are a **seed‑stage startup** delivering **computer‑vision‑driven SaaS for B2B logistics**. Our flagship application automates document‑based transactions and liberates domain expertise from everyday logistics workflows, so that people can focus on higher‑value tasks instead of manual paperwork.

*Our mission*: **「どこにでもある物流に、どこにもない価値を届ける」** — “Deliver unique value to logistics that is everywhere.”

If you feel motivated by work that affects supply chains across Japan, aspire to found your own startup someday, or simply want to pick up a wide range of skills from backend to DevOps, you’ll feel right at home here.

## Rewards & Benefits

| Item                    | Details                                                                                   |
| ----------------------- | ----------------------------------------------------------------------------------------- |
| **Compensation**        | Paid internship **¥1,800 / hour 〜** (depends on experience).                              |
| **LLM SaaS budget**     | **US\$100 / month** for ChatGPT, Gemini, Claude, etc.                                     |
| **Personal dev budget** | Up to **US\$100 / month** for books, courses, or additional cloud credits of your choice. |

---

## About this challenge

We built “Ping Me If You Can” to evaluate the real‑world skills we value most. It serves as the entry test for **frontend, backend, and infrastructure software‑engineering interns**:

* **Cloud deployment & TLS**
* **HTTP fundamentals (HATEOAS, idempotency, optimistic locking)**
* **Webhook signature verification**
* **Asynchronous & event‑driven thinking**
* **Version control & CI‑CD hygiene**

There is **no algorithm puzzle**. Your ability to ship a tiny, secure cloud service tells us far more than sorting integers ever will.

---

## Stage Map 🗺️

| Stage | You send                                                                            | We expect you to …                                                   | Fail if                                          |
| ----- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------ |
| **0** | `POST /init` with `callbackUrl`                                                     | Host a public HTTPS endpoint                                         | No TLS / non‑2xx within 5 min                    |
| **1** | Answer `/challenge/{id}` (HMAC)                                                     | Verify `X‑Signature`                                                 | Signature mismatch / timeout                     |
| **2** | `POST /profile` with **X-Registration-Key** + `Idempotency‑Key`                     | Handle duplicate requests → 409                                      | Incorrect idempotency handling                   |
| **3** | `PATCH /profile/{field}` with **X-Registration-Key** + `If‑Match` ETag              | Implement optimistic locking **and respect 0.2 req/s IP rate limit** | 412 Precondition Failed / 429 Too Many Requests  |
| **4** | **X-Registration-Key** header + Upload `resume.zip` via **presigned PUT + SHA‑256** | Resume multipart 308 / checksum OK                                   | Wrong checksum / upload stalled                  |
| **5** | `GET /events` — **SSE 1 000 ev / 10 s** (reconnect with `Last‑Event‑ID`)            | Maintain stream, reconnect ≤ 500 ms, send `/ack` after 1 000 events  | Missed events / keep‑alive dropped / ack missing |
| **6** | `POST /accept` with rotating JWT (`kid` changes every 10 min)                       | Refresh JWK cache correctly                                          | Using stale key / token expired                  |

> **Tip:** Every successful response contains a `links` array. You *must* follow those hypermedia links — hard‑coding URLs will break.

---

## Quick start ▶️

### 1 — Create your webhook in 60 seconds

#### Option A : Cloudflare Tunnel (works with any local app)

```bash
# ❶ Run your webhook locally — example using FastAPI
pip install fastapi uvicorn
python webhook.py &  # file shown further below

# ❷ Expose it
npm install -g cloudflared  # one‑time install
cloudflared tunnel --url http://localhost:8000
# → https://<random>.trycloudflare.com is now public TLS
```

#### Option B : Cloudflare Workers (TypeScript / JavaScript)

```bash
npm install -g wrangler
wrangler pages project create ping-me-worker
cd ping-me-worker
# Write index.ts, then:
wrangler deploy
# → https://ping-me-worker.<subdomain>.workers.dev
```

More free options are listed in [docs/hosting.md](docs/hosting.md).

### 2 — Verify HMAC signature

Below is a **minimal Python 3.10** FastAPI snippet that fulfils Stage 0‑1. It follows the user’s preferred typing style (`str | None`) and inline comments explain the logic.

```python
from __future__ import annotations
import hmac, hashlib
from typing import Any
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()
SHARED_SECRET: bytes = b"super-secret"  # Provided in the challenge payload


def verify(sig: str, body: bytes) -> None:  # noqa: D401 (simple verb)
    """Compare webhook signature with expected HMAC.

    Raises
    ------
    HTTPException
        401 when the signature is invalid.
    """
    expected = hmac.new(SHARED_SECRET, body, hashlib.sha256).hexdigest()
    # timing‑safe compare mitigates length‑extension & side‑channel leaks
    if not hmac.compare_digest(expected, sig):
        raise HTTPException(status_code=401, detail="invalid signature")


@app.post("/webhook")
async def webhook(
    payload: dict[str, Any],  # arbitrary JSON sent by our server
    x_signature: str | None = Header(None),
) -> dict[str, str]:
    if x_signature is None:
        raise HTTPException(status_code=400, detail="missing signature")
    verify(x_signature, str(payload).encode())
    # TODO: Handle the nonce or update logic here
    return {"status": "ok"}
```

Run it locally with `uvicorn webhook:app --port 8000` and expose via Tunnel.

### 3 — Kick off the handshake

```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"callbackUrl": "https://<your‑endpoint>/webhook"}' \
     https://apply.yourcompany.com/init
```

The response will contain `challengeId`, `nonce`, and `links[].next`.

---

## Free hosting cheat sheet

| Platform           | Free Quota (2025‑08) | Setup time | Suitable languages       |
| ------------------ | -------------------- | ---------- | ------------------------ |
| Cloudflare Workers | 100 k req/day        | ⚡ 2 min    | JS / TS / WASM           |
| Cloudflare Tunnel  | Unlimited session    | ⚡ 1 min    | Any (local → internet)   |
| Deno Deploy        | 1 M req / 100 GB mo  | ⚡ 3 min    | JS / TS / npm            |

All of them support **HTTPS & custom domains** at zero cost.

---

## Rules

* You may retry a failed stage after a **24 h cooldown**.
* Automated tools (Postman, Rest‑assured) are allowed as long as you host the webhook yourself.

---

## FAQ 🙋‍♀️🙋‍♂️

**Q. I’ve never deployed a webhook before — where should I start?**
A. Clone the FastAPI snippet above, expose it via Cloudflare Tunnel, and you’re good.

**Q. Can I use Glitch / Heroku free plan?**
A. Yes, but note that cold‑start latencies may cause timeouts in Stage 1.

**Q. Do you store my resume forever?**
A. No. All applicant data is deleted **90 days** after the task closes.

---

## License

```
The UniLicense © 2025 SMITH&VISION
```
