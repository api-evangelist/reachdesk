---
name: reachdesk-send-a-gift
description: >-
  Send one gift to one recipient through a Reachdesk manual campaign, routing the
  charge to the correct wallet and choosing whether the send is dispatched
  immediately or held for human approval. Use when a workflow needs to trigger a
  single corporate gift, e-gift or swag item to a named person.
api: reachdesk:reachdesk-api
base_url: https://app.reachdesk.com/api/v2
operations:
  - trigger-campaign
  - get-send
generated: '2026-08-13'
method: generated
source: openapi/reachdesk-api-openapi.yml + conventions/reachdesk-conventions.yml
---

# Send a gift with Reachdesk

## Before you start

- Authenticate every request with `Authorization: Bearer {api_token}`. The token is
  organization-wide and is created by an Organization Admin at
  `https://app.reachdesk.com/api_tokens`.
- You must already know the **campaign id**. The public API has no
  list-campaigns or get-campaign operation — get the id from the Reachdesk UI or
  from configuration. Do not guess it.
- The campaign must be a **manual** campaign. Automated campaigns are rejected.
- The campaign, not your request, defines the gift: amount, bundle items, note
  template and marketplace product all come from the campaign configuration.

## This operation spends money

`trigger-campaign` debits a funding wallet and dispatches a physical or digital
gift to a named person. It is not reversible through the API. Decide the approval
mode deliberately before calling.

## Steps

### 1. Choose the approval mode

Set `approved` in the body:

| Value | Behaviour |
|---|---|
| `"true"` | Process immediately. An invalid send returns an error. |
| `"false"` | Create the send in a **pending** state for manual review in the Reachdesk UI. |
| `"auto"` | Try to approve automatically; fall back to pending if invalid. |

Unless a human has explicitly authorised autonomous spending, use `"false"` and let
a person approve in the Sends tab.

### 2. Route the payment wallet explicitly

Wallet selection is driven by three fields — `payment_currency`,
`payment_wallet_type` (`User` or `Team`, default `User`) and `team_name`. The
fallback that matters:

> If `payment_wallet_type` is `Team` and the sender is **not** a member of
> `team_name`, Reachdesk silently debits the sender's own **User** wallet in that
> currency. No error is returned.

If which budget is charged matters, set `team_name` and verify `payment_wallet_id`
on the response.

If `payment_currency` is omitted the campaign's currency is used. Allowed
currencies: AUD, CAD, DKK, EUR, GBP, INR, NOK, SEK, USD.

### 3. Call `trigger-campaign`

`POST /campaigns/{id}/trigger`

```
POST https://app.reachdesk.com/api/v2/campaigns/{campaign_id}/trigger
Authorization: Bearer {api_token}
Content-Type: application/json

{
  "request_id": "<a unique id you generate for this request>",
  "source": "<name of the system triggering this send>",
  "sender": "jane.doe@example.com",
  "approved": "false",
  "payment_wallet_type": "Team",
  "team_name": "Account Managers UK",
  "payment_currency": "GBP",
  "send_method": "email",
  "confirm_address": true,
  "recipient": {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john.doe@example.com",
    "company_name": "Example Ltd.",
    "street_address_1": "45 Rockefeller Plaza",
    "city": "New York",
    "state": "NY",
    "zipcode": "10111",
    "country": "US"
  },
  "custom_attributes": [
    { "name": "custom_gift_note", "value": "Thanks for the great call" }
  ]
}
```

- `sender` must be the email of a real Reachdesk platform user with access to the
  campaign.
- `country` is an ISO 3166 two-letter code.
- Address fields are only used for physical sends.
- `send_method`: `email` (default) emails the gift; `link` returns a shareable
  `claim_url` in the response instead of emailing.
- For bundle campaigns, `items[]` sets `sku`/`quantity` within the range the
  campaign allows; omit it to take the campaign minimums.

### 4. Use custom attributes for personalised copy

Anything in `custom_attributes` can be interpolated into the campaign's gift note,
gift email and gift-option text fields as `{{name}}`. Names must use underscores
instead of spaces — `custom_gift_note`, not `custom gift note`.

### 5. Read the response

A `200` returns the created send:

```json
{
  "id": 55,
  "type": "gift_card",
  "status": "processed",
  "payment_currency": "USD",
  "payment_wallet_id": 1234,
  "claim_url": null,
  "recipient": { "first_name": "John", "last_name": "Doe", "email": "john.doe@example.com" },
  "sender":    { "first_name": "Jane", "last_name": "Doe", "email": "jane.doe@example.com" },
  "gift_card": { "amount": 15.0, "currency": "USD" }
}
```

Record `id` — it is the only handle you get. Confirm `payment_wallet_id` is the
wallet you intended. When `send_method` is `link`, `claim_url` carries the
shareable link.

### 6. Follow up

`GET /sends/{id}` (`get-send`) reads a single send back. There are no outbound
webhooks and no event stream, so status changes are only visible by polling.

## Error handling

The contract declares **only a 200** on this operation, so treat any non-2xx as
undocumented and surface it verbatim rather than interpreting it.

| Status | Meaning | Action |
|---|---|---|
| 401 | Missing, malformed or revoked token | Check the `Bearer` header; regenerate the token. |
| 404 | Unknown send/campaign, or not visible to this organization | Body is `{"error": "Send not found"}` on `get-send`. Verify the id. |
| 400 | Parameter validation failure | Some endpoints return `{"message": "..."}`; others document an empty body. |

There is no RFC 9457 problem+json, no error-code registry, and no documented 429
or 5xx.

## Retry safety

**Do not blind-retry.** `request_id` is described only as "a random unique
identifier generated for each request". Reachdesk never states that replaying the
same `request_id` is deduplicated, publishes no retention window, and offers no
`Idempotency-Key` header. A retried trigger may send — and charge for — a second
gift. On a timeout or ambiguous failure, poll `GET /sends` for the window and
reconcile before sending again.

## Rate limits

None are published, and no `RateLimit-*`, `X-RateLimit-*` or `Retry-After` headers
are documented. Throttle conservatively.
