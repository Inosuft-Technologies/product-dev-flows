# Deferred Frontend Work – Allocation Insights

The following enhancements were identified during Frontend Workstream Step 6 but intentionally deferred so we could ship the rebuilt allocation flows. Each item documents the scope, dependencies, and open questions to resolve before implementation.

## 1. Ledger-Backed Plan vs Actual Visuals  
**Screens:** `latonilux-admin/src/pages/dashboard/core/allocations/NewAllocation.tsx` (Insights tab) and `latonilux-admin/src/pages/dashboard/core/allocations/AllocationDetails.tsx` (Insights tab)

- **Goal:** Replace the placeholder copy in both Insights tabs with real plan-vs-actual visualizations (table or chart) based on ledger activity (`stockLedger`) tied to each allocation or branch/department slice.
- **Data Needs:** `getStockLedger` or a slimmer reporting endpoint that can be filtered by allocation ID, branch ID, department ID, and time range. We currently only show draft totals from the UI state.
- **Open Questions:**
  - Should the insights show only the live allocation’s ledger entries, or also historical allocations for that department?
  - What aggregation window is acceptable (e.g., last 30 days, entire allocation life)?
  - Is a table sufficient, or do we require charts (bar/line) per leadership?
- **Tech Notes:** Pulling raw ledger data directly on these pages might be heavy. Prefer a backend summary endpoint (e.g., `/reports/allocations/plan-vs-actual`) that already aggregates by item.
- **Next Steps:** Define the reporting contract, add the hook (`useAllocations` or `useInventoryReports`), and update both tabs to render the data with EmptyState fallbacks.

## 2. Cross-Screen Policy Alerts  
**Screens:** Allocation list pages (e.g., `latonilux-admin/src/pages/dashboard/core/allocations/Allocations.tsx` when reintroduced) and `AllocationDetails.tsx` policy/status cards.

- **Goal:** Surface policy cap warnings anywhere allocations are referenced (lists, dashboards) so ops can see violations without drilling into details.
- **Data Needs:** Policy metadata in list endpoints (`GET /allocations`) and possibly a lightweight flag (`capExceeded: true`).
- **Open Questions:** Do we badge only hard-cap breaches, or also soft caps? Should alerts be aggregated per department?
- **Next Steps:** Update backend DTOs, extend list components to show the badge, and document behavior in `coding-style.md`.

## 3. Historical Lifecycle Timeline Enhancements  
**Screens:** `AllocationDetails.tsx` Insights tab (Lifecycle card) and any future allocation overview widgets.

- **Goal:** Enrich the lifecycle card with actor names and contextual notes for each step (reserve/confirm/release), ideally pulled from the requisition timeline API once that service stabilizes.
- **Data Needs:** Stable `getAllocationRequisitionTimeline` responses with `createdBy` info for each event.
- **Next Steps:** Coordinate with backend on the timeline payload, then expand the Insights card to show avatars/badges per actor and filter noise (e.g., deduplicate reserve events).

## 4. Shareable Insights Deep Links  
**Screens:** `NewAllocation.tsx`, `AllocationDetails.tsx`, and any other allocation route that uses storage-backed tabs.

## 5. Menu Insights Enhancements (Deferred from Frontend Step 7)  
**Screens:** `latonilux-admin/src/pages/dashboard/core/menu/MenuDetails.tsx`, `MenuList.tsx`  
- **Goal:** Surface recipe linkages, portion-unit definitions, and live availability indicators directly on menu pages.  
- **Reason for deferral:** We’re prioritizing order insights (Step 7 items 1, 2, and 4) first; adding availability widgets requires additional inventory hooks and UI effort.  
- **Follow-up:** Revisit once order detail work is stable—this may involve reusing the insights tab pattern and pulling stock data via `useStockLots`/`useStockLedger`.

- **Goal:** Persist Insights tab/filter state to query params so managers can share URLs that open directly on the insights view (similar to Step 5 filters).
- **Open Questions:** Should query params track which card is expanded? How do we keep them in sync with storage-backed tab indexes?
- **Next Steps:** Align with the routing pattern used for StockLots/StockLedger filters, update `storage.keep` usage, and document in coding standards.

## 6. Auto-Priced Menus UI & Pricing Management (Deferred)

**Screens:** `latonilux-admin/src/pages/dashboard/core/menu/MenuList.tsx`, `MenuDetails.tsx`, and order-facing read-only views such as `OrderDetails.tsx`.

- **Goal:** Expose backend auto-pricing for menus in the admin UI so operators can see how sell prices are derived from recipes, portions, and inventory costs, and make controlled adjustments.
- **Scope (draft):**
  - Show auto-calculated price, final sell price, and margin % on Menu list/detail pages.
  - Add a lightweight “Pricing” section/tab in `MenuDetails` that:
    - Surfaces cost-per-portion from the recipe/portion builder.
    - Shows suggested (auto) price and allows an optional manual adjustment/surcharge field.
  - Update order-facing views to display the final menu price sourced from the pricing metadata (read-only), keeping existing POS UX intact.
- **Dependencies / Open Questions:**
  - Confirm final backend contract for `menuPricing` or equivalent fields (structure, overrides).
  - Decide who is allowed to override prices (role/permission model).
  - Decide whether adjustments are absolute values, percentage markups, or both.

This work is deferred until the auto-pricing backend contracts are fully stable and operations are ready to change how menu prices are managed.***
