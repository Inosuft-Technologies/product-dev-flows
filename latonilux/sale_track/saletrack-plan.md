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

6. [ ] **Refactor Product and Menu Services**
   - Task: Remove direct stock mutations from product/menu services; instead, integrate the inventory service for availability snapshots. Extend DTOs to include associated recipes, portion-unit metadata, and default/max portions per serving.  
   - Note: Menus now seed recipes through `RecipeService` and products persist portion → item links while delegating stock adjustments to the lot/ledger service.  
   - Why: Keeps product catalog focused on sellable metadata while ensuring every menu entry carries the portion-to-item linkage required for automated deductions.

7. [ ] **Rebuild Order Pay Pipeline**
   - Task: Wrap `OrderService.pay` in MongoDB transactions. For each order line, resolve recipe, compute portions, convert units, consume FIFO lots, and persist `inventoryDeductionId` plus cost snapshots. Add idempotency checks.  
   - Why: Ensures stock is deducted atomically with payments, prevents double-counting, and captures COGS accurately.

8. [ ] **Upgrade Allocation & Internal Issue Flows**
   - Task: Redesign allocations to reference items and policies. Wire requisition/issue endpoints to the ledger (transfer out/in, internal use) and enforce soft/hard caps with audit logs.  
   - Why: Closes the loop for non-sales departments and aligns inventory planning with actual consumption.

9. [ ] **Deliver Reporting & Monitoring**
   - Task: Build aggregation pipelines for sales vs COGS, allocation variance, valuation, and lot aging. Persist daily summaries, cache hot paths in Redis, and expose reporting APIs.  
   - Why: Provides finance and ops teams with actionable insights and keeps queries performant under load.

10. [ ] **Hardening & Release Readiness**
    - Task: Expand automated tests (unit + integration) to cover new services, add data-quality scripts (negative stock, orphan recipes), update CI pipelines, and document rollback procedures.  
    - Why: Reduces deployment risk, ensures ongoing data health, and prepares the team for production rollout.

---

## Admin Frontend Workstream

1. [ ] **Re-align Models and Types**
   - Task: Update TypeScript interfaces/DTOs to match new backend contracts (units, items, recipes, lots, allocations). Regenerate API clients if applicable.  
   - Why: Keeps the admin app type-safe and prevents runtime failures once backend responses change.

2. [ ] **Build Unit & Conversion Manager**
   - Task: Create UI for defining item families, base units, and conversion factors with validation and conflict warnings. Include list, detail, and audit history views, plus portion-unit presets consumable by menu configuration.  
   - Why: Allows inventory administrators to maintain the unit system required for accurate stock deductions and gives menu teams reusable portion/unit building blocks.

3. [ ] **Introduce Item & Portion Catalog Experience**
   - Task: Add pages to list and edit items, capture purchase metadata, supplier details, default units, and outlet availability. Provide UI to define default portions per menu family. Include visibility into associated lots and ledger snapshots.  
   - Why: Gives procurement and inventory teams the tools to manage non-sellable stock entities while enabling chefs/bartenders to map menu portions directly to inventory items.

4. [ ] **Recipe & Portion Builder Interface**
   - Task: Implement a recipe editor with portion configuration (default/max portions, stepped pricing), ingredient selection (items), unit conversion previews, and cost-per-portion calculations. Support version history and activation toggles.  
   - Why: Enables chefs and bar managers to translate menu offerings into precise consumption rules tied to the new portion-unit model.

5. [ ] **Stock Lot & Ledger Dashboards**
   - Task: Surface lot-level stock status, including remaining quantity, unit cost, expiry, and movement history. Provide filters by outlet, item family, and policy violations (negative stock).  
   - Why: Equips operations with real-time visibility into inventory health and supports investigation of discrepancies.

6. [ ] **Enhanced Allocation & Requisition Workflows**
   - Task: Redesign allocation forms to use items/units, display policy settings, and show plan vs actual charts. Extend requisition and issue tracking to align with new ledger events.  
   - Why: Keeps non-sales departments in lockstep with the redesigned backend flows, enabling cost-center accountability.

7. [ ] **Order & Menu Insights**
   - Task: Update order detail views to display recipe snapshots, deduction cost breakdowns, and idempotency markers. Enhance menu pages with linked recipes, portion-unit definitions, and availability indicators derived from live stock.  
   - Why: Provides front-of-house supervisors with transparency into how orders impact inventory, validates portion selections, and highlights configuration issues early.

8. [ ] **Reporting Dashboards**
   - Task: Build visualizations for sales vs COGS, allocation variance, valuation, and lot aging using backend reporting APIs. Implement drill-down to orders, lots, and departments.  
   - Why: Turns the new data into actionable insights for management and finance, completing the inventory-to-revenue loop.

9. [ ] **Testing, QA, and Adoption Prep**
   - Task: Expand frontend unit/integration tests, add smoke scripts for critical flows, update documentation/help guides, and prepare training materials for outlet managers.  
   - Why: Ensures the revamped UI is stable, well-understood, and ready for organization-wide adoption alongside the backend release.

---

Coordinate backend and frontend sprint sequencing so shared contracts (items, recipes, allocations) are feature-flagged or released in compatible slices. Use the flow in `docs/flows/saletrack-flow.md` as the execution playbook once the plan above is approved.
