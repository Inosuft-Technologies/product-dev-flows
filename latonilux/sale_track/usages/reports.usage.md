# Reports Usage – SaleTrack Dashboards (Admin UI)

This document describes how to use the SaleTrack reporting pages in the admin app and how they relate to backend endpoints.

## Common Behaviour

- **Filters drive everything**  
  - Most reports require at least a **date range** (Sales, Allocations) and optionally a **branch**.  
  - If filters are missing or too narrow, you may see empty tables with an explanatory EmptyState.
- **Branch scoping**  
  - Branch filtering honours the same branch-scope rules as the rest of the app.  
  - Global admins can see all branches; scoped admins see only their permitted branches.
- **Exports**  
  - Each report page includes an **Export Report** button using `/v1/reports/export`.  
  - Exports respect the current filters (branch, date) and return CSV data when available.  
  - If there is no data to export for the selected filters, the UI shows a toast:  
    “No data available to export for the selected filters.”

---

## Sales Reports Page

**Route:** `/dashboard/reports/sales`  
**Backend:** `POST /v1/reports/sales`

- **Filters**
  - **Group By:** `day | branch | resource` – controls the report dimension.
  - **Branch:** optional; uses `BranchSelector`.
  - **Date Range:** required; uses `DateRangeFilter`. Without both `start` and `end`, no query is sent.
- **KPIs**
  - **Total Revenue** – sum of `revenue` across all rows.  
  - **Total Orders** – total document count (orders + bookings).  
  - **Total Quantity** – sum of all `quantity` values.
- **Table**
  - Column 1 adapts to grouping:
    - `Date` (when grouped by day – `YYYY-MM-DD`).
    - `Branch` (branch name/code).
    - `Resource` (order resource: `order`, `event`, `booking`, `facility`, etc.).
  - Remaining columns:
    - `Orders` – count of documents per group.
    - `Quantity` – total units/portions sold or booked.
    - `Revenue` – total `summary.total` per group.
- **Data Source**
  - Aggregates **orders** with status `paid` or `completed`.  
  - Aggregates **bookings** with status `confirmed`, `engaged` or `completed`.  
  - Both are merged into a unified dataset on the selected grouping key.

---

## Inventory Reports Page

**Route:** `/dashboard/reports/inventory`  
**Backend:**  
- `POST /v1/reports/stock/summary`  
- `POST /v1/reports/valuation`  
- `POST /v1/reports/lots/aging`

- **Filters**
  - **Branch:** required for branch-scoped views; uses `BranchSelector`.
- **Stock Status Summary**
  - Shows overall totals and per-branch breakdown of:
    - In Stock (Qty)
    - Low Stock (Qty)
    - Out of Stock (Items)
  - Backed by `ReportService.getStockSummary`.
- **Inventory Valuation**
  - Toggle between:
    - **By Branch** – quantity and value per branch.  
    - **By Item** – quantity and value per item.
  - Backed by `ReportService.getInventoryValuation`.
- **Lot Aging**
  - Buckets open lots into:
    - `0-30`, `31-60`, `61-90`, `90+` days.  
  - Shows **Lots**, **Quantity**, and **Value** per bucket.  
  - Backed by `ReportService.getLotAging`.
- **Export**
  - Export button maps to `resource: 'stock-lots'` and returns a CSV for the current branch/date snapshot.

---

## Allocation Reports Page

**Route:** `/dashboard/reports/allocations`  
**Backend:** `POST /v1/reports/allocations/variance`

- **Filters**
  - **Branch:** optional; uses `BranchSelector`.
  - **Date Range:** required; uses `DateRangeFilter`.
  - Grouping is currently fixed to **allocation**.
- **KPIs**
  - **Reserved** – total reserved units.  
  - **Confirmed** – total confirmed (consumed) units.  
  - **Released** – total released back to stock.  
  - **Net Reserved** – `reserved - confirmed - released`.
- **Table**
  - Columns:
    - Allocation, Department, Branch.
    - Reserved, Confirmed, Released, Net Reserved (all numeric, right-aligned).
  - Backed by `ReportService.getAllocationVariance`.
- **Export**
  - Export button uses `resource: 'allocations'` and respects the same branch/date filters.

---

## Dashboard Overview – Embedded Sales Snapshot

**Route:** `/dashboard`  
**Backend:** `POST /v1/reports/sales`

- **Filters**
  - Uses the same **BranchSelector** and **DateRangeFilter** pattern as Sales Reports.  
  - Filters drive the embedded Sales snapshot and export.
- **KPIs**
  - **Total Sales** – same as Sales Reports’ total revenue.  
  - **Total Staffs** – mapped to total **orders** count from sales report (`ordersCount`).  
  - **Total Members** – mapped to total **bookings** count from sales report (`bookingsCount`).
- **Chart**
  - Uses `SalesLineChart` or `SalesBarChart` depending on user toggle.  
  - X-axis: grouped by **day** (current implementation).  
  - Y-axis: **revenue** per day.
- **Export**
  - “Export Report” reuses the Sales export flow and reflects the current dashboard filters.

---

## Troubleshooting

- **No data appears**
  - Confirm a **date range** is selected (Sales, Allocations).  
  - Confirm a valid **branch** is selected and the user has access to it.  
  - Check that there are completed/paid orders or qualifying bookings in the period.
- **CSV export returns empty**
  - Same root causes as above; CSV respects current filters.  
  - If backend returns `null` CSV, the UI will show an info toast instead of downloading a file.

