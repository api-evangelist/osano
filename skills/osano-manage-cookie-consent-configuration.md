---
name: osano-manage-cookie-consent-configuration
description: Read, change and publish an Osano Cookie Consent (CMP) configuration and its rules, and audit who changed what.
api: Osano Customer REST API
base_url: https://api.osano.com
auth: x-osano-api-key header
operations:
  - getConfigs
  - getConfig
  - createConfig
  - updateConfig
  - publishConfig
  - getRules
  - createRules
  - updateRule
  - deleteRule
  - getTattles
  - getAuditLog
generated: '2026-08-26'
method: generated
source: openapi/osano-customer-rest-api-openapi.yml (operationIds verified against the spec)
---

# Manage a Cookie Consent configuration

A CMP configuration is what a visitor actually sees. `configId` is also the path
segment in the script tag your customer's site loads:
`https://cmp.osano.com/{configId}/osano.js`. Changing and publishing a configuration
changes a live banner on a live site.

## 1. Find the configuration

```
GET /v1/cookie-consent/configs
GET /v1/cookie-consent/configs/{configId}
```

`getConfigs` accepts `limit` up to 1000. `getConfig` returns the detailed
configuration, including the whole `configuration` object: consent model, jurisdiction
behaviour, `gpcSupport` (whether the Global Privacy Control signal is honored),
Google Consent Mode, iframe/script blocking, and the full colour `palette` for the
banner, drawer and Do-Not-Sell dialogs.

## 2. Change it

```
PATCH /v1/cookie-consent/configs/{configId}
```

`updateConfig`. Two constraints the spec states outright:

- Microsoft UET consent sharing **"is incompatible with the use of IAB TCF 2.x
  consent signals"** — do not enable both.
- `gpcSupport: true` means the GPC signal is honored; it has its own rendered
  indicator with dedicated palette colours (`dialogGpcBackgroundColor` and siblings).

## 3. Publish — this is one-way

```
POST /v1/cookie-consent/configs/{configId}/publish
```

`publishConfig`. **There is no unpublish and no rollback operation in this API.**
Returns 204 on success, 409 if it conflicts with current state, 429 if you publish
too often. Require human confirmation before calling it, and read the config back
with `getConfig` first so you know exactly what you are pushing live.

## 4. Rules

```
GET  /v1/cookie-consent/configs/{configId}/rules
POST /v1/cookie-consent/rules
PATCH  /v1/cookie-consent/rules/{ruleId}
DELETE /v1/cookie-consent/rules/{ruleId}
```

`getRules`, `createRules`, `updateRule`, `deleteRule`. A rule carries `title`, `rule`,
`ruleType`, `classification`, `disclosure` and `vendorName`. Unlike configurations,
rules **can** be deleted (204). `createRules` can return 409 and 429.

## 5. What Osano found on the site

```
GET /v1/cookie-consent/configs/{configId}/discoveries
```

`getTattles` lists the cookies and scripts Osano observed. This is where an
unclassified vendor shows up before you write a rule for it.

## 6. Who changed what

```
GET /v1/cookie-consent/audit-log?startDate=...&endDate=...&limit=50
```

`getAuditLog`. `startDate` is inclusive, `endDate` is exclusive, both UTC ISO-8601.
Filter with `changeType` — one of `text_customization`, `style`, `iab`, `setting`,
`rule` — or with the machine-readable `eventTypes`, or by `actor`. `limit` defaults
to 50, max 200. Page with `next` **on its own**: the filters are baked into the token.

## Safety notes

- No idempotency key exists. `createConfig` and `createRules` retried blindly create
  duplicates.
- No configuration delete operation exists — you can create and update, but not remove.
- Errors are untyped `{ "message": "..." }` JSON. Branch on status code.
