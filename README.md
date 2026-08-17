# WasapFlow Bridge — API Reference

**Version:** 2.6.0  
**Last Updated:** 11 August 2026  
**Base URL:** `https://officialapi.wasapflow.com/bridge/v1`  
**Auth Header:** `x-partner-key: wf_your_key`

> ⚠️ **Meta pricing change — effective 1 October 2026.** Service messages (non-template replies inside the 24-hour customer service window) and utility templates sent inside that window become billable. **How you send does not change** — service messages still need no template and no Meta pre-approval. Only the billing changes. See the [Changelog](?tab=changelog) for what to do now.

> **What's new in 2.2.0:** Every message status webhook now carries a `pricing` object so you can attribute Meta's cost per message, plus a `conversation` object. Every API response carries [notice headers](#change-notices), and every webhook carries a `meta` object pointing at the changelog. All additive — no existing field changed.
>
> **2.1.0:** Template lifecycle + account webhooks — `template.status_updated` (approved/rejected), `template.quality_updated`, `template.category_updated`, `waba.account_updated`, `waba.review_updated`, and `contact.synced` (Coexistence contact sync). See [Webhook Event Payloads](#webhook-event-payloads).
>
> **2.0.0:** Coexistence message sync — `message.echo` (messages your client sends from the WhatsApp Business App) and `message.history` (past conversations backfilled after onboarding).

---

## How It Works

Your system connects directly to Meta's WhatsApp Cloud API — through WasapFlow's infrastructure.

```
Your System  →  Bridge API (officialapi.wasapflow.com)  →  Meta
```

You do not need a Meta Tech Provider account. You do not need a Meta App. Your clients register their WhatsApp Business Account (WABA) through WasapFlow's Embedded Signup flow, and you manage those WABAs directly via this API.

---

## Connection Modes

WasapFlow Bridge currently supports **Coexistence mode** for Embedded Signup.

**Coexistence mode** means your client can connect a number that is already active in the **WhatsApp Business App** while continuing to use that same Business App number. WasapFlow connects the number to WhatsApp Cloud API for automation, templates, webhooks, and API sending, without asking the client to stop using the WhatsApp Business App.

Use Coexistence when:
- The client already uses the WhatsApp Business App on their phone.
- They want to keep replying manually from the Business App.
- Your SaaS also needs API access through WasapFlow Bridge.

Do not use Coexistence for:
- WhatsApp personal app numbers.
- Numbers already fully migrated to Cloud API only.
- Manual token registration where the client gives you WABA ID, Phone Number ID, and token directly.

For Embedded Signup, set:

```json
{
  "connection_mode": "coexistence"
}
```

If you self-host the Meta popup, launch it with:

```json
{
  "setup": {},
  "featureType": "whatsapp_business_app_onboarding",
  "sessionInfoVersion": "3",
  "version": "v4"
}
```

If you use the WasapFlow hosted popup (`/bridge/connect`), this is already handled for you.

---

## Where to Get Your Credentials

| Credential | Location in Dashboard |
|------------|----------------------|
| **Partner Key** | **Dashboard** → API Credentials → Partner Key (👁️ eye icon to reveal, 📋 copy button) |
| **Webhook Secret** | **Settings** → Webhook Security → Webhook Secret (👁️ eye icon to reveal, 📋 copy button) |
| **Base URL** | **Dashboard** → API Credentials → Base URL |

---

## Authentication

Include your Partner Key in every request:

```http
x-partner-key: wf_your_partner_key
```

For WABA-scoped endpoints, also include the WABA ID:

```http
x-partner-key: wf_your_partner_key
x-waba-id: 123456789
```

---

## Response Format & HTTP Status Codes

Every Bridge API response is JSON with a top-level `success` field:

**Success response:**
```json
{ "success": true, ...endpoint-specific fields }
```
HTTP status: `200 OK` (or `201 Created` for resource creation)

**Error response:**
```json
{
  "success": false,
  "error": {
    "code": "WABA_NOT_REGISTERED",
    "message": "WABA not registered with your partner account"
  }
}
```
HTTP status: appropriate `4xx` or `5xx` based on the error category.

### HTTP Status Code Mapping

The Bridge API follows standard REST semantics — your fetch wrappers, monitoring tools, and retry middleware can rely on `res.ok` / 4xx-5xx detection.

| HTTP Status | Category | Example error codes |
|------------|----------|---------------------|
| `400 Bad Request` | Client sent malformed/missing data | `MISSING_FIELDS`, `MISSING_WABA`, `INVALID_PAYLOAD`, `CODE_EXCHANGE_FAILED` |
| `401 Unauthorized` | Missing or invalid `x-partner-key` | `UNAUTHORIZED`, `INVALID_PARTNER_KEY` |
| `402 Payment Required` | Billing issue blocks the request | `TRIAL_EXPIRED`, `PAYMENT_FAILED`, `SUBSCRIPTION_CANCELLED` |
| `403 Forbidden` | Authenticated but not allowed | `WABA_LIMIT_REACHED`, `TOO_MANY_CONTACTS`, `PARTNER_SUSPENDED` |
| `404 Not Found` | Resource doesn't exist | `WABA_NOT_REGISTERED`, `NOT_FOUND`, `NO_ACTIVE_WABA`, `MEDIA_NOT_FOUND` |
| `409 Conflict` | State conflict | `CANNOT_CANCEL` (broadcast already started) |
| `429 Too Many Requests` | Rate / quota exceeded | `RATE_LIMIT_EXCEEDED`, `QUOTA_EXCEEDED` |
| `500 Internal Server Error` | WasapFlow-side failure | `SERVER_ERROR`, `INTERNAL_ERROR` |
| `502 Bad Gateway` | Upstream Meta API error | `META_ERROR` (check `meta_code` and `meta_message`) |
| `503 Service Unavailable` | Platform not configured | `PLATFORM_NOT_CONFIGURED`, `NOT_CONFIGURED` |

### Recommended client handling

```javascript
const res = await fetch(url, opts);

if (!res.ok) {
    // 4xx / 5xx — parse error body
    const { error } = await res.json();
    throw new BridgeError(error.code, error.message, res.status);
}

const body = await res.json();
// Defensive check (legacy / unknown error codes still set success=false)
if (body.success === false) {
    throw new BridgeError(body.error.code, body.error.message, res.status);
}

return body;
```

> **Migration note (May 2026):** Before v1.4.5 the Bridge API returned `200 OK` even for failed responses. From v1.4.5 onwards, failed responses return appropriate 4xx/5xx codes. Your existing code checking `body.success === false` continues to work, but new code can now rely on HTTP status checks as well.

---

## Change Notices

Meta changes its platform on a fixed calendar — rates can move only on 1 January, 1 April, 1 July, or 1 October. When something is coming that affects you, we tell you through **every response**, so you never have to poll a page or monitor Meta's docs yourself.

**Every API response carries these headers:**

| Header | Value |
|--------|-------|
| `X-Bridge-Api-Version` | Current Bridge API version, e.g. `2.2.0` |
| `X-Bridge-Changelog` | URL of the changelog |
| `X-Bridge-Notice` | Comma-separated notice IDs (header absent when there are none) |
| `X-Bridge-Notice-Level` | Highest severity: `info`, `action_required`, or `breaking` |

**Every webhook carries the same information** in a `meta` object — see [Webhook Envelope](#webhook-envelope).

```javascript
const res = await fetch(url, opts);

const level = res.headers.get('X-Bridge-Notice-Level');
if (level === 'action_required' || level === 'breaking') {
    logger.warn('Bridge platform notice', {
        ids:       res.headers.get('X-Bridge-Notice'),
        level,
        changelog: res.headers.get('X-Bridge-Changelog')
    });
}
```

Notice IDs are stable forever, so you can suppress ones you have already acted on. Headers are purely additive — response bodies are unchanged, so strict body parsers and schema validators are unaffected.

Full detail for every notice, plus the Meta platform calendar: **[Changelog](?tab=changelog)** (mirrored at [github.com/kobaranteguh/changelog](https://github.com/kobaranteguh/changelog)).

---

## Quick Start (Node.js)

```bash
# Install SDK
npm install github:kobaranteguh/wasapflow-bridge-node   # Node.js
pip install git+https://github.com/kobaranteguh/wasapflow-bridge-python.git  # Python
composer require kobaranteguh/wasapflow-bridge-php      # PHP
```

```javascript
const { WasapFlowBridge } = require('wasapflow-bridge');

const bridge = new WasapFlowBridge({
    partnerKey: process.env.WF_PARTNER_KEY,
    webhookSecret: process.env.WF_WEBHOOK_SECRET
});

// Register your client's WABA (from Embedded Signup)
// Note: SDK uses camelCase — raw HTTP API uses snake_case (see Endpoints section)
await bridge.clients.register({
    wabaId: '123456789',
    phoneNumberId: '987654321',
    accessToken: 'EAAxxxxxxxx',
    displayName: 'My Client Business'
});

// Send a message
const waba = bridge.client('123456789');
await waba.messages.send({ to: '60123456789', text: 'Hello!' });
```

---

## Field Naming Convention

> **Important:** The raw HTTP API uses **snake_case** for all request/response fields (`waba_id`, `phone_number_id`, `access_token`). The SDKs use **camelCase** internally (`wabaId`, `phoneNumberId`, `accessToken`) and convert automatically. If you call the API directly without an SDK, always use snake_case.

---

## Endpoints

### Client (WABA) Management

#### Register from Embedded Signup Code (Recommended)
```http
POST /clients/register-from-code
x-partner-key: wf_xxx
Content-Type: application/json
```
Use this after your client completes the Meta Embedded Signup flow. Send the `code` from `FB.login` — WasapFlow exchanges it server-side and registers the WABA. **Your app never sees the Meta access token.**

For Coexistence onboarding, include `"connection_mode": "coexistence"`. This tells WasapFlow that the client is connecting an existing WhatsApp Business App number and intends to keep the Business App usable.

```json
{
  "code": "AQB8x...",
  "display_name": "My Client Business",
  "connection_mode": "coexistence"
}
```

**Response:**
```json
{
  "success": true,
  "client": {
    "waba_id": "123456789",
    "phone_number_id": "987654321",
    "display_name": "My Client Business",
    "connection_mode": "coexistence",
    "status": "active",
    "quality_rating": "GREEN",
    "registered_at": "2026-05-15T10:00:00.000Z"
  }
}
```

**Error responses:**

| Error Code | Meaning |
|------------|---------|
| `MISSING_FIELDS` | `code` not provided |
| `PLATFORM_NOT_CONFIGURED` | Meta App credentials not set up on WasapFlow platform |
| `CODE_EXCHANGE_FAILED` | Code expired or invalid — client needs to redo Embedded Signup |
| `DISCOVERY_FAILED` | Could not find WABA/Phone from the signup — client may not have completed the flow |
| `WABA_LIMIT_REACHED` | Partner exceeded max WABAs on their plan |

---

#### Create a Connect Session (2.6.0)
```http
POST /connect/session
x-partner-key: wf_xxx
Content-Type: application/json

{
  "display_name": "USRAA Skin Centre",
  "state": "your-own-client-id-123"
}
```

Server-to-server. Exchanges your partner key for a **short-lived link** you can
safely open in your client's browser.

**Response:**
```json
{
  "success": true,
  "connect_url": "https://officialapi.wasapflow.com/bridge/connect?token=93d29ead…",
  "token": "93d29ead…",
  "expires_in": 1800
}
```

Send your client to `connect_url`. Nothing else is needed — the page handles the
Meta popup, the Coexistence extras and the code exchange.

| Field | Notes |
|---|---|
| `display_name` | Optional. Your client's business name, shown during onboarding |
| `state` | Optional, max 255 chars. Opaque to us. Returned to you unchanged when the connection completes, so you can match it to your own client record |
| `expires_in` | Seconds. 30 minutes. The link may be reloaded within that window |

**Why this exists — please migrate.** The older form,
`/bridge/connect?partner_key=wf_live_…`, puts your **full API credential** in a
browser URL. From there it reaches the address bar, browser history, the
`Referer` header sent to other hosts, and our own access logs in plain text.
Anyone who reads any one of those gets complete control of your partner account —
every WABA, every send, every client.

The old form still works and we have not set a removal date, so nothing breaks
today. But a `partner_key` that has been through a browser should be treated as
exposed: rotate it in the partner portal after you migrate.

**Getting `state` back:**

```js
window.addEventListener('message', (e) => {
  if (e.data?.type === 'WASAPFLOW_CONNECT_SUCCESS') {
    e.data.state;    // "your-own-client-id-123"
    e.data.waba_id;
  }
});
```

It is also in the `/clients/register-from-code` response body as `state`.

> Before 2.6.0 the connect page accepted a `state` query parameter and silently
> discarded it. If you were passing one and wondering why it never came back,
> that is why.

---

#### Get Embedded Signup Config
```http
GET /embedded-signup/config
x-partner-key: wf_xxx
```
Returns the Meta App ID and Config ID needed for your frontend `FB.init()` and `FB.login()`. No secrets are exposed.

The response also includes the recommended Coexistence `extras`. If you are self-hosting the Facebook SDK popup, pass these values into `FB.login`. If you use WasapFlow hosted popup, you do not need to handle these extras yourself.

**Response:**
```json
{
  "success": true,
  "app_id": "123456789012345",
  "config_id": "987654321098765",
  "connection_mode": "coexistence",
  "extras": {
    "setup": {},
    "featureType": "whatsapp_business_app_onboarding",
    "sessionInfoVersion": "3",
    "version": "v4"
  }
}
```

> Also available in your **Partner Dashboard → Settings** page.

---

#### Register a WABA (Manual)
```http
POST /clients/register
x-partner-key: wf_xxx
Content-Type: application/json
```
> Use `register-from-code` instead when possible. This endpoint requires you to handle the Meta access token yourself. Performs an **upsert** — safe to call again if credentials change.

```json
{
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "access_token": "EAAxxxxxxxx",
  "display_name": "My Client Business"
}
```
**Response:**
```json
{
  "success": true,
  "client": {
    "waba_id": "123456789",
    "phone_number_id": "987654321",
    "display_name": "My Client Business",
    "status": "active",
    "tier": "STANDARD",
    "quality_rating": "GREEN",
    "registered_at": "2026-05-14T10:00:00.000Z"
  }
}
```

---

#### List WABAs
```http
GET /clients?limit=100&offset=0
x-partner-key: wf_xxx
```

| Parameter | Default | Max |
|-----------|---------|-----|
| `limit` | `100` | `500` |
| `offset` | `0` | — |

**Response:**
```json
{
  "success": true,
  "clients": [
    {
      "waba_id": "123456789",
      "phone_number_id": "987654321",
      "display_name": "My Client Business",
      "status": "active",
      "tier": "STANDARD",
      "quality_rating": "GREEN",
      "messaging_limit_tier": "TIER_2K",
      "throughput_level": "STANDARD",
      "registered_at": "2026-05-14T10:00:00.000Z",
      "last_activity_at": "2026-05-14T12:00:00.000Z"
    }
  ],
  "total": 1,
  "limit": 100,
  "offset": 0
}
```

---

#### Remove a WABA
```http
DELETE /clients/:wabaId
x-partner-key: wf_xxx
```
**Response:**
```json
{ "success": true }
```
> The WABA is marked as `deleted` — messaging stops immediately. This action cannot be undone via API; contact support to restore.

---

#### Refresh WABA Quality & Tier
```http
POST /clients/:wabaId/refresh
x-partner-key: wf_xxx
Content-Type: application/json
```
Fetches latest quality rating, messaging limit and throughput from Meta. Optionally update the stored access token in the same call.

**Body (optional):**
```json
{ "access_token": "EAAxxxxxxxx" }
```

**Response:**
```json
{
  "success": true,
  "quality_rating": "GREEN",
  "throughput_level": "STANDARD",
  "messaging_limit_tier": "TIER_2K",
  "token_updated": false
}
```

> **Messaging limit values:** `TIER_250`, `TIER_1K`, `TIER_2K`, `TIER_10K`, `TIER_100K`, `UNLIMITED`

---

#### Reconnect Meta Webhook
```http
POST /clients/:wabaId/resubscribe-webhook
x-partner-key: wf_xxx
```
Re-subscribes a WABA to WasapFlow's Meta webhook endpoint. Use this when webhook events stop arriving without re-doing the Embedded Signup flow.

**Response:**
```json
{ "success": true, "message": "Webhook resubscribed successfully" }
```

> Safe to call at any time — does not affect messaging or WABA configuration.

---

### Messages

All message endpoints require both headers:
```http
x-partner-key: wf_xxx
x-waba-id: 123456789
```

> **Phone number format:** Always use international format without `+`. Example: `60123456789` (not `+60123456789`).

All successful message sends return the Meta **wamid**, under two keys holding
the same value — read whichever you prefer:

```json
{
  "success": true,
  "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgASGBQzQUVEMUFBQkVBRUQ5NTJERTA",
  "messages": [ { "id": "wamid.HBgNNjAxMjM0NTY3ODkVAgASGBQzQUVEMUFBQkVBRUQ5NTJERTA" } ]
}
```

`message_id` has been there since 1.0. `messages[0].id` was added in 2.5.0
because it is the shape Meta's own Cloud API returns, and partners arriving
from Meta's documentation look for it there first.

**Store this id.** Every delivery callback (`message.sent`, `message.delivered`,
`message.read`, `message.failed`) is keyed on it. Without it you cannot tell
which message a status belongs to, and a `message.failed` — the only way you
learn a send did not arrive — has nothing to attach to.

---

#### Send Text Message
```http
POST /messages/send
```
```json
{
  "to": "60123456789",
  "text": "Hello from my system!",
  "preview_url": false
}
```

##### Recipient: `to`, `user_id` or `recipient` (2.5.0)

Every send endpoint takes **either** a phone number **or** a BSUID:

| Field | Value | Use when |
|-------|-------|----------|
| `to` | Phone number **or** BSUID | The normal case. A BSUID here is detected and routed correctly — you do not have to branch |
| `user_id` | BSUID, e.g. `MY.2026206508304015` | You want an explicit field rather than relying on us detecting the shape |
| `recipient` | BSUID | Alias of `user_id`, kept from 2.4.0 |

All three of these deliver to the same person:

```json
{ "to": "MY.2026206508304015", "text": "Terima kasih!" }
{ "user_id": "MY.2026206508304015", "text": "Terima kasih!" }
{ "recipient": "MY.2026206508304015", "text": "Terima kasih!" }
```

**When a phone number and a BSUID are both supplied, the phone number wins.** That is Meta's own precedence rule. Existing integrations that send a phone number in `to` are unaffected, byte for byte.

Supplying neither returns `MISSING_FIELDS`.

> **Fixed in 2.5.0 — if you are on 2.4.0, this affects you.** In 2.4.0 a BSUID placed in `to` was forwarded to Meta's *phone number* field. Meta stripped it to digits, so `MY.1777397056778201` became `1777397056778201` — a number belonging to nobody — and the send failed with `131026 Message undeliverable` **after** we had already returned `2xx`. Only `recipient` worked. From 2.5.0 all three field names work, and a BSUID is never treated as a phone number. Click-to-WhatsApp ad leads are the common case here: those users often have no phone number at all.

> **Response shape differs.** Sending to a BSUID returns `contacts[0].user_id` and **no `wa_id`**. If you read `wa_id` from a send response, it will be `undefined` for BSUID sends — read `user_id` instead.

> **Not accepted for authentication templates.** Meta rejects BSUID recipients for one-tap, zero-tap and copy-code authentication templates with error `131062`. This is permanent: OTP flows always need a real phone number.

---

#### Send Template Message
```http
POST /messages/template
```

**Simple template (body params only):**
```json
{
  "to": "60123456789",
  "template": {
    "name": "order_confirmed",
    "language": "en_US",
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "John Doe" },
          { "type": "text", "text": "RM150.00" }
        ]
      }
    ]
  }
}
```

**Template with header image:**
```json
{
  "to": "60123456789",
  "template": {
    "name": "promo_with_image",
    "language": "ms",
    "components": [
      {
        "type": "header",
        "parameters": [
          { "type": "image", "image": { "link": "https://yourserver.com/promo.jpg" } }
        ]
      },
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Ahmad" },
          { "type": "text", "text": "30%" }
        ]
      }
    ]
  }
}
```

**Template with call-to-action buttons:**
```json
{
  "to": "60123456789",
  "template": {
    "name": "order_with_tracking",
    "language": "en_US",
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "ORD-12345" }
        ]
      },
      {
        "type": "button",
        "sub_type": "url",
        "index": "0",
        "parameters": [
          { "type": "text", "text": "ORD-12345" }
        ]
      }
    ]
  }
}
```

> **Template categories:** `MARKETING`, `UTILITY`, `AUTHENTICATION`  
> Templates must be approved in Meta Business Manager before use.

---

#### Send Media (Image / Document / Audio / Video)
```http
POST /messages/media
```

**Send by URL:**
```json
{
  "to": "60123456789",
  "type": "image",
  "media": {
    "link": "https://yourserver.com/image.jpg",
    "caption": "Your receipt"
  }
}
```

**Send by media_id (from `/media/upload`):**
```json
{
  "to": "60123456789",
  "type": "image",
  "media": {
    "id": "1234567890",
    "caption": "Your receipt"
  }
}
```

**Supported types and size limits:**

| Type | Formats | Max Size |
|------|---------|----------|
| `image` | JPG, PNG | 5 MB |
| `document` | PDF, DOCX, XLSX, etc | 100 MB |
| `audio` | MP3, OGG, AAC | 16 MB |
| `video` | MP4, 3GP | 16 MB |

---

#### Send Interactive Buttons
```http
POST /messages/interactive
```

**Reply buttons (max 3):**
```json
{
  "to": "60123456789",
  "interactive": {
    "type": "button",
    "body": { "text": "Choose an option:" },
    "action": {
      "buttons": [
        { "type": "reply", "reply": { "id": "confirm", "title": "Confirm Order" } },
        { "type": "reply", "reply": { "id": "cancel", "title": "Cancel" } }
      ]
    }
  }
}
```

**List message (for menus, max 10 items per section):**
```json
{
  "to": "60123456789",
  "interactive": {
    "type": "list",
    "body": { "text": "Select a department:" },
    "action": {
      "button": "View Options",
      "sections": [
        {
          "title": "Support",
          "rows": [
            { "id": "billing", "title": "Billing", "description": "Payment & invoices" },
            { "id": "technical", "title": "Technical", "description": "Product issues" }
          ]
        }
      ]
    }
  }
}
```

> Using the SDK? `waba.messages.buttons({ to, body, buttons })` and `waba.messages.list({ to, body, button, sections })` build these automatically.

---

#### Send Location
```http
POST /messages/location
```
```json
{
  "to": "60123456789",
  "latitude": 3.1390,
  "longitude": 101.6869,
  "name": "Kuala Lumpur City Centre",
  "address": "Jalan Ampang, KL"
}
```

---

#### React to a Message
```http
POST /messages/reaction
```
```json
{
  "to": "60123456789",
  "message_id": "wamid.xxx",
  "emoji": "👍"
}
```

---

#### Mark Message as Read
```http
POST /messages/read
```
```json
{
  "message_id": "wamid.xxx"
}
```

---

### Templates

#### List Templates
```http
GET /templates
x-partner-key: wf_xxx
x-waba-id: 123456789
```

**Response:**
```json
{
  "success": true,
  "templates": [
    {
      "id": "123456789012345",
      "name": "order_confirmed",
      "language": "en_US",
      "category": "UTILITY",
      "status": "APPROVED",
      "components": [
        { "type": "BODY", "text": "Hello {{1}}..." }
      ]
    }
  ],
  "total": 1
}
```

| Field | Type | Note |
|---|---|---|
| `id` | string | Meta template id (numeric string) |
| `name` | string | Template name (unique per WABA) |
| `language` | string | e.g. `en_US`, `ms`, `id` |
| `category` | string | `MARKETING` / `UTILITY` / `AUTHENTICATION` |
| `status` | string | `APPROVED` / `PENDING` / `REJECTED` / `PAUSED` / `DISABLED` / `IN_APPEAL` |
| `components` | array | Meta component objects (BODY, HEADER, FOOTER, BUTTONS) |

> **Quality + rejection reason:** Not returned by this endpoint. For rejection reason, subscribe to webhook `template.status_updated` (carries `reason`). For quality changes, subscribe to `template.quality_updated` (carries `previous_quality` + `new_quality`).

> **Wrapping convention:** All list endpoints wrap items under a key named after the resource (lowercase plural) — never a bare array. `/clients` → `clients[]`, `/templates` → `templates[]`, `/broadcasts` → `broadcasts[]`, `/events` → `events[]`.

---

#### Create Template
```http
POST /templates
x-partner-key: wf_xxx
x-waba-id: 123456789
```

**Positional variables `{{1}}`, `{{2}}` (default):**
```json
{
  "name": "order_confirmed",
  "language": "en_US",
  "category": "UTILITY",
  "parameter_format": "POSITIONAL",
  "components": [
    {
      "type": "BODY",
      "text": "Hello {{1}}, your order {{2}} has been confirmed.",
      "example": {
        "body_text": [["Ali", "ORD123"]]
      }
    }
  ]
}
```

**Named variables `{{customer_name}}`:**
```json
{
  "name": "welcome_named",
  "language": "en_US",
  "category": "UTILITY",
  "parameter_format": "NAMED",
  "components": [
    {
      "type": "BODY",
      "text": "Hello {{customer_name}}, order {{order_id}} ready.",
      "example": {
        "body_text_named_params": [
          { "param_name": "customer_name", "example": "Ali" },
          { "param_name": "order_id",      "example": "ORD123" }
        ]
      }
    }
  ]
}
```

**Document (PDF) header — requires `header_handle` from `/templates/upload-header`:**
```json
{
  "name": "repair_ticket",
  "language": "en_US",
  "category": "UTILITY",
  "components": [
    {
      "type": "HEADER",
      "format": "DOCUMENT",
      "example": { "header_handle": ["4::aW1hZ2..."] }
    },
    {
      "type": "BODY",
      "text": "Your repair ticket {{1}} is ready.",
      "example": { "body_text": [["TKT123"]] }
    }
  ]
}
```

> **Categories:** `MARKETING`, `UTILITY`, `AUTHENTICATION`
> **`parameter_format`** is optional (default `POSITIONAL`). Set to `NAMED` for `{{name}}` style — must be at TOP level, not inside the component.
> Variable templates **must include `example` values** — Meta rejects approval otherwise.
> Template approval may take up to 24 hours. Status changes will appear in Meta Business Manager.

---

#### Upload Template Header Sample (for IMAGE/VIDEO/DOCUMENT headers)
```http
POST /templates/upload-header
x-partner-key: wf_xxx
x-waba-id: 123456789
Content-Type: application/json
```
```json
{
  "url": "https://your-cdn.com/sample.pdf",
  "mime_type": "application/pdf"
}
```

**Response:**
```json
{
  "success": true,
  "header_handle": "4::aW1hZ2UvanBlZw==:..."
}
```

> **Why a separate endpoint?** Meta requires a sample file when creating a template with a media header (IMAGE / VIDEO / DOCUMENT). That sample is uploaded via Meta's **Resumable Upload API** against the platform `app_id`, returning a `header_handle` string — **different** from the numeric `media_id` returned by `/media/upload`.
>
> | Field | Endpoint | When | Bound to |
> |---|---|---|---|
> | `header_handle` (string) | `POST /templates/upload-header` | Template approval (one-time) | `app_id` |
> | `media_id` (numeric) | `POST /media/upload` | Sending approved template (every send) | `phone_number_id`, expires 30 days |
>
> Use the returned `header_handle` in `components[].example.header_handle[]` when calling `POST /templates`.
>
> **Limits:** image 5 MB, video 16 MB, document 100 MB. Source `url` must be HTTPS and publicly reachable.

---

#### Delete Template
```http
DELETE /templates/:name
x-partner-key: wf_xxx
x-waba-id: 123456789
```

---

### Business Profile

#### Get Profile
```http
GET /profile
x-partner-key: wf_xxx
x-waba-id: 123456789
```

---

#### Update Profile
```http
PUT /profile
x-partner-key: wf_xxx
x-waba-id: 123456789
```
```json
{
  "about": "We sell quality products",
  "address": "Kuala Lumpur, Malaysia",
  "email": "hello@mybusiness.com",
  "websites": ["https://mybusiness.com"]
}
```

---

### Contacts

#### Check if Phone is on WhatsApp
```http
GET /contacts/:phone
x-partner-key: wf_xxx
x-waba-id: 123456789
```
**Response:**
```json
{ "phone": "60123456789", "exists": true, "waId": "60123456789" }
```

---

#### Download Incoming Media
```http
GET /contacts/media/:mediaId
x-partner-key: wf_xxx
x-waba-id: 123456789
```
Returns the media file as binary stream. Use the `media_id` from the `message.received` webhook payload.

> **Note:** Media IDs from incoming messages expire after 30 days. Download and store them on your own server if needed long-term.

---

### Media

#### Upload Media (Get a media_id)

This endpoint is **URL-based, not a multipart file receiver**. Host the file at a publicly reachable HTTPS URL — Bridge downloads it and uploads it to Meta on your WABA's behalf.

```http
POST /media/upload
x-partner-key: wf_xxx
x-waba-id: 123456789
Content-Type: application/json
```
```json
{
  "url": "https://your-server.com/files/receipt.png",
  "mime_type": "image/png"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `url` | ✅ | Public **HTTPS** URL of the file. Bridge fetches it server-side (30s timeout). |
| `mime_type` | ✅ | Full MIME type, e.g. `image/png`, `image/jpeg`, `application/pdf`, `video/mp4`. |

**Response:**
```json
{ "success": true, "media_id": "1234567890", "waba_id": "123456789" }
```
Use the returned **`media_id`** when sending: pass it to `POST /messages/media` as `media.id`.

**Limits:** image 5 MB · audio/video 16 MB · document 100 MB.

> **Media IDs** are bound to the WABA's phone number and **expire after ~30 days** on Meta's side — re-upload if stale.

> ⚠️ Sending `multipart/form-data` to this endpoint returns `400 UNSUPPORTED_CONTENT_TYPE`. (Historical note: docs previously showed a multipart request shape — that was never how this endpoint worked.)

---

### Analytics

#### Get WABA Analytics
```http
GET /analytics?days=7
x-partner-key: wf_xxx
x-waba-id: 123456789
```

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `days` | integer | `7` | `90` | Number of past days to retrieve |

**Response:**
```json
{
  "success": true,
  "waba_id": "123456789",
  "period_days": 7,
  "meta_analytics": {
    "sent": 1200,
    "delivered": 1180,
    "read": 950,
    "data_points": [
      { "start": 1735689600, "end": 1735775999, "sent": 40, "delivered": 39, "read": 31 }
    ]
  },
  "daily_logs": [
    { "date": "2026-05-14", "sent": 40, "failed": 1 }
  ]
}
```

---

### Broadcasts

Send a single template message to a large list of recipients.

#### Create Broadcast
```http
POST /broadcasts
x-partner-key: wf_xxx
x-waba-id: 123456789
Content-Type: application/json
```
```json
{
  "name": "Raya Promo 2026",
  "template_name": "raya_promo",
  "template_language": "ms",
  "template_components": [
    {
      "type": "body",
      "parameters": [{ "type": "text", "text": "Ahmad" }]
    }
  ],
  "contacts": ["60123456789", "60198765432"],
  "scheduled_at": null
}
```

> `scheduled_at` — ISO 8601 datetime string to schedule, or `null` to start immediately.  
> Maximum **10,000 contacts** per broadcast.

**Response:**
```json
{
  "success": true,
  "broadcast_id": 42,
  "status": "pending",
  "total": 2
}
```

---

#### List Broadcasts
```http
GET /broadcasts?limit=20&offset=0
x-partner-key: wf_xxx
x-waba-id: 123456789
```

| Parameter | Default | Max |
|-----------|---------|-----|
| `limit` | `20` | `100` |
| `offset` | `0` | — |

**Response:**
```json
{
  "success": true,
  "broadcasts": [
    {
      "id": 42,
      "waba_id": "123456789",
      "name": "Raya Promo 2026",
      "template_name": "raya_promo",
      "template_language": "ms",
      "total": 500,
      "sent": 498,
      "delivered": 480,
      "failed": 2,
      "status": "completed",
      "scheduled_at": null,
      "started_at": "2026-05-14T10:00:00.000Z",
      "completed_at": "2026-05-14T10:05:00.000Z",
      "created_at": "2026-05-14T09:58:00.000Z"
    }
  ]
}
```

**Broadcast status values:**

| Status | Meaning |
|--------|---------|
| `pending` | Queued, not started yet |
| `scheduled` | Waiting for `scheduled_at` datetime |
| `processing` | Currently sending |
| `completed` | All messages attempted |
| `cancelled` | Cancelled before completion |

---

#### Get Broadcast Details
```http
GET /broadcasts/:broadcastId
x-partner-key: wf_xxx
x-waba-id: 123456789
```
**Response:**
```json
{
  "success": true,
  "broadcast": {
    "id": 42,
    "name": "Raya Promo 2026",
    "total": 500,
    "sent": 498,
    "delivered": 480,
    "failed": 2,
    "status": "completed",
    "error_log": null,
    "started_at": "2026-05-14T10:00:00.000Z",
    "completed_at": "2026-05-14T10:05:00.000Z"
  },
  "progress": 100
}
```

---

#### Cancel Broadcast
```http
POST /broadcasts/:broadcastId/cancel
x-partner-key: wf_xxx
x-waba-id: 123456789
```
**Response:**
```json
{ "success": true }
```

> Can only cancel broadcasts with status `pending` or `scheduled`. Already-started broadcasts cannot be cancelled.

---

## Receiving Webhooks

### Overview

| | Base URL | Webhook URL |
|--|--|--|
| Whose server | WasapFlow | **Your server** |
| Direction | Your system → WasapFlow | WasapFlow → Your system |
| Purpose | Send messages, manage WABAs | Receive incoming messages & delivery status |

**Base URL** is fixed: `https://officialapi.wasapflow.com/bridge/v1`

**Webhook URL** is your own endpoint — build it, then set it in **Settings → Webhook URL**.

> If you do not set a Webhook URL, you can still send messages but you will **not receive** incoming messages or delivery receipts.

---

### Setup

1. Build an endpoint on your server:
```javascript
app.post('/webhook/wasapflow', express.json(), (req, res) => {
    res.status(200).send('OK'); // always respond immediately
    const { event, waba_id, data } = req.body;
    // handle events below...
});
```

2. Go to **Settings → Webhook URL** → enter your full public URL → Save.

3. Your **Webhook Secret** is in **Settings → Webhook Security** — use it to verify requests.

---

### Verify Incoming Webhooks

Every webhook request includes an `x-wasapflow-signature` header. Always verify before processing:

```javascript
const crypto = require('crypto');

app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
    const sig      = req.headers['x-wasapflow-signature'];
    const expected = 'sha256=' + crypto
        .createHmac('sha256', process.env.WF_WEBHOOK_SECRET)
        .update(req.body)
        .digest('hex');

    if (sig !== expected) return res.status(401).send('Unauthorized');

    res.status(200).send('OK'); // respond immediately — then process

    const { event, waba_id, phone_number_id, timestamp, data } = JSON.parse(req.body);
    // handle event...
});
```

> Always respond `200 OK` first, then process asynchronously. Your endpoint must respond within **10 seconds** or the attempt is counted as failed and will be retried.

---

### Webhook Envelope

Every webhook POST has this top-level structure:

```json
{
  "event": "message.received",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": { ... },
  "meta": {
    "api_version": "2.2.0",
    "changelog": "https://partner.wasapflow.com/bridge/docs?tab=changelog",
    "notices": [
      {
        "id": "meta-service-message-billing-2026-10-01",
        "severity": "action_required",
        "effective": "2026-10-01"
      }
    ]
  }
}
```

The `meta` object (added in 2.2.0) tells you the current API version and any Meta platform change coming your way. `notices` is an empty array when there is nothing pending. Notice IDs are stable forever, so you can suppress ones you have already handled.

> The `meta` object is inside the signed body. If you verify signatures, no change is needed — the signature is computed over the complete body as sent.

Additional headers sent with every webhook:

| Header | Value |
|--------|-------|
| `x-wasapflow-signature` | `sha256=<hmac>` |
| `x-wasapflow-event` | Event name, e.g. `message.received` |
| `x-wasapflow-waba-id` | WABA ID |

---

### Webhook Event Payloads

#### `message.received` — Inbound message from end user

```json
{
  "event": "message.received",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "from": "60123456789",
    "bsuid": "MY.2035200694071263",
    "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgASGBQzQUVEMUFBQkVBRUQ5NTJERTA",
    "type": "text",
    "text": "Hello, I need help!",
    "contact_name": "Ahmad",
    "timestamp": 1715695200,
    "raw": { ... }
  }
}
```

> **`from`** — Phone number in international format without `+`, e.g. `60123456789`. Passed through from Meta `message.from`.
> **`bsuid`** — 🆔 Business-Scoped User ID (added by Meta April 2026). A **separate field** from `from` — always sent alongside, not as a replacement. Format observed: `CC.<numeric_id>` (e.g. `MY.2035200694071263`, `SG.1234567890123`). Unique per business-user pair — same user gets a different BSUID across different businesses. May be `null` for legacy webhooks or users not yet on a BSUID-aware client. Bridge does **not** validate the format — passed through verbatim from Meta `contacts[0].user_id`.
> **`type`** — Message type: `text`, `image`, `document`, `audio`, `video`, `location`, `button`, `interactive`, `sticker`, `reaction`, `unknown`
> **`text`** — Populated for text messages only. For media messages, use `data.raw.message` to get the full media object including `id` for downloading.

**🆔 BSUID & WhatsApp Usernames (June 2026+) — detection guidance:**

`data.from` and `data.bsuid` are **two separate fields** in the same payload. Bridge does **not** replace `from` with a BSUID — you receive both. Detect BSUID presence with a null-check, not by regex-matching `from`:

```js
// ✅ Correct — presence check on the dedicated field
const hasBsuid = payload.data.bsuid != null;

// ❌ Wrong — don't regex-match data.from; it's always the phone
if (/^[A-Z]{2}\.\d+$/.test(payload.data.from)) { ... }
```

**Recommendation:** Persist BOTH `from` (phone) AND `bsuid` against each contact. Use `bsuid` as your stable customer key — phone may change/disappear when the user adopts a WhatsApp username; BSUID won't. The same BSUID also appears as `recipient_bsuid` on `message.sent/delivered/read/failed/echo`, and as `contact_bsuid` on `message.history`.

##### ⚠️ Required: never let a BSUID become a phone number (added 2.3.0)

Meta states that supporting BSUID is **required** for all partners. The part that catches most integrations is not parsing it — it is what happens downstream.

Once a customer adopts a username, Meta may omit the phone number from the webhook entirely: `wa_id`, `from` and `recipient_id` are *"omitted entirely"* when the user has a username and there has been no interaction in the last 30 days. You get a BSUID and nothing else.

A BSUID sends WhatsApp messages fine. **A courier cannot call it.**

**Check your phone normaliser today.** Nearly every integration has one shaped like this:

```javascript
let n = phone.replace(/\D/g, '');
if (n.startsWith('0')) n = '6' + n;
if (!n.startsWith('60')) n = '60' + n;
```

Give it `MY.2035200694071263` and it returns **`602035200694071263`** — a value that looks like a Malaysian number, passes naive validation, and reaches your courier. No exception, no log line. If your normaliser strips non-digits before validating, you have this bug right now.

Guard it explicitly, and return null rather than trying to clean the value:

```javascript
// 2-letter country code, dot, alphanumerics.
// Parent BSUIDs insert ENT: US.ENT.11815799212886844830
const BSUID = /^(?:whatsapp:)?[A-Z]{2}\.(?:ENT\.)?[A-Za-z0-9]{1,128}$/;

function toPhone(value) {
    if (!value) return null;
    if (BSUID.test(String(value).trim())) return null;  // never normalise a BSUID
    if (/[A-Za-z]/.test(value)) return null;            // letters mean it is not a phone
    // ...your existing normalisation...
}
```

**Split the field.** Most schemas keep one `phone` on the contact and use it for everything. It now has to be two:

| Role | Contents | Used for |
|---|---|---|
| Canonical id | Phone **or** BSUID | Sending WhatsApp, matching a contact back |
| Reachable phone | Always a real phone, may be empty | Courier, invoices, voice calls |

**Ask for the phone in your order flow.** If your flow collects name, address and postcode but never a phone — because the WhatsApp number was always assumed to be the phone — it now produces orders that cannot be delivered. When you already hold a plausible number, ask the customer to *confirm* it rather than retype it; that keeps the friction to a one-word answer. This is worth doing before usernames even arrive, since customers regularly order from a work WhatsApp or on behalf of someone else.

**Pushing to WooCommerce or an ERP:** send the real phone as `billing.phone`, and keep the canonical id in a separate meta field for matching webhooks back. Leave `billing.phone` **empty** rather than filling it with an id — an empty field makes a human ask; a fake number sends a courier to a dead end.

Full guidance, including what Meta requires that Bridge does not yet expose: [Changelog 2.3.0](?tab=changelog).

**Handling different message types:**
```javascript
const { event, data } = JSON.parse(req.body);

if (event === 'message.received') {
    switch (data.type) {
        case 'text':
            console.log(`Text from ${data.from}: ${data.text}`);
            break;
        case 'image':
        case 'document':
        case 'audio':
        case 'video':
            const mediaId = data.raw.message[data.type]?.id;
            // Download: GET /contacts/media/:mediaId
            console.log(`Media (${data.type}) ID: ${mediaId}`);
            break;
        case 'button':
            // Quick reply button tapped
            const buttonId = data.raw.message.button?.payload;
            console.log(`Button tapped: ${buttonId}`);
            break;
        case 'interactive':
            // List reply selected
            const listId = data.raw.message.interactive?.list_reply?.id;
            console.log(`List option selected: ${listId}`);
            break;
    }
}
```

---

#### `message.sent` — Message accepted by Meta

```json
{
  "event": "message.sent",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgASGBQzQUVEMUFBQkVBRUQ5NTJERTA",
    "status": "sent",
    "recipient": "60123456789",
    "recipient_bsuid": "MY.2035200694071263",
    "timestamp": 1715695200,
    "errors": null
  }
}
```

---

#### `message.delivered` — Message delivered to phone

```json
{
  "event": "message.delivered",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695260,
  "data": {
    "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgASGBQzQUVEMUFBQkVBRUQ5NTJERTA",
    "status": "delivered",
    "recipient": "60123456789",
    "recipient_bsuid": "MY.2035200694071263",
    "timestamp": 1715695260,
    "errors": null,
    "pricing": {
      "billable": false,
      "pricing_model": "PMP",
      "type": "free_customer_service",
      "category": "service"
    },
    "conversation": {
      "id": "b0d4a1f2c3e4",
      "origin": "service"
    }
  }
}
```

##### `pricing` and `conversation` (added in 2.2.0)

Meta charges per **delivered** message under per-message pricing (PMP). These two objects pass through exactly what Meta reports, so you can attribute cost per message without a separate analytics call.

| Field | Values | Meaning |
|---|---|---|
| `pricing.billable` | `true` / `false` | Whether Meta charges for this message |
| `pricing.pricing_model` | `PMP` | Per-message pricing |
| `pricing.type` | `regular`, `free_customer_service`, `free_entry_point` | See below |
| `pricing.category` | `service`, `utility`, `marketing`, `authentication`, `referral_conversion` | Which rate applies |
| `conversation.id` | string | Meta's conversation ID |
| `conversation.origin` | same values as `category` | What opened the conversation |

Both are `null` when Meta does not supply them — commonly on `read` receipts. **Treat them as optional** and never assume they are present.

**Why `pricing.type` matters right now:**

- `regular` — billable today.
- `free_customer_service` — free today, **becomes billable on 1 October 2026**. This covers both non-template service replies and utility templates sent inside an open customer service window.
- `free_entry_point` — inside the 72-hour free entry point window. **Stays free.**

To project your 1 October exposure, count messages where `pricing.type === "free_customer_service"` over a full month. That count multiplied by the published rate is your added cost. Meta publishes the rates by 1 September 2026.

---

#### `message.read` — Message opened by recipient

```json
{
  "event": "message.read",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695320,
  "data": {
    "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgASGBQzQUVEMUFBQkVBRUQ5NTJERTA",
    "status": "read",
    "recipient": "60123456789",
    "recipient_bsuid": "MY.2035200694071263",
    "timestamp": 1715695320,
    "errors": null
  }
}
```

---

#### `message.failed` — Delivery failed

```json
{
  "event": "message.failed",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgASGBQzQUVEMUFBQkVBRUQ5NTJERTA",
    "status": "failed",
    "recipient": "60123456789",
    "recipient_bsuid": "MY.2035200694071263",
    "timestamp": 1715695200,
    "errors": [
      {
        "code": 131047,
        "title": "Re-engagement message",
        "message": "More than 24 hours have passed since the customer last replied"
      }
    ]
  }
}
```

> **Common Meta error codes:**  
> `131047` — 24-hour window expired (use a template message instead)  
> `131026` — Recipient is not a WhatsApp user  
> `132001` — Template not found or not approved  
> `131000` — Generic message send failure

---

#### `message.echo` — Outbound message sent from the WhatsApp Business App (Coexistence) 🆕

> **Coexistence only.** Fired when your client (or their staff) types a reply **manually from the WhatsApp Business App** on their phone — not through the API. Lets your inbox stay in sync with messages sent outside your platform. Only delivered for WABAs registered with `connection_mode: "coexistence"`.

```json
{
  "event": "message.echo",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "direction": "outbound",
    "source": "business_app",
    "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgARGBI5QUVE...",
    "from": "60111111111",
    "recipient": "60123456789",
    "recipient_bsuid": null,
    "type": "text",
    "text": "Okay boss, sudah siap ✅",
    "timestamp": 1715695200,
    "raw": { ... }
  }
}
```

> **`direction`** — Always `"outbound"` (business → customer).
> **`source`** — Always `"business_app"`. Use this to distinguish manual Business-App replies from API-sent messages (which arrive as `message.sent`).
> **`from`** — The business's own number. **`recipient`** — The customer.
> **`text`** — Populated for text echoes. For media, read `data.raw.echo` for the media object + `id`.
>
> **Tip:** Store these as outbound messages in your inbox so the conversation mirrors WhatsApp exactly, regardless of whether the reply came from your platform or the Business App.

---

#### `message.history` — Historical message synced after onboarding (Coexistence) 🆕

> **Coexistence only, one-time.** When a client onboards via the WhatsApp Business App, Meta replays their **existing conversation history** so you can backfill past chats. Delivered as a stream of individual messages (one webhook per message) shortly after registration. Not replayed for WABAs already onboarded before this feature.

```json
{
  "event": "message.history",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "history": true,
    "direction": "outbound",
    "status": "read",
    "thread_id": "60123456789",
    "contact_wa_id": "60123456789",
    "contact_bsuid": "MY.2035200694071263",
    "contact_username": "@alice",
    "message_id": "wamid.HBgNNjAxMjM0NTY3ODkVAgARGBI...",
    "from": "60111111111",
    "type": "text",
    "text": "Terima kasih, order saya dah sampai",
    "timestamp": 1504902988,
    "phase": 1,
    "progress": 30,
    "raw": { ... }
  }
}
```

> **`history`** — Always `true`. Use this flag to route historical messages into a backfill path (don't trigger auto-replies/notifications on them).
> **`direction`** — `"outbound"` if the business sent it (`from_me`), else `"inbound"`.
> **`status`** — Original delivery status from history (`sent`, `delivered`, `read`).
> **`phase` / `progress`** — Meta's sync progress (`progress` 0→100). Lets you show a "syncing history…" indicator.
> **`timestamp`** — **Original** message time (not sync time) — use it to order backfilled messages correctly.
>
> **Note:** Media in history may use `type: "media_placeholder"`, and very old media may no longer be downloadable from Meta.

---

#### `waba.quality_updated` — Phone number quality rating changed

```json
{
  "event": "waba.quality_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "phone_number_id": "987654321",
    "quality_rating": "YELLOW",
    "previous_rating": "GREEN",
    "raw": { ... }
  }
}
```

> **Ratings:** `GREEN` (good), `YELLOW` (medium — watch your opt-out rate), `RED` (high opt-out — messaging may be restricted)

---

#### `waba.tier_updated` — Messaging limit tier changed

```json
{
  "event": "waba.tier_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "tier": "TIER_10K",
    "raw": { ... }
  }
}
```

> **Tier values:** `TIER_250`, `TIER_1K`, `TIER_2K`, `TIER_10K`, `TIER_100K`, `UNLIMITED`

---

#### `waba.alert` — Meta sent a business alert

```json
{
  "event": "waba.alert",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "alert": {
      "type": "BUSINESS_ACCOUNT_RESTRICTION",
      "details": "Your account has been flagged due to policy violation."
    },
    "raw": { ... }
  }
}
```

> Alert types and content come directly from Meta. Check Meta's Business Manager for full details. Common alerts include policy violations, account restrictions, and phone number quality warnings.

---

#### `template.status_updated` — Template approved / rejected by Meta 🆕

> Fired when a template you created (`POST /templates`) changes review status. **Essential** if you manage templates programmatically — don't send a template until its status is `APPROVED`.

```json
{
  "event": "template.status_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "template_id": 12345678,
    "template_name": "order_confirmation",
    "language": "ms",
    "status": "APPROVED",
    "category": "MARKETING",
    "reason": null,
    "raw": { ... }
  }
}
```

> **`status`** — `APPROVED`, `REJECTED`, `PENDING`, `PAUSED`, `DISABLED`. **`reason`** — populated on rejection (e.g. policy violation).

---

#### `template.quality_updated` — Template quality score changed 🆕

```json
{
  "event": "template.quality_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "template_id": 12345678,
    "template_name": "order_confirmation",
    "language": "ms",
    "previous_quality": "GREEN",
    "new_quality": "YELLOW",
    "raw": { ... }
  }
}
```

> **Quality:** `GREEN` (good), `YELLOW` (watch), `RED` (high block rate — template may be paused). Act before a template gets disabled.

---

#### `template.category_updated` — Meta re-categorised a template 🆕

```json
{
  "event": "template.category_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "template_id": 12345678,
    "template_name": "order_confirmation",
    "language": "en_US",
    "previous_category": "MARKETING",
    "new_category": "UTILITY",
    "correct_category": "MARKETING",
    "appeal_status": "ELIGIBLE",
    "raw": { ... }
  }
}
```

> Meta may re-categorise templates (affects pricing). `appeal_status` tells you whether you can appeal the change.

---

#### `waba.account_updated` — WABA account status changed 🆕

```json
{
  "event": "waba.account_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "phone_number": "60123456789",
    "event": "VERIFIED_ACCOUNT",
    "raw": { ... }
  }
}
```

> **`event`** examples: `VERIFIED_ACCOUNT`, `DISABLED_UPDATE`, `ACCOUNT_RESTRICTION`. Watch this to detect when a client's WABA is verified, disabled, or restricted.

---

#### `waba.review_updated` — Business verification review decision 🆕

```json
{
  "event": "waba.review_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "decision": "APPROVED",
    "raw": { ... }
  }
}
```

> **`decision`** — `APPROVED` or `REJECTED`. Result of Meta's business verification review.

---

#### `contact.synced` — Contact added/edited in the WhatsApp Business App (Coexistence) 🆕

> **Coexistence only.** Fired when your client adds, edits, or removes a contact directly in the WhatsApp Business App. Keeps your contact list in sync with their phone. One event per contact.

```json
{
  "event": "contact.synced",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "action": "add",
    "type": "contact",
    "full_name": "Ahmad Razak",
    "first_name": "Ahmad",
    "phone_number": "60123456789",
    "version": 1,
    "timestamp": 1715695200,
    "raw": { ... }
  }
}
```

> **`action`** — `add`, `update`, or `remove`. Upsert/delete the contact in your CRM by `phone_number`.

---

#### `waba.pricing_tier_updated` — Volume pricing tier reached 🆕

```json
{
  "event": "waba.pricing_tier_updated",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "pricing_category": "UTILITY",
    "tier": "25000001:50000000",
    "region": "Malaysia",
    "effective_month": "2026-09",
    "tier_update_time": 1743451903,
    "raw": { ... }
  }
}
```

> **Not the same as `waba.tier_updated`.** That one is the *messaging* tier — how many unique recipients per 24 hours. This one is about **price**: reaching a new volume tier unlocks a lower rate for utility and authentication messages in that market.
>
> **`tier`** — `"<lower>:<upper>"`. Subtract your current volume from `<upper>` to see how many more messages reach the next tier. `<upper>` may be the string `MAX`.
> **`pricing_category`** — `UTILITY` or `AUTHENTICATION` only. **Service messages have no volume tiers**, so this never fires for them.
>
> Meta may send more than one webhook for the same tier change — use the one with the smallest `tier_update_time`.

---

#### `waba.offboarded` / `waba.reconnected` — Coexistence re-registration 🆕

Fired when a Coexistence client re-registers their WhatsApp Business App (new device, reinstall). Meta offboards the number, then reonboards it automatically in the background, usually within a few minutes.

```json
{
  "event": "waba.offboarded",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "waba_id": "123456789",
    "phone_number_id": "987654321",
    "reason": null,
    "raw": { ... }
  }
}
```

> Treat `waba.offboarded` as a **soft pause**, not a disconnection. Sending may fail until `waba.reconnected` arrives. No action is required from you — do not delete the client record.

---

#### `standby.*` — Meta Business Agent is handling the conversation 🆕

When a business enables **Meta Business Agent** on a phone number, Meta's AI agent and your app coexist on that number. Only one is the **active handler** at a time. While your app is the passive listener, Meta **stops sending `message.received`** and sends standby events instead.

| Event | Fired when |
|-------|-----------|
| `standby.message_received` | A user sent a message while your app is not the active handler |
| `standby.message_echo` | Meta Business Agent sent a message on the client's behalf |
| `standby.message_status` | Delivery/read status for a Business Agent message (carries `pricing`) |

```json
{
  "event": "standby.message_received",
  "waba_id": "123456789",
  "phone_number_id": "987654321",
  "timestamp": 1715695200,
  "data": {
    "standby": true,
    "active_handler": "meta_business_agent",
    "message_id": "wamid....",
    "from": "60123456789",
    "contact_name": "Aiman",
    "contact_wa_id": "60123456789",
    "type": "text",
    "text": "Ada saiz S tak?",
    "timestamp": 1715695200,
    "raw": { ... }
  }
}
```

> ⚠️ **Sending a service message while in standby makes your app the active handler.** Per Meta's documentation this is the intended way to escalate to a human agent — but it also means an automated reply will silently take control away from the agent. **If you run a bot, gate it on `standby !== true`:**
>
> ```javascript
> if (data.standby) return saveContextOnly(data);  // observe, don't reply
> ```

`standby.message_echo` carries `template` and `flow` — the full unhydrated definitions of what the agent sent, when applicable. `standby.message_status` carries the same `pricing` and `conversation` objects as regular status events, so Business Agent traffic appears in your cost reporting alongside your own.

**Prerequisite:** the business must enable Meta Business Agent and grant standby permission. Until then these events never fire and nothing changes for you.

##### Required: show your client a visible warning

This is the single most confusing failure mode on the platform, and it is not a failure at all.

When Meta Business Agent takes over, your client sees their automation stop replying. Nothing is broken, nothing errors, and **nothing appears in your logs** — because the messages never reach you. Your client will open a support ticket saying "the bot is dead", and without a banner you will have no way to explain it.

**You must surface this in your UI.** Build a persistent warning banner — in the app header and in the inbox for the affected number — that appears when a `standby.*` event arrives and disappears when normal `message.received` events resume.

Use a **warning** style (yellow/amber), not an error style (red). Nothing has failed; your client simply needs to know who is answering and what their options are.

Recommended wording:

> 🤖 **Meta Business Agent is answering your customers — not your AI.** To take back control, turn it off in WhatsApp Manager → Account tools → Business Agent. Manual chat still works normally.

Implementation shape:

```javascript
// On any standby.* event — raise the banner (idempotent, once per number)
if (event.startsWith('standby.')) {
    await alerts.raise({
        clientId,
        phoneNumberId: payload.phone_number_id,
        type: 'meta_agent_active',
        severity: 'warning'
    });
    return saveContextOnly(data);   // observe — never auto-reply
}

// On a normal inbound message — we are the active handler again, clear it
if (event === 'message.received') {
    await alerts.clear({ clientId, type: 'meta_agent_active' });
}
```

Standby events can arrive many times a minute, so make the raise idempotent — create the alert once and update it, rather than emitting a notification per event.

WasapFlow's own product implements exactly this pattern; we are asking you to mirror it so your clients get the same clarity.

---

### Failed Webhook Events (Recovery)

If your endpoint was down or returned errors, failed events are stored and can be retrieved or replayed.

#### List Failed Events
```http
GET /webhooks/failed?limit=50
x-partner-key: wf_xxx
```
**Response:**
```json
{
  "success": true,
  "count": 2,
  "events": [
    {
      "id": 1,
      "waba_id": "123456789",
      "event": "message.received",
      "payload": { "event": "message.received", "data": { ... } },
      "attempts": 3,
      "last_error": "ECONNREFUSED",
      "status": "failed",
      "created_at": "2026-05-15T10:00:00.000Z"
    }
  ]
}
```

#### Retry a Single Failed Event
```http
POST /webhooks/retry/:eventId
x-partner-key: wf_xxx
```
Re-sends the stored event to your current webhook URL. On success, status changes to `delivered`.

#### Retry All Failed Events
```http
POST /webhooks/retry-all
x-partner-key: wf_xxx
```
Replays up to 100 failed events in chronological order.

**Response:**
```json
{ "success": true, "replayed": 8, "failed": 1, "total": 9 }
```

> Failed events are stored for **30 days**. After that they are purged automatically.

---

### Webhook Retry Policy

| Attempt | Delay after failure |
|---------|-------------------|
| 1st retry | 2 seconds |
| 2nd retry | 4 seconds |
| 3rd retry | 8 seconds |

After 3 failed attempts, the event is dropped and logged. Ensure your endpoint responds `200 OK` within 10 seconds.

---

## Access Token Management

The `access_token` (`EAAxxxxxxxx`) stored per WABA is a **permanent system user token** from Meta. It does not expire on its own, but can be invalidated if:

- The system user is removed from the Business Manager
- The Meta App permissions are changed
- Someone manually revokes the token

**Signs your token is invalid:**
- You receive `500 META_ERROR` responses when sending messages
- Quality/tier refresh calls fail

**How to fix:**
1. Generate a new permanent token from Meta Business Manager → System Users
2. Call `POST /clients/:wabaId/refresh` with `{ "access_token": "new_token" }` — or re-call `POST /clients/register` (safe upsert)

There is no automatic token expiry webhook from Meta. Build a periodic health check in your system (e.g. call `/refresh` weekly) to catch invalid tokens early.

---

## Error Codes

| HTTP | Code | Meaning | Action |
|------|------|---------|--------|
| `400` | `INVALID_REQUEST` | Missing or invalid field | Check request body |
| `400` | `MISSING_FIELDS` | Required fields not provided | Check required params |
| `400` | `TEMPLATE_NOT_FOUND` | Template name/language wrong | Verify in Meta Business Manager |
| `400` | `PHONE_NOT_ON_WHATSAPP` | Recipient not on WhatsApp | Check with `/contacts/:phone` first |
| `400` | `CANNOT_CANCEL` | Broadcast already started | Cannot cancel in-progress broadcast |
| `400` | `TOO_MANY_CONTACTS` | Over 10,000 contacts | Split into multiple broadcasts |
| `401` | `INVALID_KEY` | Partner key invalid | Check `x-partner-key` header |
| `402` | `PAYMENT_REQUIRED` | Subscription unpaid | Pay invoice in Billing page |
| `403` | `SUBSCRIPTION_INACTIVE` | Subscription cancelled | Resubscribe in Billing page |
| `404` | `WABA_NOT_FOUND` | WABA not registered | Register WABA first |
| `404` | `WABA_NOT_REGISTERED` | WABA not found for this partner | Check `x-waba-id` header |
| `429` | `RATE_LIMITED` | Too many requests | Slow down — check `Retry-After` header |
| `500` | `META_ERROR` | Meta rejected request | Check `meta_error_code` in response |
| `500` | `SERVER_ERROR` | Internal error | Retry — contact support if persists |

---

## SDK Reference

> SDKs are installed directly from GitHub (not yet published to npm/PyPI/Packagist):
> ```bash
> npm install github:kobaranteguh/wasapflow-bridge-node
> pip install git+https://github.com/kobaranteguh/wasapflow-bridge-python.git
> composer require kobaranteguh/wasapflow-bridge-php
> ```

### Node.js

```javascript
const { WasapFlowBridge } = require('wasapflow-bridge');

const bridge = new WasapFlowBridge({ partnerKey, webhookSecret });

// Embedded Signup (recommended)
const config = await bridge.clients.getEmbeddedSignupConfig(); // { app_id, config_id }
await bridge.clients.registerFromCode({ code, displayName });  // exchange code securely

// Client management (manual)
await bridge.clients.register({ wabaId, phoneNumberId, accessToken, displayName });
await bridge.clients.list();
await bridge.clients.remove(wabaId);
await bridge.clients.refresh(wabaId, { accessToken }); // accessToken optional — updates token + syncs quality
await bridge.clients.resubscribeWebhook(wabaId);       // reconnect Meta webhook

// Scoped to a WABA
const waba = bridge.client(wabaId);
await waba.messages.send({ to, text });
await waba.messages.template({ to, template, language, params });
await waba.messages.media({ to, type, url, caption });
await waba.messages.interactive({ to, body, buttons });
await waba.messages.location({ to, latitude, longitude, name, address });
await waba.messages.reaction({ to, messageId, emoji });
await waba.messages.markRead({ messageId });
await waba.templates.list();
await waba.templates.create({ name, language, category, components });
await waba.templates.delete(name);
await waba.profile.get();
await waba.profile.update({ about, address, email, websites });
await waba.analytics.get({ days: 7 });
await waba.broadcasts.create({ name, templateName, templateLanguage, templateComponents, contacts });
await waba.broadcasts.list();
await waba.broadcasts.get(broadcastId);
await waba.broadcasts.cancel(broadcastId);
await waba.contacts.check(phone);
await waba.contacts.downloadMedia(mediaId);

// Webhook verification
bridge.webhooks.verify(headers, body); // returns parsed event object or null
```

---

### Python

```python
from wasapflow_bridge import WasapFlowBridge

bridge = WasapFlowBridge(partner_key=os.environ['WF_PARTNER_KEY'])

config = bridge.clients.get_embedded_signup_config()  # { app_id, config_id }
bridge.clients.register_from_code(code, display_name='My Client')

bridge.clients.register(waba_id, phone_number_id, access_token, display_name)  # manual
bridge.clients.refresh(waba_id, access_token=None)
bridge.clients.resubscribe_webhook(waba_id)

waba = bridge.client(waba_id)
await waba.messages.send(to='60123456789', text='Hello!')
await waba.messages.template(to='60123456789', template='order_confirmed', language='en_US', params=['John'])
await waba.analytics.get(days=7)
```

---

### PHP

```php
use WasapFlow\Bridge\WasapFlowBridge;

$bridge = new WasapFlowBridge(['partnerKey' => getenv('WF_PARTNER_KEY')]);

$config = $bridge->clients->getEmbeddedSignupConfig(); // { app_id, config_id }
$bridge->clients->registerFromCode($code, $displayName);

$bridge->clients->register($wabaId, $phoneNumberId, $accessToken, $displayName); // manual
$bridge->clients->refresh($wabaId, $accessToken = null);
$bridge->clients->resubscribeWebhook($wabaId);

$waba = $bridge->client($wabaId);
$waba->messages->send(['to' => '60123456789', 'text' => 'Hello!']);
$waba->messages->template(['to' => '60123456789', 'template' => 'order_confirmed', 'language' => 'en_US', 'params' => ['John']]);
$waba->analytics->get(['days' => 7]);
```

---

## Health Check

```http
GET /health
```
No authentication required. Use for uptime monitoring.

**Response:**
```json
{ "status": "ok", "service": "wasapflow-bridge", "timestamp": "2026-05-15T00:00:00.000Z" }
```

---

## Rate Limits

| Setting | Default |
|---------|---------|
| Requests/second | 200 |
| Response on limit | `429 Too Many Requests` with `Retry-After` header |

Contact support to increase your limit.

---

## Platform & Security

| Item | Details |
|------|---------|
| Hosting | DigitalOcean (Singapore region) |
| Token Encryption | AES-256-CBC, encrypted at rest with random IV |
| Webhook Signing | HMAC-SHA256 |
| API Version (Meta) | v24.0 (auto-updated by WasapFlow) |
| Failed Events Retention | 30 days |

---

## Meta API Coverage

WasapFlow Bridge proxies the most commonly used Meta WhatsApp Cloud API endpoints.

**Coexistence sync (Supported since 2.0.0):** For WABAs onboarded via the WhatsApp Business App (`connection_mode: "coexistence"`), Bridge forwards messages your client sends manually from the Business App (`message.echo`) and replays their past conversation history once after onboarding (`message.history`). See [Webhook Event Payloads](#webhook-event-payloads).

The following Meta features are **not currently supported** through Bridge and are on the roadmap:

| Feature | Status |
|---------|--------|
| Phone Number Management (verify, list) | Roadmap |
| Template Editing (update existing) | Roadmap |
| Media Deletion | Roadmap |
| WhatsApp Flows | Roadmap |
| WhatsApp Business Calling | Roadmap |
| QR Code Generation | Roadmap |
| Commerce / Product Catalog | Roadmap |
| Two-Step Verification (2FA PIN) | Handled during Embedded Signup |

> Meta API evolves regularly. WasapFlow tracks the latest changes and updates Bridge endpoints accordingly. If you need a specific Meta endpoint that isn't available, contact support — we may be able to add it.

---

*WasapFlow Bridge API Reference v2.6.0 — for registered partners only.*
