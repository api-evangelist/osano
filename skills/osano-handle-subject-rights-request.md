---
name: osano-handle-subject-rights-request
description: Work a Data Subject Access Request (DSAR) in Osano end to end — find the request, read its action items, record what each data store returned, and close it out.
api: Osano Customer REST API
base_url: https://api.osano.com
auth: x-osano-api-key header
operations:
  - getDsars
  - getDsar
  - getDsarActionItems
  - getDsarActionItem
  - addDsarSummary
  - updateDsarActionItem
  - getRequestSummaries
  - postRequestActivityLog
  - updateDsar
generated: '2026-08-26'
method: generated
source: openapi/osano-customer-rest-api-openapi.yml (operationIds verified against the spec) plus https://developers.osano.com/webhooks/workflows
---

# Handle a Subject Rights Request

Osano is a system of record for regulated activity. Everything below writes to a
compliance audit trail, and **none of it can be deleted** — subject rights requests
are state-transitioned, never removed.

## Before you start

- Authenticate with `x-osano-api-key: <key>` on every call. Keys are generated at
  https://my.osano.com/api-keys and require admin privilege. API access may need to be
  enabled by Osano support depending on the plan tier.
- Base URL is `https://api.osano.com`. All paths are `/v1/...`.
- There is **no idempotency key** on this API. If a call times out, read state back
  before retrying — do not blind-retry a POST.

## 1. Find the request

```
GET /v1/subject-rights/requests?limit=100
```

`getDsars` returns a page of requests. Page with the opaque `next` token from the
response — send it **on its own**, because the original filters are encoded in it.

Then read the one you want:

```
GET /v1/subject-rights/requests/{dsarId}
```

`getDsar` returns 404 "The specified Subject Rights Request does not exist" if the id
is wrong or belongs to another customer. The response carries `dsarDetails` with the
subject's email, given name, family name and phone number — this is personal data.
Do not log it, do not echo it into a prompt you persist, and do not send it anywhere
outside the request-handling flow.

## 2. Read the action items

```
GET /v1/subject-rights/action-items
GET /v1/subject-rights/action-items/{actionItemId}
```

`getDsarActionItems` / `getDsarActionItem`. Each action item belongs to a request via
`dsarId` and usually names a `dataStore` — it is one system's share of the work.
Check `status` and `requestStatus` before touching anything.

## 3. Record what a data store returned

```
POST /v1/subject-rights/action-items/{actionItemId}/summaries
```

`addDsarSummary` uploads the evidence for that action item. Add an activity-log note
alongside it so a human reviewer can follow the reasoning:

```
POST /v1/subject-rights/action-items/{actionItemId}/activity-log
```

## 4. Complete the action item

```
PATCH /v1/subject-rights/action-items/{actionItemId}
{ "status": "COMPLETED" }
```

`updateDsarActionItem`. A **409 Conflict — "Action item already completed"** means
someone (or an earlier retry) already closed it. That 409 is the guard, not a failure:
stop, re-read, and do not force it.

## 5. Close the request

```
PATCH /v1/subject-rights/requests/{dsarId}
{ "status": "COMPLETED" }
```

`updateDsar`. This is the shape the Osano workflows guide itself publishes.

## Sending the summary to the subject — stop and confirm

```
POST /v1/subject-rights/requests/{requestId}/summary-notification
```

`sendSummaryNotification` packages a ZIP containing the PDF summary and every
associated file and **delivers it to the requestor**. It is irreversible, it leaves
Osano, and it contains personal data. Never call it autonomously. Get explicit human
confirmation first. A second call returns 409 — "The summary notification has already
been sent for this request".

Check what will be sent first:

```
GET /v1/subject-rights/requests/{requestId}/summary-notification
GET /v1/subject-rights/requests/{requestId}/summaries
```

## Errors

Every error is `application/json` with an untyped `{ "message": "..." }` body — no
error code, no type URI. Branch on the HTTP status, not the message text:

| Status | Meaning | Do |
|---|---|---|
| 400 | Bad Request | Fix the payload/params. `limit` maxes at 500 on generic lists. |
| 404 | Not Found | The record does not exist for this customer. Do not retry. |
| 409 | Conflict | Already done. Re-read state; do not force. |
| 429 | Too Many Requests | Back off. No `Retry-After` is sent — choose your own backoff. |
