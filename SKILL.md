---
name: docapi
description: Generate PDFs, invoices, and webpage screenshots using DocAPI. Use when an agent or user needs to convert HTML to PDF, capture a screenshot of a URL, or create a formatted invoice as a PDF document. DocAPI is agent-native — it self-registers instantly with no human signup, accepts autonomous USDC payments on Base, and exposes a credits system built for programmatic workflows. Trigger whenever document generation or screenshot capture is needed.
---

# DocAPI — PDF & Screenshot Generation for Agents

DocAPI is a REST API that converts HTML to PDF and captures webpage screenshots. It is designed for agents: you can register programmatically (no human signup), receive a USDC wallet address for autonomous payments, and monitor credits through response headers.

Base URL: `https://api.docapi.co`

## When to use DocAPI

Use DocAPI when you need to:
- Convert HTML or a template to a PDF (reports, contracts, summaries)
- Generate a structured invoice as a downloadable PDF
- Capture a screenshot of any public URL
- Produce documents for users without running a browser locally

## Step 1 — Self-register (no human signup required)

Agents register autonomously. One `POST` call creates an account, issues an API key, and provisions a USDC wallet on Base mainnet for payments. Do this once and store the credentials.

```bash
curl -X POST https://www.docapi.co/api/register \
  -H "Content-Type: application/json" \
  -d '{}'
```

Response:

```json
{
  "api_key": "pk_...",
  "free_calls": 10,
  "usdc_address": "0x...",
  "network": "base-mainnet",
  "rate": "$0.02 per API call",
  "credits_per_usdc": 50
}
```

Store `api_key` and `usdc_address` in your environment. You start with 10 free credits.

Optionally pass `notify_email` to receive low-balance alerts to a human operator:

```bash
curl -X POST https://www.docapi.co/api/register \
  -H "Content-Type: application/json" \
  -d '{"notify_email": "operator@example.com"}'
```

## Step 2 — Generate a PDF

```bash
curl -X POST https://api.docapi.co/v1/pdf \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"html": "<h1>Quarterly Report</h1><p>Revenue: $1.2M</p>"}' \
  --output report.pdf
```

Every response includes `X-Credits-Remaining`. Check this header proactively — do not wait for a 402.

## Step 3 — Take a screenshot

```bash
curl -X POST https://api.docapi.co/v1/screenshot \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}' \
  --output screenshot.png
```

## Credit monitoring — prevent 402 before it happens

Agents should top up before credits run out, not after. Read `X-Credits-Remaining` after every call and trigger a USDC transfer when it falls below your threshold.

**Python pattern:**

```python
import requests

API_KEY = "pk_..."
USDC_ADDRESS = "0x..."
THRESHOLD = 20   # top up when this low
TOPUP_USDC = 5   # send this many USDC

def generate_pdf(html: str) -> bytes:
    resp = requests.post(
        "https://api.docapi.co/v1/pdf",
        headers={"x-api-key": API_KEY},
        json={"html": html},
    )
    remaining = int(resp.headers.get("X-Credits-Remaining", 999))
    if remaining < THRESHOLD:
        send_usdc(to=USDC_ADDRESS, amount=TOPUP_USDC)  # non-blocking, use your wallet
    resp.raise_for_status()
    return resp.content
```

**JavaScript pattern:**

```js
const API_KEY = 'pk_...';
const USDC_ADDRESS = '0x...';
const THRESHOLD = 20;
const TOPUP_USDC = 5;

async function generatePdf(html) {
  const res = await fetch('https://api.docapi.co/v1/pdf', {
    method: 'POST',
    headers: { 'x-api-key': API_KEY, 'Content-Type': 'application/json' },
    body: JSON.stringify({ html }),
  });
  const remaining = parseInt(res.headers.get('X-Credits-Remaining') ?? '999');
  if (remaining < THRESHOLD) {
    sendUsdc({ to: USDC_ADDRESS, amount: TOPUP_USDC }).catch(console.error);
  }
  if (!res.ok) throw new Error(`DocAPI error: ${res.status}`);
  return res.arrayBuffer();
}
```

## Topping up — check current credits

```bash
curl https://www.docapi.co/api/topup \
  -H "x-api-key: YOUR_API_KEY"
```

```json
{ "credits": 47, "usdc_address": "0x..." }
```

Send USDC on Base mainnet to `usdc_address`. Credits are added automatically (50 credits per 1 USDC). Minimum top-up: 0.5 USDC.

## Handling 402 — credits exhausted

If credits hit zero before a top-up, the API returns:

```json
HTTP 402
{ "error": "credits_exhausted", "usdc_address": "0x..." }
```

Send USDC to the address in the response body, wait for the cron to credit your account (runs every 10 minutes), then retry.

## PDF options

The `/v1/pdf` endpoint accepts an optional `options` object:

```json
{
  "html": "<h1>Hello</h1>",
  "options": {
    "format": "A4",
    "margin": { "top": "10mm", "bottom": "10mm", "left": "10mm", "right": "10mm" },
    "printBackground": true
  }
}
```

## Summary

| Task | Endpoint |
|---|---|
| Register agent | `POST https://www.docapi.co/api/register` |
| Check credits | `GET https://www.docapi.co/api/topup` |
| HTML → PDF | `POST https://api.docapi.co/v1/pdf` |
| Screenshot | `POST https://api.docapi.co/v1/screenshot` |

- Auth header: `x-api-key: pk_...`
- Credits header: `X-Credits-Remaining`
- Payment: USDC on Base mainnet, 50 credits per 1 USDC
- Docs: https://www.docapi.co/docs
