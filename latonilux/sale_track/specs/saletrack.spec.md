# SaleTrack Specification
**Version:** 1.2
**Date:** 2025-11-03
**Owner:** LATONILUX (Hospitality Platform)
**Stack:** Node.js, TypeScript, ExpressJS, MongoDB, Redis

---

# Context

LATONILUX is a monolithic hospitality platform (hotel management system) with RBAC and modules including membership subscriptions, inventory, orders, events, menu, staff/roles. The system must close the loop from **inventory → usage/consumption → sales → revenue/COGS** for both Kitchen and Bar, while also handling **non-sales departments** (Housekeeping, Laundry, Maintenance, etc.) that consume inventory without selling.

Operational nuances:

* **Food (Kitchen)**: *Rice* is purchased in **bags**, converted to **congos**, then to **plates → portions**. Example: *1 bag → 30 congos*, *1 congo → 6 plates*, *1 plate → 1 portion*. A single **plate can hold multiple portions**.
* **Bar (Liquids)**: Some products are sold as **full bottles** (e.g., Hennessy bottle at ₦100,000), others by **shots** (e.g., Tequila *per shot*). Liquids are tracked in **ml**; e.g., *1 bottle → 700 ml*, *1 shot → 30 ml*.
* **Service Departments**: Do not sell but **consume** supplies (e.g., detergents, toiletries). Their usage must be recorded for cost reporting by department.
* **Allocations**: Department managers plan/allocate items for a period (e.g., “Kitchen allocated 10 bags of rice this week”). Allocations may be informational caps or drive auto-transfers from central Store but do **not** directly determine sales.

---

# Goals

1. Provide a consistent model for **Units & Conversions** across item families (dry goods, liquids, countables).
2. Separate **Items** (what we buy & store) from **Products** (what we sell) and connect them via **Recipes/BOMs** (consumption rules per portion/shot/bottle).
3. **Menu items must be directly tied to portions and units** so that orders automatically carry the correct portion–unit relationship and connect to inventory.
4. Support **multi-portion plates** with linear or stepped pricing and correct consumption math.
5. Track inventory using **Lots** and an append-only **Stock Ledger** with FIFO consumption and **unit cost snapshots** for accurate **COGS**.
6. Enable **non-sales departments** to request, receive, and consume items (no revenue) for cost-center reporting.
7. Implement **Allocations** per department/period with optional caps and optional auto-transfer from Store.
8. Provide **Order → Payment** flows that deduct stock idempotently and atomically, respecting recipes and portions.
9. Deliver **Sales, Usage, Allocation-vs-Actual, and Valuation** reports with drill-down to lots and orders.
10. Enforce **RBAC**, validations, and audit trails across critical operations.
11. Cache hot read paths (recipes, conversions, menus, allocations) with Redis.
12. Provide **auto-calculated menu pricing** driven by ingredient unit prices, with optional manual adjustments for surcharges or discounts.

---

# Key Definitions

* **Item**: Stock-keeping entity we purchase and store (e.g., Rice 50kg, Tequila 70cl).
* **Product (Menu Item)**: Sellable entry **tied to a measurable base** (portion, shot, or bottle). Each product must be linked to its **portion–unit definition** and its **recipe**.
* **Recipe (BOM)**: Consumption rules linking a Product to Items and units (defined per portion/shot/bottle).
* **Unit/Conversion**: Admin-defined units per item family with factors to a base unit (e.g., congo for grains, ml for spirits).
* **Stock Lot**: A batch of purchased goods with cost and quantity in base units.
* **Stock Ledger**: Append-only record of all stock movements (Purchase, Transfer, Consumption, Spoilage, Adjustment, Return).
* **Outlet**: Physical sales/stock location (Kitchen, Bar, Store).
* **Department**: Organizational group. Kitchen/Bar = **SalesOutlet**; Housekeeping/Laundry/Maintenance = **Service**.
* **Allocation**: Planned quantities per department/period.
* **Order**: Customer order with product, quantity, and optional portion count.

---

# Integration of Menu, Portions, and Units

Every menu item created in the system must carry a direct connection to its measurable stock base. This includes:

* **Portion/Serving definition** (e.g., 1 portion = 1/6 congo; 1 shot = 30 ml).
* **Linked recipe** that defines consumption per serving.
* **Default & maximum portions** per serving.

When a customer places an order from the menu, the system automatically references this configuration. Each order line retrieves its portion–unit mapping, consumption rates, and pricing from the menu setup. This guarantees that **sales are always connected to measurable inventory consumption** without manual intervention.

**Relationship chain:**

```
Menu Item → Portion → Unit → Recipe → Stock → Order → Sale
```

---

# Menu Component Pricing

Menus inherit their base selling price from ingredient prices defined at the item+unit level. Each item may expose one or more `menuPricing` entries (e.g., rice portion = ₦450, chicken piece = ₦1,500). When an admin builds or edits a menu:

1. The service resolves every portion component `{ itemId, unitId, quantity }` to its `menuPricing` entry. Missing entries raise warnings or block save (configurable) so pricing remains transparent.
2. The portion **auto price** is the sum of `componentPrice × componentQuantity`, multiplied by the menu’s default portion count. Multiple portions (e.g., double serving) scale automatically.
3. Menus persist `{ autoPrice, manualAdjustment, finalPrice }`. Manual adjustment lets outlets add surcharges or discounts without losing the ingredient-derived base price.
4. Recipe snapshots store component prices so historical orders keep the original pricing even if ingredient prices change later. Order/payment flows recompute totals from the snapshot to prevent tampering.

All menu creation paths (single branch, multi-branch clone, CSV import) must include at least one portion definition. Each portion can either reference a `portionPresetId` or enumerate `itemId`, `unitId`, and `components`. If any referenced item lacks a `menuPricing` entry for the supplied unit, the API rejects the request with `422 item menu pricing not configured for: <item (unit)>`. This guarantees that auto-calculated prices always align with maintained ingredient rates.

Front-end tooling should display the computed auto price, the manual adjustment (if any), and highlight components missing unit pricing so admins can resolve them before publishing.

> **Portion presets** provide a reusable starting point for these component definitions. When the admin selects a preset, the UI hydrates the portion with the preset’s components, pricing mode, and default/max portions, drastically reducing repetitive data entry and ensuring consistency across menus/products that share the same serving profile.

---

# Data Model (Fields & Collections)

> MongoDB collection names are indicative; adjust to actual codebase.

## Units & Conversions

* **units**

  * `_id`, `name` (bag, congo, plate, portion, bottle, shot, ml, g, etc.)
  * `baseFor` (itemFamilyId)
  * `toBaseFactor` (number)
  * `type` (Mass | Volume | Count | Custom)
* **itemFamilies**

  * `_id`, `name` (grains-family, spirits-family)
  * `baseUnitId`

**Rules**

* Each Item belongs to an `itemFamily`; conversions must reference that family.
* Conversions form a DAG; back edges are disallowed.

## Items

* **items**

  * `_id`, `name`, `categoryId`, `itemFamilyId`, `baseUnitId`, `defaultPurchaseUnitId`
  * Optional: `volumeMl` (per bottle), `metadata` (JSON)
  * `menuPricing[]`: `{ unitId, price, label?, currency? }` — unit-linked prices used for automatic menu price calculation.

## Portion Presets

* **portionPresets**

  * `_id`, `name`, `description`, `familyId`
  * `defaultPortions`, `maxPortions`, `stepSize`
  * `pricingMode` (`linear` | `stepped`)
  * `components[]`: `{ itemId, unitId, quantity, wasteAllowancePct? }`
  * `stepPricing[]`: `{ portions, price }` (only when `pricingMode = stepped`)
  * `createdBy`, timestamps, `_version`

Portion presets are reusable templates that capture how a serving should consume inventory and how it should be priced before an actual menu or product is created. Each preset stores the canonical `itemId`, `unitId`, descriptive labels, default/max portions, step sizing, pricing mode, waste allowance, and the component list with `{ itemId, unitId, quantity, wasteAllowancePct }`. Menu and product builders can pick a preset to auto-populate portion definitions, enforce consistent ingredient usage, and keep menu auto-pricing aligned with ingredient costs. Updating a preset does not retroactively change existing menus/products; they inherit the values at creation time via their recipe snapshots. Even when a preset is used, the menu/product flow still validates that the referenced item’s `menuPricing` covers the preset’s units before allowing the save.

## Stock

* **stockLots**

  * `_id`, `itemId`, `outlet`, `qtyInBase`, `unitCost`, `supplier`, `receivedAt`, `expiryAt?`
* **stockLedger**

  * `_id`, `itemId`, `lotId?`, `outlet`, `type`, `qtyInBase` (pos=in, neg=out)
  * `reason?` (e.g., `Sale:orderId`, `InternalUse:issueId`, `Spoilage`)
  * `unitCostSnapshot?`, `createdAt`

## Departments & Outlets

* **departments**

  * `_id`, `name`, `type` (SalesOutlet | Service | Admin), `outlet?`, `costCenterCode?`
* **outlets**

  * Enumerated: Store, Kitchen, Bar (extendable)

## Products & Recipes

* **products**

  * `_id`, `name`, `productType` (Portion | Shot | Bottle | Combo)
  * `outlet` (Kitchen | Bar)
  * `price` (default base serving price) or `priceTable?` (per portion/shot map)
  * `portions[]` (required; identical to menu portions, accepts `portionPresetId` shortcuts)
  * `basePortionsPerServing?` (default=1), `maxPortionsPerServing?`
  * `portionUnitId?` (unit linked to serving/shot)
  * `portionSize?` (e.g., 1 portion = 1/6 congo)
  * `taxRate?`, `isActive`, `recipeId?`
  * `autoPrice?`, `manualAdjustment?`, `finalPrice?` — derived during menu build from recipe component pricing.
* **recipes**

  * `_id`, `productId`, `version`, `effectiveFrom`
  * `lines[]`: `{ itemId, unitId, qty, wastePct? }` **(qty defined per 1 portion/shot)**

## Orders

* **orders**

  * `_id`, `outlet`, `lines[]`
  * Line: `{ productId, qty, unitPrice, portionsPerServing? }`
  * `subtotal`, `tax`, `total`, `status` (Open|Paid|Voided|Refunded)
  * `createdAt`, `paidAt?`, `paymentRef?`, `inventoryDeductionId?`

## Internal Consumption (Allocations Reserve → Confirm → Release)

Allocations are the canonical way service departments reserve stock, confirm actual usage, and release unused quantities back into inventory. Instead of separate requisition/issue documents, the workflow hinges on `ProductService.reserveStock`, `confirmStock`, and `releaseStock` tied to allocation items.

* **allocations**

  * `_id`, `departmentId`, `periodStart`, `periodEnd`
  * `lines[]: { itemId, plannedQty, unitId, policy? (SoftCap|HardCap|None) }`
  * `status` (Planned|Committed|Closed), `notes?`, `createdBy`, `createdAt`
  * Policy metadata drives soft/hard cap warnings when reservations exceed projected usage.

* **reserve (create/update allocation)**

  * Each line reserves `plannedQty` via `ProductService.reserveStock`, emitting ledger entries of type `allocation-reserve`.
  * Soft-cap violations return warnings while hard-cap violations block the request.

* **confirm (`PUT /allocations/use/:id`)**

  * Confirms the quantity actually consumed (`items[].used += qty`, `items[].quantity -= qty`) and writes ledger entries of type `allocation-confirm`.

* **release (`PUT /allocations/release/:id`)**

  * Returns unused quantities to availability (`allocation-release` ledger entries) so stock becomes usable elsewhere.

## Allocations

* **allocations**

  * `_id`, `departmentId`, `periodStart`, `periodEnd`
  * `lines[]: { itemId, plannedQty, unitId, policy? (SoftCap|HardCap|None) }`
  * `status` (Planned|Committed|Closed), `notes?`, `createdBy`, `createdAt`

## Summaries (optional, precomputed)

* **salesSummaries** (daily/weekly)

  * `_id: { day, outlet }`, `revenue`, `cogs`, `grossMargin`, `topProducts[]`, `createdAt`

## Bulk Catalog Imports

CSV-driven menu or product onboarding must follow the same validation rules as the API. Every row needs a `portions` column that supplies a JSON array of portion definitions (or at least a `portionPresetId`). Rows missing this column, failing to parse the JSON, or referencing items without matching `menuPricing` entries are rejected with row-level errors. This ensures automated imports cannot bypass the inventory linkage or auto-pricing guarantees.

---

# Core Services

## Conversion Service

* `toBase(itemFamilyId, unitId, qty) -> qtyInBase`
* `fromBase(itemFamilyId, unitId, qtyInBase) -> qty`
* Cached per `itemFamilyId` in Redis; cache bust on units update.

## FIFO Lot Picker

* For a given `itemId` & `outlet`, pick lots ordered by `receivedAt` with remaining `qtyInBase > 0`.
* Split consumption across lots; write one ledger row per lot with `unitCostSnapshot = lot.unitCost`.

## Recipe Resolver

* Given `productId` and `asOf` timestamp, return the active recipe (by `effectiveFrom` and `version`).
* Cache resolved recipe (by product + version) in Redis.

## Pricing Resolver

* For `Portion`/`Shot`: use `priceTable[portionsPerServing]` if present, else `portionsPerServing × price`.
* Taxes/service charges applied to compute `unitPrice` stored on order lines.

---

# Workflows

## Admin: Configure Units & Conversions

1. Create `itemFamily` and set `baseUnitId` (e.g., congo).
2. Define units with `toBaseFactor` (e.g., bag→30 congos, congo→6 plates, plate→1 portion).
3. Validate DAG; warn on conflicting factors.

## Admin: Define Items

1. Create `Item` with `itemFamilyId`, `baseUnitId`, and `defaultPurchaseUnitId`.
2. For liquids, set `volumeMl` if relevant.

## Admin: Define Products & Recipes

1. Create `Product` with `productType`, `outlet`, `price/priceTable`.
2. For portionable items, set `basePortionsPerServing` and `maxPortionsPerServing`.
3. **Link product to portion–unit and recipe** (cannot activate product without this linkage).
4. Create `Recipe` **per portion/shot** (lines reference Item + unit). Set `effectiveFrom`.
5. Admin UI (menu/product “Recipes” tab) provides the recipe builder, version list, and activation controls so operations can update consumption rules without touching the API.

## Inventory: Purchase → Lot → Ledger

1. On GRN: compute `qtyInBase` via Conversion Service.
2. Create `stockLot` with `unitCost = totalCost / qtyInBase`.
3. Write `stockLedger` (type=Purchase, +qtyInBase, link lot).

## Inventory: Transfers

* Store → Kitchen/Bar/Dept: write `TransferOut` (Store) and `TransferIn` (target). Reuse lot IDs where possible.

## Service Departments: Reserve → Confirm → Release

Allocations implement the entire internal consumption flow. Instead of creating separate requisitions and issues, departments:

1. **Reserve** stock when the allocation is created or updated (`ProductService.reserveStock`, ledger type `allocation-reserve`). Policies (soft/hard caps) determine whether the reservation is allowed or only warned.
2. **Confirm** usage via `PUT /allocations/use/:id`, which deducts from the remaining quantity, increments `items[].used`, and writes ledger entries (`allocation-confirm`).
3. **Release** unused stock back into availability with `PUT /allocations/release/:id`, emitting `allocation-release` ledger entries.

This keeps the workflow lightweight while still preserving audit trails (ledger + allocation history) and policy enforcement.

## Orders: Create → Pay (Inventory Deduction)

1. Create Order with `lines` (each may set `portionsPerServing`, bounded by `maxPortionsPerServing`).
2. On `/pay` (transactional):

   * Resolve recipe version.
   * `effectivePortions = line.qty × (line.portionsPerServing || product.basePortionsPerServing)`.
   * For each recipe line: compute `requiredQtyPerPortion × effectivePortions`, convert to base, apply `wastePct`.
   * Run FIFO Lot Picker at `order.outlet`; write `Consumption` ledger rows with `unitCostSnapshot`.
   * Compute totals, set `inventoryDeductionId` (idempotency), set `status=Paid`, `paidAt`.

## Allocations: Reserve → Use → Close

* **Reserve**: Creating/updating an allocation reserves item quantities (and applies soft/hard cap validation).
* **Use**: `PUT /allocations/use/:id` confirms portions of the reserved quantity and records ledger consumption.
* **Release**: `PUT /allocations/release/:id` returns unused stock.
* **Close/Delete**: Optionally remove the allocation (`DELETE /allocations/:id`) once usage is reconciled; any remaining reservations are released automatically.

---

# API Endpoints (Indicative)

## Admin & Config

* `POST /units` — create/update units
* `GET /units?family=:id` — list units per family
* `POST /items` — create item
* `POST /products` — create/update product
* `POST /recipes` — create recipe version

## Stock

* `POST /stock/purchase` — { itemId, outlet, qty, unitId, totalCost, supplier }
* `POST /stock/transfer` — { itemId, fromOutlet, toOutlet, qty, unitId, lotId? }
* `POST /stock/adjust` — { itemId, outlet, qtyDelta, unitId, reason }

## Orders

* `POST /orders` — create (Open)
* `POST /orders/:id/pay` — finalize, deduct inventory, mark Paid
* `POST /orders/:id/void` — compensating ledger (reverse)

## Service Departments (Allocations)

* `POST /allocations/:id` — create/reserve stock for a department.
* `PUT /allocations/use/:id` — confirm consumption for a specific product code.
* `PUT /allocations/release/:id` — release unused reservations.
* `DELETE /allocations/:id` — cancel an allocation and free any remaining stock.

## Allocations

* `POST /allocations` — create period plan
* `POST /allocations/:id/commit` — apply policy (cap / transfer)
* `POST /allocations/:id/close`

## Reports

* `GET /reports/sales?from&to&outlet`
* `GET /reports/internal-usage?from&to&departmentId`
* `GET /reports/allocation-vs-actual?periodStart&periodEnd&departmentId`
* `GET /reports/stock/valuation?outlet`
* `GET /reports/usage?itemId&from&to`

---

# Business Rules & Validation

* Recipe must exist for `Portion`/`Shot` products before sale.
* `portionsPerServing` ∈ [1, maxPortionsPerServing]; default to `basePortionsPerServing`.
* Conversions must belong to the same `itemFamily` as the Item.
* No negative stock unless configuration `allowBackorder` is enabled; otherwise block payment.
* Lot unit costs are immutable; `unitCostSnapshot` is captured on consumption.
* Voids/refunds produce compensating ledger entries; if physical return is impossible, record as `Spoilage`.
* Allocations with **HardCap** block actions that would exceed projected usage; **SoftCap** requires override with audit trail.
* **Product activation requires** portion–unit linkage and an active recipe.

---

# Transactions, Idempotency, and Caching

* Wrap `/orders/:id/pay` in a Mongo transaction.
* Use an **idempotency key** (`inventoryDeductionId`) on Order to avoid double consumption.
* Cache in Redis: conversion maps, active recipe versions, product configs (incl. price tables), current allocations.

---

# Reporting Requirements

## Sales (Kitchen/Bar)

* By date range/outlet: units sold (servings), portions sold, revenue, discounts, taxes, **COGS**, **gross margin**.
* Product leaderboard with drill-down to orders and consumed lots.

## Internal Usage (Service Depts)

* Item usage by department/date range: qty and cost, top items, budget vs actual (future).

## Allocation vs Actual

* For each allocation line: planned vs actual (qty & cost), variance %, notes.

## Stock Valuation

* Per outlet: sum(remaining lot qty × lot unitCost) at a timestamp.

---

# Example Scenarios

## Rice (Portionable)

* Units: `bag → 30 congos`, `congo → 6 plates`, `plate → 1 portion` (base: congo).
* Product: **Rice Plate**; price table `{1: ₦700, 2: ₦1,300, 3: ₦1,800}`; `maxPortionsPerServing=3`.
* Order: 10 plates × 2 portions each → effective portions = 20.
* Recipe per portion consumes `1/6 congo` → total consumption = 3.333… congos (FIFO). Revenue = 10 × ₦1,300.

## Hennessy (Bottle)

* Product type: Bottle; sale consumes **1 bottle** from lot; price ₦100,000.

## Tequila (Shots)

* Units: `bottle → 700 ml`, `shot → 30 ml` (base: ml).
* Selling 26 shots consumes 780 ml; depletes one bottle (700 ml) + 80 ml from next lot.

## Housekeeping (Service Dept)

* Requisition 5L detergent. Issue recorded as **Consumption** (Option A) or **Transfer + later Consumption** (Option B). No revenue.

## Allocation

* Kitchen allocates 10 bags of rice for week N. On Commit with auto-transfer: Store → Kitchen transfer of 10 bags; sales flow unchanged, but consumption compares against plan.

---

# Open Questions & Assumptions

* Enforce **theoretical max shots per bottle** via `wastePct` or hard cap?
* Tax/Service charge rules per outlet vary; handled by Pricing Resolver.
* Negative stock policy default = **block** at payment; manager override optional.
* Price rounding rules to be confirmed.

---

# Phased Implementation Plan

1. **Foundations**: Units & Conversions, Items, Departments, Outlets.
2. **Inventory**: Lots + Ledger (Purchase, Transfer, Adjustment), FIFO & cost snapshots.
3. **Products & Recipes**: versioned recipes per portion/shot; pricing resolver.
4. **Orders**: create/pay/void with idempotent consumption.
5. **Service Depts**: allocation reserve/confirm/release workflow.
6. **Allocations**: plan/reserve/close with caps and policy enforcement.
7. **Reports**: sales, internal usage, allocation vs actual, valuation; nightly summaries.
8. **RBAC & Audits**: role gates and audit logs.
9. **Caching & Tuning**: Redis & aggregation performance.

---

# Acceptance Criteria (High Level)

* Multi-portion plate sale deducts inventory correctly and prices per policy.
* Shots deduct ml across lots; cost snapshots captured.
* Service department allocation usage (reserve/confirm/release) appears as ledger consumption with accurate costs.
* Allocations guide limits or create transfers; reports show variance.
* Sales reports: revenue, COGS, margins by product/outlet/date.
* Valuation: remaining lot qty × unit cost at any time.

---

# Appendix A — Mongoose Schema Stubs (TypeScript)

> Minimal stubs; add validation/hooks as needed.

```ts
// src/db/models/common.ts
import { Schema, model, Types } from 'mongoose';

export enum UnitType { Mass='Mass', Volume='Volume', Count='Count', Custom='Custom' }
export enum LedgerType { Purchase='Purchase', TransferIn='TransferIn', TransferOut='TransferOut',
  Production='Production', Consumption='Consumption', Spoilage='Spoilage', Adjustment='Adjustment', Return='Return' }
export enum ProductType { Portion='Portion', Shot='Shot', Bottle='Bottle', Combo='Combo' }
export enum DepartmentType { SalesOutlet='SalesOutlet', Service='Service', Admin='Admin' }
export enum Outlet { Store='Store', Kitchen='Kitchen', Bar='Bar' }
```

```ts
// src/db/models/unit.ts
const UnitSchema = new Schema({
  name: { type: String, required: true, unique: true },
  baseFor: { type: Types.ObjectId, ref: 'ItemFamily', required: true },
  toBaseFactor: { type: Number, required: true, min: 0 },
  type: { type: String, enum: Object.values(UnitType), required: true },
}, { timestamps: true });

UnitSchema.index({ baseFor: 1, name: 1 }, { unique: true });
export const Unit = model('Unit', UnitSchema);
```

```ts
// src/db/models/itemFamily.ts
const ItemFamilySchema = new Schema({
  name: { type: String, required: true, unique: true },
  baseUnitId: { type: Types.ObjectId, ref: 'Unit', required: true },
}, { timestamps: true });

export const ItemFamily = model('ItemFamily', ItemFamilySchema);
```

```ts
// src/db/models/item.ts
const ItemSchema = new Schema({
  name: { type: String, required: true },
  categoryId: { type: Types.ObjectId, ref: 'Category' },
  itemFamilyId: { type: Types.ObjectId, ref: 'ItemFamily', required: true },
  baseUnitId: { type: Types.ObjectId, ref: 'Unit', required: true },
  defaultPurchaseUnitId: { type: Types.ObjectId, ref: 'Unit', required: true },
  volumeMl: { type: Number },
  metadata: { type: Schema.Types.Mixed },
}, { timestamps: true });

ItemSchema.index({ name: 1, itemFamilyId: 1 }, { unique: true });
export const Item = model('Item', ItemSchema);
```

```ts
// src/db/models/stock.ts
const StockLotSchema = new Schema({
  itemId: { type: Types.ObjectId, ref: 'Item', required: true },
  outlet: { type: String, enum: Object.values(Outlet), required: true },
  qtyInBase: { type: Number, required: true, min: 0 },
  unitCost: { type: Number, required: true, min: 0 },
  supplier: String,
  receivedAt: { type: Date, required: true },
  expiryAt: Date,
}, { timestamps: true });

StockLotSchema.index({ itemId: 1, outlet: 1, receivedAt: 1 });

const StockLedgerSchema = new Schema({
  itemId: { type: Types.ObjectId, ref: 'Item', required: true },
  lotId: { type: Types.ObjectId, ref: 'StockLot' },
  outlet: { type: String, enum: Object.values(Outlet), required: true },
  type: { type: String, enum: Object.values(LedgerType), required: true },
  qtyInBase: { type: Number, required: true }, // pos=in, neg=out
  reason: String,
  unitCostSnapshot: Number,
  createdAt: { type: Date, default: () => new Date() },
}, { timestamps: false, versionKey: false });

StockLedgerSchema.index({ itemId: 1, createdAt: 1 });
StockLedgerSchema.index({ outlet: 1, createdAt: 1 });

export const StockLot = model('StockLot', StockLotSchema);
export const StockLedger = model('StockLedger', StockLedgerSchema);
```

```ts
// src/db/models/org.ts
const DepartmentSchema = new Schema({
  name: { type: String, required: true, unique: true },
  type: { type: String, enum: Object.values(DepartmentType), required: true },
  outlet: { type: String, enum: Object.values(Outlet) },
  costCenterCode: String,
}, { timestamps: true });

export const Department = model('Department', DepartmentSchema);
```

```ts
// src/db/models/product.ts
const ProductSchema = new Schema({
  name: { type: String, required: true },
  productType: { type: String, enum: Object.values(ProductType), required: true },
  outlet: { type: String, enum: Object.values(Outlet), required: true },
  price: { type: Number },            // optional if priceTable provided
  priceTable: { type: Map, of: Number }, // key: portions/shots → price
  basePortionsPerServing: { type: Number, default: 1, min: 1 },
  maxPortionsPerServing: { type: Number, default: 3, min: 1 },
  taxRate: { type: Number, default: 0 },
  isActive: { type: Boolean, default: true },
  recipeId: { type: Types.ObjectId, ref: 'Recipe' },
  portionUnitId: { type: Types.ObjectId, ref: 'Unit' },
  portionSize: { type: Number }, // e.g. 1/6 (congo per portion), or 30 (ml per shot)
}, { timestamps: true });

ProductSchema.index({ outlet: 1, isActive: 1 });
export const Product = model('Product', ProductSchema);
```

```ts
// src/db/models/recipe.ts
const RecipeLineSchema = new Schema({
  itemId: { type: Types.ObjectId, ref: 'Item', required: true },
  unitId: { type: Types.ObjectId, ref: 'Unit', required: true },
  qty: { type: Number, required: true, min: 0 }, // per 1 portion/shot
  wastePct: { type: Number, default: 0, min: 0 },
}, { _id: false });

const RecipeSchema = new Schema({
  productId: { type: Types.ObjectId, ref: 'Product', required: true },
  version: { type: Number, required: true },
  effectiveFrom: { type: Date, required: true },
  lines: { type: [RecipeLineSchema], default: [] },
  notes: String,
}, { timestamps: true });

RecipeSchema.index({ productId: 1, effectiveFrom: 1 }, { unique: true });
export const Recipe = model('Recipe', RecipeSchema);
```

```ts
// src/db/models/order.ts
const OrderLineSchema = new Schema({
  productId: { type: Types.ObjectId, ref: 'Product', required: true },
  qty: { type: Number, required: true, min: 1 },
  unitPrice: { type: Number, required: true, min: 0 },
  portionsPerServing: { type: Number, min: 1 }, // optional override
}, { _id: false });

const OrderSchema = new Schema({
  outlet: { type: String, enum: Object.values(Outlet), required: true },
  lines: { type: [OrderLineSchema], default: [] },
  subtotal: { type: Number, required: true, min: 0 },
  tax: { type: Number, required: true, min: 0 },
  total: { type: Number, required: true, min: 0 },
  status: { type: String, enum: ['Open','Paid','Voided','Refunded'], default: 'Open' },
  createdAt: { type: Date, default: () => new Date() },
  paidAt: Date,
  paymentRef: String,
  inventoryDeductionId: { type: String }, // idempotency
}, { timestamps: false });

OrderSchema.index({ status: 1, createdAt: 1 });
export const Order = model('Order', OrderSchema);
```

```ts
// src/db/models/allocation.ts
const AllocationLineSchema = new Schema({
  itemId: { type: Types.ObjectId, ref: 'Item', required: true },
  plannedQty: { type: Number, required: true, min: 0 },
  unitId: { type: Types.ObjectId, ref: 'Unit', required: true },
  policy: { type: String, enum: ['SoftCap','HardCap','None'], default: 'None' },
}, { _id: false });

const AllocationSchema = new Schema({
  departmentId: { type: Types.ObjectId, ref: 'Department', required: true },
  periodStart: { type: Date, required: true },
  periodEnd: { type: Date, required: true },
  lines: { type: [AllocationLineSchema], default: [] },
  status: { type: String, enum: ['Planned','Committed','Closed'], default: 'Planned' },
  notes: String,
  createdBy: { type: Types.ObjectId, ref: 'User', required: true },
  createdAt: { type: Date, default: () => new Date() },
});

AllocationSchema.index({ departmentId: 1, periodStart: 1, periodEnd: 1 }, { unique: true });
export const Allocation = model('Allocation', AllocationSchema);
```

```ts
// src/db/models/summary.ts
const SalesSummarySchema = new Schema({
  day: { type: String, required: true }, // YYYY-MM-DD
  outlet: { type: String, enum: Object.values(Outlet), required: true },
  revenue: { type: Number, default: 0 },
  cogs: { type: Number, default: 0 },
  grossMargin: { type: Number, default: 0 },
  topProducts: [{ productId: { type: Types.ObjectId, ref: 'Product' }, name: String, unitsSold: Number, revenue: Number }],
  createdAt: { type: Date, default: () => new Date() },
});

SalesSummarySchema.index({ day: 1, outlet: 1 }, { unique: true });
export const SalesSummary = model('SalesSummary', SalesSummarySchema);
```

---

# Appendix B — Pay-Order Transaction Sequence Diagram

```mermaid
sequenceDiagram
  autonumber
  actor U as Cashier/Waiter
  participant API as POST /orders/:id/pay
  participant SVC as OrderService
  participant RS as RecipeResolver
  participant CS as ConversionService
  participant FIFO as FifoLotPicker
  participant DB as MongoDB (session)

  U->>API: Pay order {orderId, paymentRef}
  API->>SVC: beginPay(orderId, paymentRef)
  activate SVC
  SVC->>DB: startTransaction()
  SVC->>DB: load Order (status=Open)
  SVC->>RS: resolveActiveRecipe(productId, asOf=order.createdAt)
  RS-->>SVC: recipe(lines per portion/shot)
  loop for each order line
    SVC->>SVC: portions = qty × (portionsPerServing || basePortions)
    loop for each recipe line
      SVC->>CS: toBase(itemFamilyId, unitId, qty = recipe.qty × portions × (1+wastePct))
      CS-->>SVC: qtyInBase
      SVC->>FIFO: consume(itemId, outlet, qtyInBase)
      FIFO->>DB: fetch lots (itemId,outlet,qty>0 ORDER BY receivedAt)
      FIFO-->>SVC: plan [{lotId, qtyInBase, unitCost}...]
      SVC->>DB: insert stockLedger(-qtyInBase, type=Consumption, lotId, unitCostSnapshot)
      SVC->>DB: decrement stockLot(lotId).qtyInBase
    end
  end
  SVC->>DB: compute totals; set Paid + inventoryDeductionId
  SVC->>DB: commitTransaction()
  deactivate SVC
  DB-->>API: success
  API-->>U: 200 OK { order, totals, receipt }
```

**Failure Paths**

* Missing recipe → abort txn, 422.
* Insufficient stock (Hard policy) → abort txn, 409; (Soft) → allow with negative and audit.
* Idempotency: if `inventoryDeductionId` exists → short-circuit and return existing receipt.

---

# Notes

* Normalize and validate inputs (names/units). Use smallest currency unit (kobo) or Decimal128.
* Consider optimistic locking or per-order mutex to prevent double-pay.
