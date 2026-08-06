---
name: Pull Arkestro event analytics incrementally
description: Read Arkestro's dbt-built procurement analytics marts — buyer leaderboards, quotes, supplier invitation statuses and survey results — and poll them incrementally with refreshed_after instead of re-reading everything.
api: openapi/arkestro-api-v2-openapi.yml
operations:
  - GET /api/v2/event_analytics/buyer_leaderboards
  - GET /api/v2/event_analytics/quotes
  - GET /api/v2/event_analytics/supplier_invitation_statuses
  - GET /api/v2/event_analytics/survey_results
  - POST /api/v2/event_analytics/metrics/export_task
  - POST /api/v2/event_analytics/rounds/export_task
generated: '2026-08-06'
method: generated
---

# Pull Arkestro event analytics incrementally

Paths are relative to `https://api.arkestro.com`. Authenticate with `X-Token`.

## What these endpoints are

They are **not** normalized resources. They are denormalized reporting marts — every record
carries `dbt_run_id` and `dbt_run_ts`, so you are reading dbt model output directly. That has
two consequences you must design around:

- Rows change only when the marts are rebuilt, not when the underlying event changes. Polling
  faster than the rebuild cadence returns identical data.
- `refreshed_after` exists precisely so you can ask for rows touched by a newer run.

## The four marts

| Endpoint | Grain |
|---|---|
| `GET /api/v2/event_analytics/buyer_leaderboards` | One row per buyer, with 28-day and 90-day rolling windows |
| `GET /api/v2/event_analytics/quotes` | One row per quote |
| `GET /api/v2/event_analytics/supplier_invitation_statuses` | One row per supplier invitation |
| `GET /api/v2/event_analytics/survey_results` | One row per survey answer |

All four return a `data` array plus the standard `pagination` object.

## Incremental pull

1. Store the highest `dbt_run_ts` you have seen per mart.
2. On each poll, call the endpoint with `refreshed_after=<that timestamp>`.
3. Page with `limit` and `offset` until `pagination.offset + pagination.returned_count` reaches
   `pagination.total`.
4. Sort with `sort_by` and `sort_order` so paging is stable across the walk.

`refreshed_after` is available on `quotes`, `supplier_invitation_statuses` and
`survey_results`. It is **not** a parameter on `buyer_leaderboards` — that mart must be pulled
in full, which is tolerable because its grain is one row per buyer.

## Filters

Narrow before you page, not after:

- `business_unit_id`
- `event_state`
- `event_tag_names`
- `owner_full_name` / `creator_user_full_name`
- `supplier_org_name`
- `start_date` and `end_date` (on `supplier_invitation_statuses` and `survey_results`)
- `data_mart_limit` on `buyer_leaderboards`

## Reading the leaderboard fields

`buyer_leaderboards` carries paired current and previous windows so you can compute movement
without keeping history yourself: `baseline_total_28d` against `baseline_total_prev_28d`,
`baseline_total_90d` against `baseline_total_prev_90d`, `event_count_28d` against
`event_count_prev_28d`, plus precomputed `baseline_trend_28d`, `baseline_trend_90d` and
`event_count_trend_90d`. Use the precomputed trend fields rather than deriving your own —
they encode Arkestro's own definition of the comparison window.

On the quotes mart, respect the boolean qualifiers before aggregating:
`is_included_in_reporting`, `is_awarded_quote`, `is_buyer_side_quote` and `is_opted_out`.
Summing `quotes` without filtering on `is_included_in_reporting` will not match what Arkestro
reports in its own UI.

## Exports

`POST /api/v2/event_analytics/metrics/export_task` and
`POST /api/v2/event_analytics/rounds/export_task` kick off export tasks rather than returning
rows inline. These are **writes** — and since the API has no idempotency key, a retried export
POST starts a second task. Record the response before retrying.

## Errors

`{"error": "<message>"}` on every failure. `401` bad token, `403` insufficient rights or the
API feature not enabled, `422` invalid filter combination, `500` server error (labelled "Bad
Request" in the spec). No `429` is declared, so there is no documented rate-limit signal to
back off against — space your polls to the mart rebuild cadence rather than probing for a limit.
