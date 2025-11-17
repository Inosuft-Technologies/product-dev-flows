# Reports API

This document outlines the reporting endpoints added for SaleTrack, their parameters, and how to export results as CSV.

## Common

- All endpoints require authentication and appropriate role access (admin/manager/staff depending on route).
- Branch scoping applies: requests are restricted to the caller's branch scope unless the caller is global.
- Add `?format=csv` to return a CSV string instead of JSON rows.

---

## Stock Summary

Provides an at-a-glance view of inventory health by status (in stock, low stock, etc.). Operations teams use this report to monitor which branches need replenishment. It pulls aggregated data from `ProductRepository.summarizeStock` and honours branch scoping.
- Route: `POST /v1/reports/stock/summary`
- Body:
  - `branchID?: string`
  - `status?: string | string[]` (one of: in-stock, low-stock, out-stock, not-stocked)
  - `includeBranchBreakdown?: boolean` (default: true)
- Response (JSON):
  - `data.totals` – overall counts by status and overall
  - `data.branches[]` – per-branch totals if breakdown enabled

---

## COGS (Cost of Goods Sold)

Calculates the cost of goods sold by analysing `StockLedger` consumption entries. Finance can pivot by day, item, or branch to understand profitability trends. Pair this report with sales exports for margin analysis.
- Route: `POST /v1/reports/cogs`
- Query:
  - `format?: csv` – include to receive CSV as part of result
- Body:
  - `start?: string` – `YYYY-MM-DD`
  - `end?: string` – `YYYY-MM-DD`
  - `branchID?: string`
  - `groupBy?: 'day' | 'item' | 'branch'` (default: `day`)
- Response (JSON):
  - `data.groupBy` – grouping used
  - `data.totals` – total `quantity`, `cogs`
  - `data.rows[]` – grouped rows
  - `data.csv` – csv data
- Response (CSV):
  - For `day`: columns `date,quantity,cogs`
  - For `item`: columns `item_id,item_name,quantity,cogs`
  - For `branch`: columns `branch_id,branch_name,branch_code,quantity,cogs`

---

## Allocation Variance

Highlights how allocations are performing relative to their reserved quantities. It compares reserved, confirmed, and released amounts so department heads can spot under/over usage. Data comes from allocation ledger entries.
- Route: `POST /v1/reports/allocations/variance`
- Query:
  - `format?: csv`
- Body:
  - `start?: string`
  - `end?: string`
  - `branchID?: string`
  - `groupBy?: 'allocation' | 'department'` (default: `allocation`)
- Response (JSON):
  - `data.totals` – sums of reserved, confirmed, released, netReserved
  - `data.rows[]` – columns: `allocationId, allocationCode, departmentName, branchName, reserved, confirmed, released, netReserved`
- Response (CSV):
  - Columns `allocation_id,allocation_code,department,branch,reserved,confirmed,released,net_reserved`

---

## Inventory Valuation

Shows the book value of open stock lots using quantity remaining × unit cost. Finance can group by branch to gauge on-hand value or by item to see the highest-cost ingredients.
- Route: `POST /v1/reports/valuation`
- Query:
  - `format?: csv`
- Body:
  - `branchID?: string`
  - `groupBy?: 'branch' | 'item'` (default: `branch`)
- Response (JSON):
  - `data.totals` – total `quantity`, `value`
  - `data.rows[]` – grouped rows (branch or item)
- Response (CSV):
  - For `branch`: `branch_id,branch_name,branch_code,quantity,value`
  - For `item`: `item_id,item_name,quantity,value`

---

## Lot Aging

Buckets lots by age in days so warehouse staff can identify items nearing expiry or stagnating. Useful for enforcing FEFO (first-expired-first-out) policies.
- Route: `POST /v1/reports/lots/aging`
- Query:
  - `format?: csv`
- Body:
  - `branchID?: string`
- Response (JSON):
  - `data.totals` – total counts and value across buckets
  - `data.rows[]` – columns: `bucket, lots, quantity, value` for buckets `0-30, 31-60, 61-90, 90+`
- Response (CSV):
  - Columns `bucket,lots,quantity,value`

---

## Caching & Freshness

Each endpoint caches results to keep reporting fast. Inventory mutating operations call `ReportService.bumpGlobalVersion`/`bumpBranchVersion` so stale cache entries are automatically invalidated.
- Results are cached in Redis for a short TTL (default 10 minutes) with versioned keys.
- Inventory changes (new lots, reserves/confirms/releases, adjustments) bump cache versions so new results appear immediately.
