---
name: Verify and handle Arkestro webhooks
description: Verify the HMAC-SHA256 signature on an inbound Arkestro webhook, survive secret rotation, deduplicate retries, and respond within the delivery timeout.
api: openapi/arkestro-api-v2-openapi.yml
operations: []
source: https://api.arkestro.com/api-docs/v2/openapi.yaml
generated: '2026-08-06'
method: generated
---

# Verify and handle Arkestro webhooks

Arkestro signs **every** outbound webhook with HMAC-SHA256. The rules below are Arkestro's own,
published as a `Webhooks` trait tag inside the API V2 OpenAPI. There are no API operations in
this skill — webhook subscriptions are configured in the Arkestro application, not over the API.

## Headers on every delivery

| Header | Meaning |
|---|---|
| `X-Arkestro-Signature` | One or more HMAC-SHA256 signatures, each `sha256=<hex>`, comma-joined with **no whitespace** |
| `X-Arkestro-Timestamp` | Unix epoch **seconds**, as a string. Part of the signed payload |
| `X-Arkestro-Idempotency-Key` | UUID for the delivery event. **Constant across every retry.** Not part of the signed payload |

## Verify, in order

1. **Capture the raw body.** You must have the exact bytes received. If your framework parses
   JSON before your handler runs, configure it to retain the raw body. A `JSON.parse` followed by
   a re-`stringify` will change the bytes and every signature check will fail.

2. **Rebuild the signed payload** as `"{X-Arkestro-Timestamp}.{raw_request_body}"` — the
   timestamp, a literal `.`, then the raw body.

3. **Compute** `HMAC-SHA256(secret, signed_payload)` as a **lowercase hex** digest and prefix it
   with `sha256=`.

4. **Split the header on commas and match any candidate.** Multiple signatures appear during
   secret rotation — one per secret Arkestro currently holds. Hold both secrets during a
   rotation and accept a match against any of them, or you will drop deliveries mid-rotation.

5. **Compare in constant time.** Use `crypto.timingSafeEqual`, `hmac.compare_digest`, or
   `ActiveSupport::SecurityUtils.secure_compare`. Arkestro publishes reference implementations
   in Node.js, Python and Ruby in the spec.

6. **Check the age** of `X-Arkestro-Timestamp` for replay protection. The published examples use
   a **300 second** tolerance. Fail closed on a malformed or missing timestamp.

## Deduplicate

`X-Arkestro-Idempotency-Key` is generated once when the event fires and stays the same across
all retries of that delivery. Key your processing on it.

Do **not** use the body's `request_id` for this — that changes on every attempt.

Note the direction: this solves duplicate *delivery*. It is not request idempotency for the
REST API, which has none. See `conventions/arkestro-conventions.yml`.

## Respond

- **Any 2xx acknowledges.** `200`, `201`, `202`, `204` are treated identically. The response
  body is ignored.
- **You have 10 seconds.** Enqueue slow work and return 2xx immediately.
- Anything else — non-2xx, a timeout, or a connection error — is a failed delivery.

## Retry behaviour to plan for

- **Retried**: `429`, any `5xx`, and transport errors (timeouts, connection failures).
- **Not retried, delivery discarded**: any `4xx` except `429`. If you return `400` or `422`
  because your own validation rejected the payload, that event is gone permanently.
- Up to **10 attempts** with exponential backoff. The final attempt lands roughly **4-5 hours**
  after the first. After that the delivery is permanently abandoned.

The practical consequence: if your handler hits a bug, return `500` — not `400` — so you keep
the retry window. Reserve `4xx` for payloads you have genuinely decided to discard.

## What is not published

There is no event-type catalog and no payload schemas in the public spec, and no AsyncAPI
document. Write your handler defensively against unknown event shapes and do not assume a
closed set of event types.
