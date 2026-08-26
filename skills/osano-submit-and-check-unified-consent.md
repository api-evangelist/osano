---
name: osano-submit-and-check-unified-consent
description: Collect, check and read a subject's consent through the Osano Unified Consent Core API, including Global Privacy Control signals and subject verification.
api: Osano Unified Consent Core API
base_url: https://uc.api.osano.com
auth: x-uc-api-key header (x-osano-api-key for subject create/update/merge)
operations:
  - POST /v2/consents
  - POST /v2/consents/gpc
  - GET /v2/consents/check/{subjectId}
  - GET /v2/consents/unified/{subjectRef}
  - GET /v2/consent-profiles/{hashedSubjectId}
  - GET /v2/collections
  - GET /v2/config
  - POST /v2/subjects/profile
  - POST /v2/subjects/send-code
  - POST /v2/subjects/profile/verify
  - POST /v2/subjects/merge
generated: '2026-08-26'
method: generated
source: openapi/osano-unified-consent-core-api-openapi.yml — this spec ships NO operationIds, so paths+methods are used verbatim instead of invented ids
---

# Submit and check Unified Consent

**The most important thing about this API: consent is append-only.** There is no
delete, no revoke and no withdraw operation in the public spec. A subject changing
their mind is a *new* consent record; the previous one stays as the audit trail.
Treat every write here as permanent.

The API is regionally distributed — US traffic is processed in `us-east-1`, everything
else in `eu-central-1`. You call one host either way: `https://uc.api.osano.com`.

## Authentication — two different keys

| Key | Header | Used for |
|---|---|---|
| Unified Consent API key | `x-uc-api-key` | Everything except subject mutation |
| Osano API key | `x-osano-api-key` | Creating, updating and merging subjects |

Both are generated in the Osano app. The UC key is the one safe to hand to a
browser-side client; the Osano key is not.

## 1. Know what you are collecting consent for

```
GET /v2/config
GET /v2/collections?configId=...
GET /v2/collections/{collectionId}
```

Collections resolve per `configId` **and jurisdiction** — the same config returns a
different set depending on where the subject is. A collection holds the privacy
protocols the subject is being asked about. Read this before you submit anything;
submitting against the wrong collection records the wrong agreement.

## 2. Submit a consent

```
POST /v2/consents
```

The body carries `collections[]` (each entry optionally with its own `jurisdiction`),
free-form `tags[]` (the spec's own example is `['ccpa', 'tcpa']`), `attributes` and
`origin`.

Two details worth getting right:

- `origin` is `'api'` or `'gpc'`. Use `'api'` for a normal submission.
- `attributes.ipAddress` and `attributes.userAgent` are **populated automatically**
  and anything you pass for those keys is overwritten. Don't bother sending them.

Returns 201 with an empty body.

## 3. Submit a Global Privacy Control consent

```
POST /v2/consents/gpc
```

Use this — not `/v2/consents` — when the signal came from the browser's GPC header.
It keeps the provenance distinguishable in the record, which is the whole point in an
enforcement conversation. Two optional headers override IP-based geolocation:

- `x-country-code-override`: a valid ISO 3166-1 country code
- `x-region-code-override`: a valid ISO 3166-2 region code

## 4. Check consent before you act

```
GET /v2/consents/check/{subjectId}
```

The cheap yes/no: has this subject given consent in a collection. Call this on the hot
path before you fire marketing or analytics.

Fuller reads:

```
GET /v2/consents/unified/{subjectRef}      # 400 "No consents found for the subject"
GET /v2/consent-profiles/{hashedSubjectId} # hashed id — usable from a browser
```

Note the failure shape: a subject with no consent returns **400 with an empty body**,
not 404 and not a payload. Handle it as "no record", not as an error to retry.

## 5. Subjects and verification

```
POST /v2/subjects/profile                        # create a profile      (Osano API key)
POST /v2/subjects/profile/verification-code      # create a code
POST /v2/subjects/send-code                      # email the code
POST /v2/subjects/send-code/sms                  # text the code
POST /v2/subjects/profile/verify                 # verify by email
POST /v2/subjects/profile/verify/sms             # verify by SMS
GET  /v2/subjects/{subjectRef}
GET  /v2/subjects/{id}/profile
```

`send-code` and `send-code/sms` send a real message to a real person. They are not
retry-safe and there is no idempotency key — a repeated call is a second email or a
second text to the subject.

## 6. Merging subjects — irreversible

```
POST /v2/subjects/merge
```

Merges an anonymous subject into a verified one. **There is no unmerge.** Require
human confirmation, and be certain the two references are the same person.

## What this API does not give you

- No error codes — failures are bare statuses, often with no body.
- No 401/403 documented anywhere, despite requiring a key.
- No rate-limit headers, no `Retry-After`, no published limits.
- No operationIds in the spec, so generated clients will name methods from paths.
