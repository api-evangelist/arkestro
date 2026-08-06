---
name: Sync suppliers and the spend catalog into Arkestro
description: Keep Arkestro's supplier organizations, supplier contacts, corporate categories, items and purchase orders in step with an upstream ERP or supplier master, using external_id as the correlation key.
api: openapi/arkestro-api-v2-openapi.yml
operations:
  - GET /api/v2/supplier_organizations
  - POST /api/v2/supplier_organizations
  - GET /api/v2/supplier_organizations/{id}
  - PATCH /api/v2/supplier_organizations/{id}
  - GET /api/v2/supplier_contacts
  - POST /api/v2/supplier_contacts
  - PATCH /api/v2/supplier_contacts/{id}
  - DELETE /api/v2/supplier_contacts/{id}
  - GET /api/v2/corporate/categories
  - POST /api/v2/corporate/categories
  - PATCH /api/v2/corporate/categories/{id}
  - GET /api/v2/corporate/items
  - POST /api/v2/corporate/items
  - PATCH /api/v2/corporate/items/{id}
  - GET /api/v2/corporate/purchase_orders
  - POST /api/v2/corporate/purchase_orders
  - PATCH /api/v2/corporate/purchase_orders/{id}
generated: '2026-08-06'
method: generated
---

# Sync suppliers and the spend catalog into Arkestro

This is an upsert loop against Arkestro API V2. Paths are relative to
`https://api.arkestro.com`; authenticate with `X-Token` on every request.

## The rule that makes this safe

Arkestro has **no request idempotency**. There is no `Idempotency-Key` header on any of the
46 operations. The only duplicate protection available is the `external_id` field, which is
present and filterable on supplier organizations, supplier contacts, items, categories and
purchase orders.

So the loop for every record is always:

1. `GET <collection>?external_id=<your key>`
2. If `pagination.total` is `0` -> `POST <collection>` with `external_id` set
3. If it is `1` -> `PATCH <collection>/{id}` with the changed fields
4. If it is greater than `1` -> stop and alert. A duplicate already exists and the API cannot
   resolve it for you.

Never retry a `POST` blindly after a timeout. Re-run step 1 first.

## Order matters

Load in dependency order, because the child records reference the parents:

1. **Categories** — `POST /api/v2/corporate/categories` (`name`, `code`, `description`).
2. **Items** — `POST /api/v2/corporate/items` (`name`, `code`, `sku`, `description`,
   `category`). Items belong to a category.
3. **Supplier organizations** — `POST /api/v2/supplier_organizations` (`name`, `firm`,
   `domain`, `status`, `external_id`, and the address fields `line_one`, `line_two`, `city`,
   `state`, `postal_code`, `country`).
4. **Supplier contacts** — `POST /api/v2/supplier_contacts` (`first_name`, `last_name`,
   `email`, `phone`, `title`, `supplier_organization`). Contacts belong to an organization.
5. **Purchase orders** — `POST /api/v2/corporate/purchase_orders` (`name`, `title`, `code`,
   `currency`, `price_per_unit`, `quantity_of_items`, `items_per_unit`, `purchased_on`,
   `spend_category`, `external_id`, `external_division_id`, `purchasing_org_id`).

## Deletion asymmetry — read this before designing the sync

The delete surface is uneven, and a naive two-way sync will break on it:

| Resource | Delete available |
|---|---|
| Categories | yes — `DELETE /api/v2/corporate/categories/{id}` |
| Items | yes — `DELETE /api/v2/corporate/items/{id}` |
| Purchase orders | yes — `DELETE /api/v2/corporate/purchase_orders/{id}` |
| Supplier contacts | yes — `DELETE /api/v2/supplier_contacts/{id}` |
| **Supplier organizations** | **no** |

There is no `DELETE /api/v2/supplier_organizations/{id}`. To retire a supplier, `PATCH` its
`status` instead and leave the record in place. Design the sync so that removal upstream maps
to a status change here, not a delete.

## Paging a full reconciliation

Walk each collection with `limit` and `offset`. Read `pagination.total` from the first page to
size the walk, and `pagination.returned_count` to know when you are done. Sort deterministically
with `sort_by` and `sort_order` so pages do not shift under you mid-walk.

## Errors

All failures return `{"error": "<message>"}` only.

- `401` — bad or missing `X-Token`.
- `403` — valid token, insufficient rights, or the API feature is not enabled for the tenant.
  Note that `GET`/`POST /api/v2/supplier_organizations` declare `404` where the other
  collections declare `403`, so handle both on that resource.
- `404` — record missing or belongs to another tenant; the API does not distinguish them.
- `422` — validation failed. The message is a single freeform string with no field path, so log
  the request body alongside it or you will not be able to tell which field was rejected.
- `500` — server error, despite being described as "Bad Request" in the spec.
