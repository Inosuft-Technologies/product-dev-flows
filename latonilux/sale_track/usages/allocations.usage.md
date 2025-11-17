# Allocations API

Documentation for the allocation reserve/confirm/release workflow introduced in SaleTrack Step 8.

## Common

- Base route: `/v1/allocations`.
- Authentication required. All operations are restricted to `superadmin` and `admin` roles unless otherwise noted.
- Each allocation reserves inventory via `ProductService.reserveStock`; usage confirmation triggers `confirmStock`, and rollbacks call `releaseStock`.
- Responses follow the standard envelope `{ error, errors, data, message, status }`.

---

## List & Fetch

Use these endpoints to monitor existing allocations. They provide snapshot visibility into what each department has reserved and how much remains. Most reporting dashboards rely on this list, so filters mirror the reporting criteria (status, branch, department).
- `GET /v1/allocations`
  - Supports paging (`page`, `limit`), sorting, and filtering through the advanced results middleware.
- `GET /v1/allocations/:id`

- `POST /v1/allocations/search`
  - Body: `{ "key": "text" }` to search by code/description.
- `POST /v1/allocations/filter`
  - Body accepts `status`, `branch`, and `department` fields (all optional).

---

## Create Allocation

Creating an allocation locks stock for a non-sales workflow (housekeeping, spa, maintenance). Think of it as a mini purchase order that draws down over time. Every line reserves lots via `ProductService.reserveStock` and records policy metadata for soft/hard caps.
- `POST /v1/allocations/:branchId`
  - Body example:
    ```json
    {
      "staffID": "STAFF-123",
      "description": "Housekeeping monthly stock",
      "policy": {
        "capType": "soft",
        "maxQuantity": 120,
        "notes": "Alert at 120 units"
      },
      "items": [
        { "code": "PROD-LAUNDRY", "quantity": 40 },
        { "code": "PROD-GLOVES", "quantity": 80 }
      ]
    }
    ```
  - Behaviour:
    - Validates product codes (must be internal/non-sale items).
    - Reserves stock for each product immediately; insufficient stock returns a 400 error.
    - Attaches warnings when soft caps are exceeded.

---

## Update & Usage

After an allocation is created, operations teams log actual consumption through the `use` endpoint. This confirms the previously reserved quantities and writes ledger entries with type `allocation-confirm`. If stock needs to be returned, the `release` endpoint reverses the reservation. Updating the allocation keeps descriptions/policies current.
- `PUT /v1/allocations/use/:id`
  - Body: `{ "code": "PROD-LAUNDRY", "quantity": 10, "idempotencyKey": "issue-20241103-1" }`
  - Confirms reserved stock and writes ledger entries; supports idempotency to avoid double consumption.
- `PUT /v1/allocations/release/:id`
  - Body mirrors `use` but returns unused stock to availability.
- `PUT /v1/allocations/:id`
  - General update endpoint (description, status, etc.).

---

## Deletion

Deletion is a last resort for allocations that should never have been created. It ensures any still-reserved stock is released back to inventory before the allocation document is removed.
- `DELETE /v1/allocations/:id`
  - Cancels an allocation. If unused stock remains, the service releases reservations before removal.

---

## Status & Reporting

Status fields power the allocation variance reports. They give at-a-glance insight into whether a department has enough stock left (`available`), is approaching the minimum (`limited`), or is out (`depleted`). Soft-cap warnings bubble up in the API response so UIs can alert managers immediately.
- Allocation documents track `items[].quantity` (remaining) and `items[].used` (consumed).
- `AllocationService.updateStatus` auto-classifies allocations as `available`, `limited`, or `depleted` based on remaining quantity.
- Soft-cap warnings are returned in the create response (`data.warnings[]`) but are not persisted in Mongo.
