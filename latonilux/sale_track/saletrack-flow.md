# SaleTrack Rollout Flow

This runbook sequences the SaleTrack implementation from schema groundwork through admin rollout. Follow the steps in order; each depends on successful completion of the previous milestones.

> Prerequisites  
> - Local MongoDB seeded with the latest production snapshot (or sanitized copy) and `.env` from `latonilux-backend`.  
> - Node.js toolchain ready (`npm install` already executed in `latonilux-backend` and `latonilux-admin`).  
> - Redis instance running for cache validation.  
> - Access to current `saletrack.spec.md` and feature branch created for the initiative.

## 1. Establish Unit & Item Foundations

1. Add `itemFamilies`, `units`, and `unitConversions` models, repositories, and validation DTOs.  
   - Ensure each family defines a base unit and that conversions resolve to a DAG.  
   - Create admin seeding script to register core families (grains, proteins, spirits, consumables).
2. Introduce the `items` collection with purchase metadata (`defaultPurchaseUnitId`, `baseUnitId`, supplier refs, stocking outlets) and portion/serving definitions that map menu offerings to items + units.  
3. Expose secured CRUD endpoints for families, units, items, and menu portion presets (RBAC: inventory admins only).  
4. Verification:
   ```bash
   mongo --eval 'db.units.find({}, {name:1, baseFor:1, toBaseFactor:1}).pretty()'
   curl -H "x-access-token: <admin>" http://localhost:5000/api/identity/v1/units
   ```

## 2. Provision Lots & Ledger Infrastructure

1. Create `stockLots` and `stockLedger` schemas with indexes on `{ itemId, outlet, receivedAt }` and `{ itemId, outlet, createdAt }`.  
2. Implement FIFO picker and ledger writer services; wire them behind an inventory domain service.  
3. Migrate existing `product.stock` records into:  
   - `items` (one per product.sku) with inferred family/unit.  
   - Opening `stockLots` entries carrying current quantity and average cost (fallback: price or zero).  
   - Balancing `stockLedger` entries (type=`OpeningBalance`).  
4. Validation:
   ```bash
   mongo --eval 'db.stockLots.aggregate([{$group:{_id:"$itemId", qty:{$sum:"$qtyInBase"}}}]).pretty()'
   mongo --eval 'db.stockLedger.find({type:"OpeningBalance"}).limit(5).pretty()'
   ```

## 3. Split Product vs Item Responsibilities

1. Refactor `Product` model to drop embedded `stock` blob; keep sellable metadata (price, availability) while attaching portion definitions.  
2. Introduce `Recipe` (BOM) documents referencing `productId`, `itemId`, portionUnitId, waste %, effective dates, and pricing strategies.  
3. Update product/menu services to:  
   - Resolve current recipes and default/max portions via a `RecipeResolver`.  
   - Delegate stock adjustments to the inventory service (no direct `quantity` mutation).  
4. Adjust repositories, DTOs, and mappers to return recipes + portion metadata alongside products/menus for admin usage.  
5. Regression checks:
   ```bash
   npm run test -- product.service
   curl -H "x-access-token: <admin>" http://localhost:5000/api/identity/v1/menu/<id>?include=recipe
   ```

## 4. Rebuild Order Deduction Pipeline

1. Wrap `OrderService.pay` in Mongo transactions, injecting the new inventory service.  
2. For each order line:  
   - Resolve recipe + portion snapshot (as-of order creation).  
   - Compute `portions = quantity × selectedPortions` (falling back to menu default).  
   - Convert to base units via the conversion service.  
   - Consume FIFO lots, writing ledger entries (type=`Consumption`) with unit-cost snapshot.  
3. Record `inventoryDeductionId`, recipe snapshot, portion metadata, and cost breakdown on the order document.  
4. Add idempotency guard to short-circuit already-paid orders.  
5. Tests & verification:
   ```bash
   npm run test -- order.service
   mongo --eval 'db.orders.find({inventoryDeductionId:{$exists:true}}).limit(3).pretty()'
   mongo --eval 'db.stockLedger.find({type:"Consumption"}).sort({createdAt:-1}).limit(5).pretty()'
   ```

## 5. Upgrade Allocations & Internal Movements

1. Replace legacy `Allocation.items` with item references (`itemId`, `plannedQty`, `unitId`, `policy`).  
2. Introduce requisition/issue flows that create `stockLedger` movements (type=`TransferOut`, `TransferIn`, `InternalUse`).  
3. Enforce policy handling (hard cap blocks, soft cap warns and logs).  
4. Expose APIs for department managers to confirm receipts and log usage.  
5. Confirmations:
   ```bash
   mongo --eval 'db.allocations.find({}, {"lines.itemId":1, "lines.policy":1}).limit(5).pretty()'
   mongo --eval 'db.stockLedger.find({type:{$in:["TransferOut","InternalUse"]}}).limit(5).pretty()'
   ```

## 6. Reporting & Cache Enablement

1. Build aggregation pipelines for Sales vs COGS, Usage vs Allocation, Valuation, and Lot aging.  
2. Persist daily/weekly summaries to `salesSummaries` (Appendix B of spec).  
3. Add Redis caches for recipes, conversions, menu cards, and active allocations with invalidation hooks.  
4. Monitoring:
   ```bash
   curl -H "x-access-token: <admin>" http://localhost:5000/api/identity/v1/reports/sales?from=2025-01-01&to=2025-01-07
   redis-cli keys "saletrack:recipe:*"
   ```

## 7. Admin UI Integration

1. Update `latonilux-admin` DTOs/models to match new APIs (items, portions, recipes, lots, allocations).  
2. Build inventory console sections:  
   - Unit & conversion manager.  
   - Item catalog with lot visibility and portion presets.  
   - Recipe editor with portion preview, default/max portions, and cost per portion.  
   - Allocation dashboards showing plan vs actual.  
3. Ensure order, menu, and product views display portion choices, item consumption, and live stock status from the backend.  
4. Validate with frontend smoke tests (`npm run test` in admin project).

## 8. Regression, Data Quality, and Automation

1. Execute full backend test suite, lint, and build.  
   ```bash
   npm run lint && npm run test && npm run build
   ```
2. Run targeted data quality scripts: negative stock scan, orphan recipe detection, duplicate units.  
3. Configure CI pipelines to seed unit tests with sample lots and recipes to prevent regressions.  
4. Document rollback procedures (reverting migration, disabling deduction flow) in the ops wiki.

## 9. Cutover & Post-Go-Live Monitoring

1. Schedule downtime window; communicate to outlets and service departments.  
2. Deploy backend + admin in lockstep, run smoke script covering purchase, transfer, and service usage.  
3. Monitor Mongo `stockLedger` and Redis hit rates; enable alerts on negative stock and transaction failures.  
4. Conduct user training and gather feedback for backlog grooming.  
5. After stabilization (≈2 weeks), deprecate legacy stock endpoints and clean up unused fields.

---

Repeat validation for both Kitchen and Bar scenarios (multi-portion plates, bottle-to-shot conversions) and non-sales departments to ensure allocation and usage flows remain consistent end-to-end.
