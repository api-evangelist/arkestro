---
name: Run an Arkestro sourcing event
description: Create a sourcing event, attach the documents suppliers must submit, then track quote submissions and read the award — the end-to-end sourcing flow over Arkestro API V2.
api: openapi/arkestro-api-v2-openapi.yml
operations:
  - POST /api/v2/events
  - GET /api/v2/events
  - GET /api/v2/events/{id}
  - PATCH /api/v2/events/{id}
  - POST /api/v2/events/{event_id}/documents
  - GET /api/v2/events/{event_id}/documents
  - GET /api/v2/events/{event_id}/document_submissions
  - GET /api/v2/events/{id}/schedule
  - GET /api/v2/quote_submissions
  - GET /api/v2/quote_submissions/{id}
  - GET /api/v2/events/{id}/award
generated: '2026-08-06'
method: generated
---

# Run an Arkestro sourcing event

Arkestro API V2 has **no operationIds**, so every step below addresses an operation by
method and path. All paths are relative to `https://api.arkestro.com`.

## Before you start

- Authenticate with `X-Token: <PERSONAL_ACCESS_TOKEN>` on every request. The token is created
  at User Settings -> Personal Access Tokens, requires an **admin** user, and the API feature
  must have been enabled for the tenant on request — a 403 on every call usually means the
  feature is off, not that your token is wrong.
- Send `Accept: application/json`.
- **There is no idempotency key on this API.** Before retrying any POST, re-query the
  collection filtered by `external_id` to check whether your first attempt already landed.

## Steps

1. **Look up the business unit.** `GET /api/v2/business_units` and pick the `id` for the unit
   the event belongs to. This is read-only reference data.

2. **Check the event does not already exist.** `GET /api/v2/events?external_id=<YOUR_KEY>`.
   Always set your own `external_id` on creation so this check works. If a match comes back,
   stop and use that event rather than creating a second one.

3. **Create the event.** `POST /api/v2/events`. Set `name`, `description`, `currency`,
   `external_id`, the `business_unit`, and any `tags`. A new event starts in `draft`.
   Expect `201`. On `422` read the `error` string — it is a single freeform message, there is
   no field-level detail.

4. **Attach the documents suppliers must return.** For each one,
   `POST /api/v2/events/{event_id}/documents`. Uploads are **presigned-URL** style: the
   response is a `document_with_upload_url` payload, and you must then PUT the file bytes to
   the URL it returns. Posting multipart to Arkestro will not work. Use `submission_required`,
   `approval_required`, `consent_required` and `blocks_access` to control whether a supplier
   can proceed without returning the document.

5. **Confirm the document set.** `GET /api/v2/events/{event_id}/documents`.

6. **Update the event as it is configured.** `PATCH /api/v2/events/{id}`. Re-read with
   `GET /api/v2/events/{id}` to see the full event including `lots`, `line_items`,
   `custom_column_headers` and `custom_value_fields`.

7. **Read the schedule.** `GET /api/v2/events/{id}/schedule` returns start/end times, round
   count, round length and the nested `rounds`. This is **read-only** — the schedule is set in
   the Arkestro application, not over the API. Do not plan to launch an event purely through
   this API.

8. **Poll for supplier activity.**
   - `GET /api/v2/events/{event_id}/document_submissions` — what suppliers have returned, with
     `approved` and `rejection_reason`.
   - `GET /api/v2/quote_submissions?event_id=<id>` — quotes, with `round_number`, `status`,
     `total` and `submitted_at`. Filter with `submitted_at_gte` / `submitted_at_lte` to poll
     incrementally rather than re-reading the whole collection.
   - `GET /api/v2/quote_submissions/{id}` for the full detail including
     `quote_submission_lots` and `quote_submission_line_items`.

9. **Watch the event state.** `GET /api/v2/events/{id}` and read `status`. It moves through
   `draft` -> `open_for_bidding` / `open_for_questions` -> `closed` -> `ready_to_award` ->
   `awarded` or `unawarded`.

10. **Read the award.** Once `status` is `awarded`, `GET /api/v2/events/{id}/award` returns
    `awarded_at`, `awarded_line_items`, the winning `supplier`, `total_price` and
    `quoted_price_per_unit`. **Awarding is not an API operation** — the decision is made in the
    application and this endpoint only reads it back.

## Paging

Every collection uses `limit` and `offset`, and returns a `pagination` object with `limit`,
`offset`, `total` and `returned_count` next to the named data key. `total` is the count after
filtering, so you can size the loop up front. Order with `sort_by` and `sort_order`.

## Errors

Every failure returns `{"error": "<message>"}` and nothing else — no code, no field path.

| Status | What it means here |
|---|---|
| 400 | Malformed body or unparseable parameter value |
| 401 | `X-Token` missing, expired, or not valid for this tenant |
| 403 | Token valid but the user lacks rights, or the API feature is not enabled |
| 404 | Record missing **or** outside your tenant — the API does not distinguish these |
| 422 | Business-rule validation failed; read the message |
| 500 | Server error. The spec labels this "Bad Request" — that is a mislabel in the contract |

No `429` is declared anywhere and no rate-limit headers are documented. Back off
conservatively on repeated failures.
