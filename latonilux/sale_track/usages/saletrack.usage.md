# SaleTrack End-to-End Usage Guide

This document walks admins through the full SaleTrack lifecycle: configuring inventory, building menus/products, stocking items, processing orders and allocations, and monitoring results through reports. Follow the steps in order to unlock the complete flow.

---

## 1. Configure Inventory Foundations

Everything in SaleTrack references the inventory module. Set these up first so menus/products and allocations have valid items and units to tether to.

1. **Create Item Families** (`POST /v1/inventory/families`)
   - Group similar ingredients (e.g., *Spirits*, *Dry Goods*, *Housekeeping Supplies*).
   - Families establish the default base unit and simplify reporting rollups.

2. **Define Units** (`POST /v1/inventory/units`)
   - Add measurable units for each family (litre, millilitre, kilogram, shot, etc.).
   - Flag the canonical base unit (`isBase = true`) and set conversion factors for derived units.

3. **Map Unit Conversions** (`POST /v1/inventory/conversions`)
   - Connect units inside a family (e.g., `1 bag = 25 kilogram`).
   - Without conversions the system cannot auto-convert recipe quantities during order fulfilment.

4. **Create Items** (`POST /v1/inventory/items`)
   - Capture purchase metadata (SKU, default purchase unit, supplier details, branch scope).
   - Items are the canonical identifiers every recipe, portion, and allocation will reference.

5. **Create Portion Presets** (`POST /v1/inventory/portion-presets`)
   - For each family, define reusable portion templates with default/max portions, component items/units, and optional stepped pricing. Menus and products can hydrate their portion definitions from these presets instead of re-entering the math every time.
   - Example payload:
     ```http
     POST /v1/inventory/portion-presets
     {
       "name": "Rice Plate",
       "familyId": "FAM-GRAINS",
       "defaultPortions": 2,
       "maxPortions": 4,
       "components": [
         { "itemId": "ITEM-RICE", "unitId": "UNIT-PORTION", "quantity": 1 }
       ]
     }
     ```
   - Revisit presets whenever pricing or waste assumptions change; downstream menus keep their own recipe snapshots so historical orders remain consistent.

6. **Configure Menu Pricing** (`PUT /v1/inventory/items/:id`)
   - Attach `menuPricing` entries to each item/unit combo you plan to sell. Auto-pricing requires these values.

✔️ **Checkpoint**: At this stage, `/v1/inventory/items` should list the ingredients/consumables you plan to sell or allocate, and `/v1/inventory/portion-presets` should return the reusable templates the catalog will reference.

---

## 2. Seed Stock Lots

Before orders can deduct inventory, load the initial on-hand balances.

1. **Create Stock Lots** (`POST /v1/inventory/lots`)
   - Provide item, branch, quantity, unit cost, and optional metadata (supplier, reference notes).
   - Each lot represents a FIFO bucket; the system uses these lots to compute cost of goods sold.

2. **Adjust Lots** (`PUT /v1/inventory/lots/adjust/:id`)
   - Use for correcting stock after counts. The ledger records adjustments for audit purposes.

3. **Consume Lots Manually (optional)** (`POST /v1/inventory/lots/consume`)
   - Handy for standalone tests or to deduct stock outside the order/allocation flows.

✔️ **Checkpoint**: `/v1/inventory/lots` should show open lots with positive `quantityRemaining` per branch.

---

## 3. Build the Catalog (Menus & Products)

With items and units in place, construct what guests can buy.

1. **Create Menus** (`POST /v1/menus` or `/v1/menus/bulk`)
   - Define department and the `portions` array. You can either pick an existing portion preset (recommended) or craft the portion manually. Prior to this, ensure every underlying inventory item has `menuPricing` entries for the units you plan to sell; otherwise menu creation fails with `menu pricing not configured for ...`.
   - Each portion references `itemId` + `unitId` pairs and optional components; the service creates a corresponding recipe version automatically and computes an `autoPrice` from the linked item prices.
   - The optional `price` field in the payload is treated as a manual adjustment (surcharge or discount). Responses expose `autoPrice`, `manualAdjustment`, and `finalPrice` along with the component-level `unitPrice`, `priceLabel`, and `currency` so you can surface the cost breakdown in the UI.

2. **Create Products** (`POST /v1/products`)
   - For retail SKUs, captured similarly to menus but can include initial stock entries.
   - Product portions generate recipes so product orders deduct ingredients through `ProductService`.

3. **Attach Menus/Products to Branches** (part of create/update payload or via `PUT /v1/.../detach/:id`)
   - Ensure each outlet has the correct offerings. Detach when an item should disappear from a location.

✔️ **Checkpoint**: Hitting `/v1/menus` and `/v1/products` should return entries with their `portions` array populated, `activeRecipe` ids set, and pricing fields (`autoPrice`, `manualAdjustment`, `finalPrice`) aligned with the ingredient pricing matrix.

---

## 4. Accept and Process Orders

Once the catalog is ready, orders can flow through the payment pipeline.

1. **Create Orders**
   - Menus: `POST /v1/orders/buy-menu/:memberId`
   - Products: `POST /v1/orders/buy-product/:memberId`
   - Events/Facilities: `POST /v1/orders/buy-event/:memberId`, `POST /v1/orders/buy-facility/:memberId`
   - The controller validates inventory and reserves stock (for product lines) ahead of payment.

2. **Initiate Payment** (`POST /v1/orders/pay/:id`)
   - `medium = digital`: creates a Paystack transaction via `OrderService.initOrderPayment`.
   - `medium = cash`: immediately records the transaction and marks the order paid.

3. **Handle Provider Webhooks** (`POST /v1/providers/paystack/webhook`)
   - On success, `OrderService.finalizeOrderPayment` confirms reserved lots, records consumption entries, and embeds inventory snapshots on the order.
   - On failure, the service releases reservations so stock is not locked.

4. **Fulfil & Complete**
   - `PUT /v1/orders/fulfill/:id` logs operational fulfilment, often used for multi-step workflows.
   - `PUT /v1/orders/complete/:id` moves the order to `completed` once delivery is verified.

5. **Cancel / Delete (Admin only)**
   - `DELETE /v1/orders/:id` releases reservations and removes the order. Use for aborted or test transactions.

✔️ **Checkpoint**: After a successful payment, the order document should contain `inventory.deductionId` and the `StockLedger` should show matching `consumption` entries.

---

## 5. Manage Allocations (Reserve → Confirm → Release)

Departments that consume stock outside of sales rely entirely on allocations.

1. **Create Allocation** (`POST /v1/allocations/:branchId`)
   - Reserves stock for non-sales activities and enforces soft/hard caps.

2. **Confirm Usage** (`PUT /v1/allocations/use/:id`)
   - Deducts a portion of the reservation. The system writes ledger entries with type `allocation-confirm`.

3. **Release Unused Stock** (`PUT /v1/allocations/release/:id`)
   - Returns unneeded quantities to availability (ledger type `allocation-release`).

4. **Close or Delete** (`DELETE /v1/allocations/:id`)
   - Cancels the allocation and releases any remaining reservations.

✔️ **Checkpoint**: Allocation records should show updated `items[].quantity` / `items[].used`, and the ledger should contain `reservation`, `confirm`, or `release` entries referencing the allocation code.

---

## 6. Monitor Through Reports

Leverage the reporting APIs to keep finance and operations informed.

- **Stock Summary** (`POST /v1/reports/stock/summary`): current stock health by status and branch.
- **COGS** (`POST /v1/reports/cogs`): consumption cost by day/item/branch.
- **Allocation Variance** (`POST /v1/reports/allocations/variance`): reserved vs confirmed vs released for allocations.
- **Inventory Valuation** (`POST /v1/reports/valuation`): value of open lots.
- **Lot Aging** (`POST /v1/reports/lots/aging`): bucketed age analysis for lots.

Each report supports `?format=csv` for export and relies on the ledger entries generated in earlier steps. Cache invalidation is handled automatically whenever inventory mutations occur.

---

## 7. Quick Troubleshooting Tips

- **Missing ingredients in orders**: ensure the menu/product portion references valid `itemId` and `unitId`, and that `RecipeService.prime` has run (automatic on create/update).
- **Insufficient stock errors**: check `/v1/inventory/lots` for the branch/item and confirm lots have `quantityRemaining > quantityReserved`.
- **Allocations stuck with zero usage**: verify the `use` endpoint returned 200; the ledger should show `allocation-confirm` entries.
- **Reports outdated**: if Redis cache persists old results, confirm that inventory mutations are completed and try forcing a re-run by triggering `ReportService.bumpGlobalVersion` (happens automatically on stock changes).

Following this lifecycle ensures SaleTrack’s modules stay in sync—from configuration, through fulfilment, to analytics.
