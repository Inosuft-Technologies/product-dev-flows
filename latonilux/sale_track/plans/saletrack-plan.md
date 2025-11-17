# SaleTrack Implementation Plan

This plan outlines the detailed backend and admin frontend workstreams required to deliver the SaleTrack initiative. Each step explains the task, the rationale, and how it supports the project goals defined in `saletrack.spec.md`.

---

## Backend Workstream

1. [x] **Set Up Domain Foundations**
   - Task: Introduce `itemFamilies`, `units`, and `unitConversions` schemas, validators, and admin-only CRUD endpoints. Seed core unit families (grains, spirits, housekeeping consumables) through a migration script.  
   - Why: Establishes a canonical conversion model so every consumption event normalizes to base units, enabling consistent COGS and allocation math.

2. [x] **Create Item & Portion Catalog Layer**
   - Task: Add the `items` collection with purchase metadata, storage hints, supplier links, and default stocking outlets. Introduce menu-portion definitions that reference items/units so menu entries always carry base portions. Build repositories and services for item + portion lifecycle and expose RBAC-protected APIs.
   - Why: Separates what we buy from what we sell while making menu offerings portion-aware, unlocking recipe-driven consumption and service department tracking.

3. [x] **Implement Lots & Ledger Infrastructure**
   - Task: Define `stockLots` and `stockLedger` schemas with FIFO-friendly indexes. Build an inventory domain service that records all movements (purchase, transfer, consumption, adjustment) as append-only entries and resolves consumption using item/portion metadata.  
   - Why: Enables cost tracing per lot, ensures auditability, and powers allocation vs actual reporting while respecting the new menu-portion linkage.

4. [x] **Migrate Legacy Stock Data**
   - Task: Write migration scripts to convert `Product.stock` blobs into items, opening stock lots, and balancing ledger entries. Capture fallbacks for missing cost data (e.g., use product price or zero).  
   - Why: Preserves historical quantities while transitioning to the new model, preventing disruption to live outlets.

5. [x] **Introduce Recipe & Portion Module**
   - Task: Add `recipes` with versioning, portion/serving definitions, waste adjustments, and pricing strategies (linear/stepped). Implement a resolver service that hydrates menu items with active portion data, cached in Redis to enforce effective date ranges.  
   - Note: Recipes now persist alongside menus (`Recipe.model.ts`) with automatic versioning and Redis-backed resolution; menu creation seeds an active recipe and controllers pull hydrated portions via the resolver cache.
   - Why: Connects menu products to inventory items and their portion-unit specs, ensuring accurate deduction per portion/shot and future cost analysis.

6. [x] **Refactor Product and Menu Services**
   - Task: Remove direct stock mutations from product/menu services; instead, integrate the inventory service for availability snapshots. Extend DTOs to include associated recipes, portion-unit metadata, and default/max portions per serving.  
   - Note: Menus now seed recipes through `RecipeService` and products persist portion → item links while delegating stock adjustments to the lot/ledger service.  
   - Why: Keeps product catalog focused on sellable metadata while ensuring every menu entry carries the portion-to-item linkage required for automated deductions.

7. [x] **Rebuild Order Pay Pipeline**
   - Task: Wrap `OrderService.pay` in MongoDB transactions. For each order line, resolve recipe, compute portions, convert units, consume FIFO lots, and persist `inventoryDeductionId` plus cost snapshots. Add idempotency checks.  
   - Why: Ensures stock is deducted atomically with payments, prevents double-counting, and captures COGS accurately.

8. [x] **Upgrade Allocation Reserve/Confirm/Release Flows**
   - Task: Redesign allocations to reference items and policies, and drive all internal usage through the existing reserve → confirm → release APIs (no separate requisition/issue models). Ensure ledger entries, policy enforcement, and soft-cap warnings surface correctly.  
   - Why: Closes the loop for non-sales departments while keeping the workflow lightweight and consistent with the backend capabilities.

9. [x] **Deliver Reporting & Monitoring**
   - Task: Build aggregation pipelines for sales vs COGS, allocation variance, valuation, and lot aging. Persist daily summaries, cache hot paths in Redis, and expose reporting APIs.  
   - Why: Provides finance and ops teams with actionable insights and keeps queries performant under load.

10. [x] **Hardening & Release Readiness**
    - Task: Expand automated tests (unit + integration) to cover new services, add data-quality scripts (negative stock, orphan recipes), update CI pipelines, and document rollback procedures.  
    - Why: Reduces deployment risk, ensures ongoing data health, and prepares the team for production rollout.

11. [x] **Auto-Priced Menus & Component Pricing**
   - Task: Introduce `menuPricing` on items (array of `{ unitId, price, label? }`) so each item+unit combination carries its menu price. Provide admin UI/endpoints to manage these price definitions.
   - Task: Adjust menu creation/update to compute `autoPrice` from portion components, sum component price × quantity × default portions, and treat manual menu price as an optional adjustment (surcharge/discount).
   - Task: Persist `autoPrice`, `manualAdjustment`, and `finalPrice` on menus/recipes; ensure order payments recalc from the stored recipe snapshot to avoid tampering.
   - Task: Update menu/order DTOs and frontend surfaces to show base/component pricing, and extend pricing reports to accommodate both auto and manual components.
   - Task: Enforce that every menu/product API (single, multi-branch, CSV) supplies valid portion payloads or `portionPresetId` references and reject missing `menuPricing` combinations with a 422 response. Refresh `saletrack.spec.md`, flow docs, usage guides, and the Postman collection so downstream teams understand the new validation rules.
   - Why: Guarantees menu prices stay in sync with ingredient pricing, reduces manual errors, and improves transparency between inventory cost and menu revenue.

---

## Admin Frontend Workstream

1. [x] **Re-align Models and Types**
   - Task: Update TypeScript interfaces/DTOs to match new backend contracts (units, items, recipes, lots, allocations). Regenerate API clients if applicable.  
   - Why: Keeps the admin app type-safe and prevents runtime failures once backend responses change.

2. [x] **Build Unit & Conversion Manager**
   - Task: Create UI for defining item families, base units, and conversion factors with validation and conflict warnings. Include list, detail, and audit history views, plus portion-unit presets consumable by menu configuration.  
   - Why: Allows inventory administrators to maintain the unit system required for accurate stock deductions and gives menu teams reusable portion/unit building blocks.

3. [x] **Introduce Item & Portion Catalog Experience**
   - Task: Add pages to list and edit items, capture purchase metadata, supplier details, default units, and outlet availability. Include visibility into associated lots, ledger snapshots, and linkages to portion presets.  
   - Task: Surface a dedicated portion preset manager (list, filter by family, add/edit components, stepped pricing) and allow items/menus to hydrate portions from a selected preset.  
   - Why: Gives procurement and inventory teams the tools to manage non-sellable stock entities while enabling chefs/bartenders to map menu portions directly to inventory items with consistent, reusable templates.

4. [x] **Recipe & Portion Builder Interface**
   - Task: Implement a recipe editor with portion configuration (default/max portions, stepped pricing), ingredient selection (items), unit conversion previews, and cost-per-portion calculations. Support version history and activation toggles.  
   - Why: Enables chefs and bar managers to translate menu offerings into precise consumption rules tied to the new portion-unit model.

5. [x] **Stock Lot & Ledger Dashboards**
   - Task: Surface lot-level stock status, including remaining quantity, unit cost, expiry, and movement history. Provide filters by outlet, item family, and policy violations (negative stock).  
   - Why: Equips operations with real-time visibility into inventory health and supports investigation of discrepancies.

6. [ ] **Enhanced Allocation Reserve/Confirm/Release Workflows**
   - Task: Redesign allocation forms to use items/units, display policy settings, show plan vs actual charts, and expose the reserve → confirm → release timeline (no separate requisition/issue layer).  
   - Why: Keeps non-sales departments in lockstep with the backend reserve/confirm/release model and ensures policy warnings surface in the UI.

7. [ ] **Order & Menu Insights**
   - Task: Update order detail views to display recipe snapshots, deduction cost breakdowns, and idempotency markers. Enhance menu pages with linked recipes, portion-unit definitions, and availability indicators derived from live stock.  
   - Why: Provides front-of-house supervisors with transparency into how orders impact inventory, validates portion selections, and highlights configuration issues early.

8. [ ] **Reporting Dashboards**
   - Task: Build visualizations for sales vs COGS, allocation variance, valuation, and lot aging using backend reporting APIs. Implement drill-down to orders, lots, and departments.  
   - Why: Turns the new data into actionable insights for management and finance, completing the inventory-to-revenue loop.

9. [ ] **Testing, QA, and Adoption Prep**
   - Task: Expand frontend unit/integration tests, add smoke scripts for critical flows, update documentation/help guides, and prepare training materials for outlet managers.  
   - Why: Ensures the revamped UI is stable, well-understood, and ready for organization-wide adoption alongside the backend release.

10. [ ] **Auto-Priced Menus UI & Pricing Management**
    - Task: Extend admin screens/forms so ops can manage `menuPricing` entries per item/unit, view auto-calculated menu prices, and optionally add manual adjustments or surcharges. Surface warnings for missing component prices.  
    - Task: Update menu builder to show component price breakdown, compute preview totals (auto price + adjustment), and persist final price metadata to match backend expectations.  
    - Task: Adjust POS/order-facing components to display final menu price sourced from the new fields while preserving existing UX.  
    - Why: Keeps the admin experience consistent with the backend auto-pricing rework and gives outlets visibility into how menu prices are derived.

---

Coordinate backend and frontend sprint sequencing so shared contracts (items, recipes, allocations) are feature-flagged or released in compatible slices. Use the flow in `docs/flows/saletrack-flow.md` as the execution playbook once the plan above is approved.
