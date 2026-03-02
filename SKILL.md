---
name: docapi
description: Generate PDFs, invoices, and webpage screenshots using DocAPI. Use when an agent or user needs to convert HTML to PDF, render a document (invoice, report, contract, resume, certificate), capture a screenshot of a URL, or produce a downloadable file from structured data or templates. DocAPI is agent-native — it self-registers with no human signup, accepts autonomous USDC payments on Base mainnet, and exposes a credits system built for programmatic workflows. Trigger whenever document generation, PDF export, or screenshot capture is needed.
metadata:
  version: 1.0.0
---

# DocAPI — PDF & Screenshot Generation for Agents

You are an expert at generating professional PDFs and screenshots programmatically. Your goal is to produce high-quality documents using DocAPI, handle the agentic payment lifecycle autonomously, and avoid the common HTML-to-PDF pitfalls that produce broken output.

Base URL: `https://api.docapi.co`

---

## Initial Assessment

Before generating, understand the scope:

1. **Document type** — invoice, report, certificate, contract, resume, screenshot?
2. **Data available** — is HTML already written, or does it need to be generated from structured data?
3. **Format requirements** — A4 vs Letter, portrait vs landscape, specific margins?
4. **Credentials** — does an API key already exist in the environment (`DOCAPI_KEY`)? If not, register first.

---

## Step 1 — Self-register (no human signup required)

Agents register autonomously. One POST creates an account, issues an API key, and provisions a USDC wallet on Base mainnet. Do this once and persist the credentials.

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
  "credits_per_usdc": 50,
  "rate": "$0.02 per API call"
}
```

Store `api_key` as `DOCAPI_KEY` and `usdc_address` as `DOCAPI_USDC_ADDRESS` in your environment. You start with 10 free credits (10 API calls).

Pass `notify_email` if a human operator should receive low-balance alerts:

```bash
curl -X POST https://www.docapi.co/api/register \
  -H "Content-Type: application/json" \
  -d '{"notify_email": "operator@example.com"}'
```

---

## Step 2 — Generate a PDF

```bash
curl -X POST https://api.docapi.co/v1/pdf \
  -H "x-api-key: $DOCAPI_KEY" \
  -H "Content-Type: application/json" \
  -d '{"html": "<h1>Quarterly Report</h1><p>Revenue: $1.2M</p>"}' \
  --output report.pdf
```

**Always check `X-Credits-Remaining` in the response headers** — use it to top up before credits run out, not after.

### PDF options reference

```json
{
  "html": "...",
  "options": {
    "format": "A4",
    "landscape": false,
    "margin": { "top": "15mm", "bottom": "15mm", "left": "15mm", "right": "15mm" },
    "printBackground": true,
    "scale": 1
  }
}
```

| Option | Values | Default |
|---|---|---|
| `format` | `A4`, `Letter`, `Legal`, `Tabloid` | `A4` |
| `landscape` | `true` / `false` | `false` |
| `margin` | `top`, `bottom`, `left`, `right` in `mm` / `cm` / `in` | `10mm` all sides |
| `printBackground` | `true` / `false` | `false` |
| `scale` | `0.1` – `2` | `1` |

---

## Step 3 — Capture a screenshot

```bash
curl -X POST https://api.docapi.co/v1/screenshot \
  -H "x-api-key: $DOCAPI_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}' \
  --output screenshot.png
```

---

## HTML best practices for PDF output

⚠️ HTML-to-PDF is not the same as browser rendering. Follow these guidelines to avoid broken output.

### CSS

**Do:**
- Use inline styles or `<style>` tags in `<head>` — external stylesheets are sometimes not loaded
- Use `pt`, `mm`, `cm` for print dimensions; `px` works but is less predictable across sizes
- Set `printBackground: true` if you use background colors or images
- Use `@page` for document-level margins and orientation:
  ```css
  @page { margin: 15mm; size: A4 portrait; }
  ```
- Use `page-break-before: always` or `break-before: page` to force new pages
- Use `break-inside: avoid` on elements that must not be split (tables, cards, signatures)

**Avoid:**
- CSS `position: fixed` — works but may produce unexpected results across pages
- `vh` / `vw` units — they reference the viewport, not the page size
- CSS variables (`var()`) in complex calculations — renderer support is inconsistent
- Animations and transitions — they don't apply to static PDFs

### Fonts

- System fonts (`Arial`, `Helvetica`, `Georgia`, `Times New Roman`, `monospace`) always work
- Google Fonts: include the `<link>` tag in `<head>` — the renderer will fetch them
  ```html
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap" rel="stylesheet">
  ```
- Custom fonts: embed as base64 in a `@font-face` declaration
- Emoji: supported but rendering depends on system fonts — use sparingly in professional documents

### Images

- Use publicly accessible URLs or base64-encoded data URIs:
  ```html
  <img src="data:image/png;base64,iVBORw0KGgo..." />
  ```
- Local file paths (`/Users/...`, `file://`) will not work
- SVG is fully supported inline or as `<img src="...">`
- Large images slow rendering — compress before embedding

### Tables

- Use `<table>` for tabular data, not CSS grid — table layout is more predictable across page breaks
- Add `break-inside: avoid` to rows that must not be split:
  ```css
  tr { break-inside: avoid; }
  ```

### Page breaks

```css
/* Force a new page before an element */
.new-page { page-break-before: always; }

/* Prevent an element from being split across pages */
.keep-together { break-inside: avoid; }

/* Keep heading with its following content */
h2, h3 { break-after: avoid; }
```

---

## Invoice generation

DocAPI has a dedicated invoice endpoint that accepts structured data and returns a formatted PDF:

```bash
curl -X POST https://www.docapi.co/api/tools/generate-invoice \
  -H "Content-Type: application/json" \
  -d '{
    "companyName": "Acme Corp",
    "companyEmail": "billing@acme.com",
    "clientName": "John Smith",
    "clientEmail": "john@example.com",
    "invoiceNumber": "INV-001",
    "invoiceDate": "2026-03-01",
    "dueDate": "2026-03-31",
    "lineItems": [
      { "description": "Consulting services", "quantity": 10, "unitPrice": 150 }
    ],
    "taxRate": 8,
    "notes": "Thank you for your business."
  }' \
  --output invoice.pdf
```

No API key required for the invoice tool — it is free and rate-limited by IP.

---

## Credit monitoring — prevent 402 before it happens

Every API response includes `X-Credits-Remaining`. Monitor it and top up proactively — never wait for a 402.

**Python:**

```python
import requests, os

API_KEY = os.environ["DOCAPI_KEY"]
USDC_ADDRESS = os.environ["DOCAPI_USDC_ADDRESS"]
THRESHOLD = 20   # top up when credits drop below this
TOPUP_USDC = 5   # send this many USDC

def generate_pdf(html: str) -> bytes:
    resp = requests.post(
        "https://api.docapi.co/v1/pdf",
        headers={"x-api-key": API_KEY},
        json={"html": html},
    )
    remaining = int(resp.headers.get("X-Credits-Remaining", 999))
    if remaining < THRESHOLD:
        send_usdc(to=USDC_ADDRESS, amount=TOPUP_USDC)  # use your wallet (e.g. Coinbase AgentKit)
    resp.raise_for_status()
    return resp.content
```

**JavaScript:**

```js
const API_KEY = process.env.DOCAPI_KEY;
const USDC_ADDRESS = process.env.DOCAPI_USDC_ADDRESS;
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

---

## USDC top-up

Credits are added automatically when USDC is received on Base mainnet. Rate: **50 credits per 1 USDC** ($0.02/call).

- Send USDC on **Base mainnet** (not Ethereum mainnet, not other chains) to `DOCAPI_USDC_ADDRESS`
- Minimum meaningful top-up: 0.5 USDC (25 calls)
- Credits are credited within ~10 minutes

Check current balance anytime:

```bash
curl https://www.docapi.co/api/topup \
  -H "x-api-key: $DOCAPI_KEY"
# { "credits": 47, "usdc_address": "0x..." }
```

---

## Handling 402 — credits exhausted

If a call fails with HTTP 402:

```json
{ "error": "credits_exhausted", "usdc_address": "0x..." }
```

1. Send USDC to the `usdc_address` in the response (or from your stored `DOCAPI_USDC_ADDRESS`)
2. Poll `GET /api/topup` until credits are restored
3. Retry the original request

This should not happen if proactive monitoring is implemented. Treat 402 as a fallback, not a normal flow.

---

## Common issues

| Symptom | Likely cause | Fix |
|---|---|---|
| PDF is blank or white | Background not rendered | Add `printBackground: true` |
| Font looks wrong / fallback font used | Webfont not loaded | Use Google Fonts `<link>` or base64 `@font-face` |
| Content cut off at page boundary | No page-break handling | Add `break-inside: avoid` to affected elements |
| Image missing in PDF | Local path used | Use public URL or base64 data URI |
| PDF too small/large | Wrong format or scale | Set `format: "Letter"` or adjust `scale` |
| External CSS not applied | Remote stylesheet blocked | Inline all CSS in `<style>` tags |
| Emoji renders as boxes | Missing emoji font | Avoid emoji in professional docs or use SVG icons |

---

## Task-specific questions

Ask these before generating if not already known:

1. What format — A4 or Letter? Portrait or landscape?
2. Should background colors and images be included?
3. Are there custom fonts or branding requirements?
4. Is this a one-off or will it be called repeatedly (agent workflow)?
5. Does a human operator need low-balance email alerts?

---

## Quick reference

| Task | Endpoint |
|---|---|
| Register agent | `POST https://www.docapi.co/api/register` |
| Check credits | `GET https://www.docapi.co/api/topup` |
| HTML → PDF | `POST https://api.docapi.co/v1/pdf` |
| Webpage screenshot | `POST https://api.docapi.co/v1/screenshot` |
| Generate invoice PDF | `POST https://www.docapi.co/api/tools/generate-invoice` |

- Auth: `x-api-key: pk_...` header
- Credits header: `X-Credits-Remaining`
- Payment: USDC on Base mainnet, 50 credits per 1 USDC
- Docs: https://www.docapi.co/docs
