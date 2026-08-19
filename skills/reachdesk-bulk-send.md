---
name: reachdesk-bulk-send
description: >-
  Send one Reachdesk campaign to many recipients in a single asynchronous request,
  then reconcile the resulting individual sends. Use for batch gifting — event
  follow-up, ABM waves, employee milestones — up to 5000 recipients per call.
api: reachdesk:reachdesk-api
base_url: https://app.reachdesk.com/api/v2
operations:
  - bulk-create
  - list-sends
  - get-send
generated: '2026-08-13'
method: generated
source: openapi/reachdesk-api-openapi.yml + conventions/reachdesk-conventions.yml
---

# Bulk gift send with Reachdesk

`bulk-create` is the batch counterpart to `trigger-campaign`. It accepts a list of
recipients, returns **immediately** with a bulk send id, and creates the individual
gifts in the background.

## Before you start

- `Authorization: Bearer {api_token}` on every request.
- Know the `campaign_id`. There is no campaign lookup operation in the API.
- The campaign must be a **manual** campaign.
- As with the single trigger, everything about the gift — gift-card amount, bundle
  items, note template, marketplace product — comes from the campaign. You supply
  only recipients, sender, payment routing and custom attributes. It works for every
  campaign type: gift card, note, bundle and marketplace.

## This spends money at scale

One call can dispatch up to 5000 gifts. Treat it as requiring explicit human
authorisation of the batch — the recipient count, the campaign, and the funding
wallet — before it is issued.

## Steps

### 1. Call `bulk-create`

`POST /bulk_sends`

Required fields: `request_id`, `campaign_id`, `sender`, `recipients`.

```
POST https://app.reachdesk.com/api/v2/bulk_sends
Authorization: Bearer {api_token}
Content-Type: application/json

{
  "request_id": "<a unique id you generate for this request>",
  "campaign_id": 4321,
  "sender": "jane.doe@example.com",
  "recipients": [
    {
      "first_name": "John",
      "last_name": "Doe",
      "email": "john.doe@example.com",
      "company_name": "Example Ltd.",
      "country": "US"
    }
  ]
}
```

- `sender` must be a user in your organization **with access to the campaign**.
- Maximum **5000 recipients** per request.
- Recipient fields match the single-trigger recipient object: name, email,
  `company_name`, address lines, `city`, `state`, `zipcode`, `country` (ISO 3166
  alpha-2), `phone_number`, `role`, `tax_id`, and the optional CRM linkage fields
  `provider` / `provider_contact_id` / `provider_contact_type`.
- Address fields are only used for physical sends.

### 2. Handle the 202

```json
{ "id": 98123 }
```

This is an **acknowledgement, not a result**. The gifts do not exist yet.

### 3. Reconcile — this is the hard part

There is no `GET /bulk_sends/{id}`. The id from the 202 cannot be looked up, and
Reachdesk publishes no outbound webhooks or completion events. To find out what
actually happened:

1. Poll `GET /sends` (`list-sends`) with `start_date` / `end_date` (format
   `YYYY-MM-DD`), or `start_updated_date` / `end_updated_date` for changes. Those two
   updated-date parameters are mutually dependent — send both or neither.
2. Page with `page` and `per_page` (defaults 1 and 25). No total or next-page field
   is documented, so keep paging until a page comes back short.
3. Match rows to your batch by recipient email and campaign, and read
   `payment_wallet_id` to confirm the correct budget was debited.
4. `GET /sends/{id}` (`get-send`) for detail on any individual send.

Build the reconciliation window from your own submission timestamp. Do not assume
the batch has finished just because the 202 returned.

## Error handling

| Status | Body | Meaning |
|---|---|---|
| 400 | `{"message": "..."}` | Parameter validation failed; the message names the offending parameter. |
| 401 | `text/plain`, empty | Missing or invalid API token. |
| 404 | `text/plain`, empty | Unknown `campaign_id`, or the sender has no access to it. |
| 422 | `text/plain`, empty | Payload valid but semantically rejected — check required fields, the 5000 ceiling, and that the campaign is manual. |

Three of those four responses carry **no body at all**. Only the status code is
informative; log the request and status and escalate to a human.

## Retry safety

`request_id` is required here, but Reachdesk never documents replay semantics for
it — no statement that a duplicate `request_id` is deduplicated, no retention
window, no `Idempotency-Key` header. **Never blind-retry a bulk send.** A repeat
could dispatch up to 5000 duplicate gifts and charge for them. On an ambiguous
failure, reconcile through `list-sends` first and only then decide.

## Rate limits

Not published. No `RateLimit-*` or `Retry-After` headers are documented. Space
large batches out.
