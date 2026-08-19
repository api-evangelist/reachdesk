---
name: reachdesk-reconcile-spend
description: >-
  Read back what a Reachdesk organization has sent and what it cost — enumerate
  sends over a date window, pull individual send detail, and reconcile against the
  transaction ledger filtered by team, user, currency and campaign type. Use for
  gifting spend reporting, budget checks and ROI attribution.
api: reachdesk:reachdesk-api
base_url: https://app.reachdesk.com/api/v2
operations:
  - get-organization
  - list-sends
  - get-send
  - list-transactions
  - list-contacts
generated: '2026-08-13'
method: generated
source: openapi/reachdesk-api-openapi.yml + data-model/reachdesk-data-model.yml
---

# Reconcile Reachdesk sends and spend

All operations here are **read-only**. Nothing in this skill spends money.

## 1. Confirm which organization the token belongs to

`GET /organization` (`get-organization`) returns the basic information about the
organization associated with the API token. Call it first to confirm you are
pointed at the right account.

> The provider declares this response body as an empty object with an example of
> `{}` — the actual field set is undocumented. Read whatever comes back; do not
> assume a shape.

## 2. Enumerate sends over a window

`GET /sends` (`list-sends`)

| Parameter | Notes |
|---|---|
| `start_date` | `YYYY-MM-DD`. Sends created from this date onwards. |
| `end_date` | `YYYY-MM-DD`. Sends created before this date. |
| `start_updated_date` / `end_updated_date` | Filter by update date. **Mutually dependent** — send both or neither. Use these for incremental syncs. |
| `page` | Default 1. |
| `per_page` | Default 25. |

Page until a short page comes back — no total, next or has-more field is
documented.

Each send carries `id`, `type`, `status`, `payment_currency`, `payment_wallet_id`,
`packaging_notes`, `claim_url`, `custom_attributes[]`, `created_by_user`,
`recipient`, `sender`, and the gift detail. Since January 2026 the response also
exposes the chosen gift or charity name and warehouse items with quantities.

## 3. Pull detail on a specific send

`GET /sends/{id}` (`get-send`). A 404 returns `{"error": "Send not found"}` —
which also happens when the send belongs to a different organization.

## 4. Reconcile against the transaction ledger

`GET /transactions` (`list-transactions`) is where the money is.

| Parameter | Notes |
|---|---|
| `start_date` / `end_date` | Creation date bounds. |
| `transaction_types[]` | e.g. `balance_allocation`. Repeated bracket-suffixed array param. |
| `states[]` | `cancelled`, `pending`, `processed`. |
| `currencies[]` | ISO 4217 three-letter codes. |
| `campaign_types[]` | `bundle`, `gift_card`, `marketplace`. |
| `team_ids[]` | Integer team ids. |
| `user_ids[]` | Integer user ids. |
| `page` | Default 1. |

Array parameters repeat with the bracket suffix, Rails-style:
`?states[]=processed&states[]=pending`.

**Watch two traps here.**

1. `list-transactions` declares `page` but **no `per_page`**, unlike the other list
   operations. Page size is neither controllable nor documented.
2. The write path addresses a team by **name** (`team_name` on a send) while this
   read path filters by **integer id** (`team_ids[]`), and no operation maps one to
   the other. Carry your own team name-to-id mapping.

Join sends to transactions through `payment_wallet_id`, currency, campaign type and
the user/team dimensions. There is no explicit foreign key from a transaction to a
send in the published contract.

## 5. Resolve contacts

`GET /contacts` (`list-contacts`) lists contacts in the organization, filterable by
`account_name` (the contact's company name), with `page` / `per_page`. The response
shape is declared as an empty object by the provider and is undocumented.

## Error handling

`400` on any of the list operations returns a body the contract documents as `{}`
— there is nothing to parse. When you get a 400, re-check by hand:

- dates are `YYYY-MM-DD`, not timestamps;
- `page` / `per_page` are integers;
- array filters use the `[]` suffix and only the enum values named in the parameter
  descriptions;
- `start_updated_date` and `end_updated_date` were sent as a pair.

`401` means the `Authorization: Bearer {api_token}` header is missing or the token
was revoked.

## Notes

- No outbound webhooks or events exist. Every status change is discovered by
  polling — use `start_updated_date`/`end_updated_date` for incremental runs rather
  than re-reading whole windows.
- No rate limits are published and no rate-limit headers are documented. Back off
  conservatively when paging large windows.
