---
name: reachdesk-gdpr-request
description: >-
  File a GDPR data-subject request against Reachdesk — erase or export all data
  held about a person, keyed by their email address — and check its status. Use
  when honouring a right-to-erasure or subject-access request that covers gifting
  records.
api: reachdesk:reachdesk-api
base_url: https://app.reachdesk.com/api/v2
operations:
  - create-gdpr-request
  - get-gdpr-request
generated: '2026-08-13'
method: generated
source: openapi/reachdesk-api-openapi.yml
---

# File a GDPR data-subject request with Reachdesk

Reachdesk exposes data-subject rights as a first-class API surface, which is
uncommon — most platforms handle erasure and export through a support ticket.

## Erasure is permanent and irreversible

`request_type: "erase_subject"` deletes a data subject's records. There is no undo
in the API. **Never issue this on an agent's own initiative.** It should be
triggered only by a verified, human-authorised data-subject request, and the
authorisation should be recorded outside Reachdesk before the call is made.

## Steps

### 1. Create the request

`POST /gdpr/requests` (`create-gdpr-request`)

```
POST https://app.reachdesk.com/api/v2/gdpr/requests
Authorization: Bearer {api_token}
Content-Type: application/json

{
  "request": {
    "request_type": "erase_subject",
    "subject": "john.doe@example.com"
  }
}
```

| Field | Values |
|---|---|
| `request.request_type` | `erase_subject` — permanently delete the subject's data. `export_subject` — produce an export of it. |
| `request.subject` | The **email address** of the data subject. This is the only identifier accepted. |

Note the nesting: the payload is wrapped in a `request` object.

The operation declares only a `200`. No error responses are documented at all, so
capture and surface whatever non-2xx you receive verbatim.

### 2. Check status — not yet available

`GET /gdpr/requests/{id}` (`get-gdpr-request`) is published in the contract to
"Get the information and status about an existing GDPR Request", but the provider's
own description marks it **"(Coming Soon)"**.

Do not build a workflow that depends on reading the status back. Until Reachdesk
ships it:

- record the request locally at the moment you submit it, with the subject email,
  the request type, and a timestamp;
- confirm completion out of band with Reachdesk support (`support@reachdesk.com`)
  rather than by polling;
- do not treat the absence of an error as proof the erasure completed.

### 3. Preserve your own evidence

Because there is no readable status and no event or webhook, your submission record
is the only audit trail on your side. Keep the `request_id`-equivalent handle the
API returns, the exact subject string you sent, and the response body.

## Related

- Reachdesk's own privacy documentation: <https://www.reachdesk.com/privacy-policy>
- Trust centre: <https://trust.reachdesk.com/>
- Reachdesk's GitHub organization carries a fork of `gdpr_admin`, a Rails engine
  for automating GDPR processes, which is plausibly the machinery behind these
  endpoints.
