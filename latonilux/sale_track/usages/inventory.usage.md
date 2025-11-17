# Inventory API

Usage guide for the inventory building blocks introduced in SaleTrack (Steps 1 and 3):
item families, units, unit conversions, items, and stock lots/ledger flows.

## Common

- All routes are prefixed with `/v1/inventory` and require authentication.
- Unless stated otherwise, the allowed roles are `superadmin` and `admin`.
- `branchID` values accept either the branch's code or Mongo `_id`; the service resolves both.
- Timestamps accept ISO strings (e.g. `2024-11-03T12:30:00Z`).

---

## Item Families

- Item families group inventory items into logical categories (spirits, grains, housekeeping, etc.). They provide default base units and metadata that downstream modules (items, menus, recipes) inherit. Maintaining accurate families ensures conversions and reporting roll up correctly.
- `GET /v1/inventory/families`
  - Supports query params `page`, `limit`, `sort`, and `search` via the advanced results middleware.
  - Returns paginated list with populated base unit and creator metadata.
- `GET /v1/inventory/families/:id`
- `POST /v1/inventory/families`
  - Body: `{ name: string, description?: string, notes?: string, isEnabled?: boolean }`
- `PUT /v1/inventory/families/:id`
  - Body may update any create field plus `baseUnitId` to remap the family's base unit.

---

## Units

- Units define the measurable quantities available inside a family (litre, millilitre, shot, kilogram). Portions, recipes, and stock conversions rely on these definitions, so every menu/product portion links back to a valid unit here.
- `GET /v1/inventory/units`
- `GET /v1/inventory/units/:id`
- `POST /v1/inventory/units`
  - Body: `{ familyId, name, type, toBaseFactor?, isBase?, label?, symbol?, notes?, precision? }`
  - `type` is one of: `mass`, `volume`, `count`, `other` (matches enum in services).
  - If `isBase` is `true`, `toBaseFactor` defaults to `1`.
- `PUT /v1/inventory/units/:id`
  - Same schema as create, all fields optional.

---

## Unit Conversions

- Unit conversions connect two units within the same family (e.g., 1 bag = 25 kilograms). They power automatic conversions when recipes or allocations request quantities in a different unit than the stock lot. Missing or incorrect conversions will cause order fulfilment to fail validation.
- `GET /v1/inventory/conversions`
- `GET /v1/inventory/conversions/:id`
- `POST /v1/inventory/conversions`
  - Body: `{ familyId, fromUnitId, toUnitId, factor, isBidirectional?, notes? }`
  - `factor` expresses how many `toUnit` units equal one `fromUnit`.
- `PUT /v1/inventory/conversions/:id`
  - Body: `{ factor?, isBidirectional?, notes? }`

---

## Items

- Items are the canonical inventory records representing what procurement buys and what recipes consume. Items reference a family, base unit, and optional departments/outlets. Portions defined in menus/products must point at these items so `StockLotService` can deduct the correct ingredient.
- `GET /v1/inventory/items`
  - Accepts filters via query string (`familyId`, `status`, `branchId`, `departmentId`).
- `GET /v1/inventory/items/:id`
- `POST /v1/inventory/items`
  - Body summary:
    ```json
    {
      "name": "Granulated Sugar",
      "description": "25kg bag",
      "sku": "SUG-25KG",
      "familyId": "FAM-SWEETENERS",
      "baseUnitId": "UNIT-KG",
      "defaultPurchaseUnitId": "UNIT-BAG",
      "leadTimeDays": 5,
      "minReorderQty": 10,
      "tags": ["pantry"],
      "defaultOutletIds": ["branch-code"],
      "departmentIds": ["BAR"],
      "supplier": { "name": "Vendor", "email": "vendor@example.com" },
      "metadata": { "storage": "Dry", "notes": "Keep sealed" }
    }
    ```
- `PUT /v1/inventory/items/:id`
  - Same fields as create, all optional plus `status` (e.g. `active`, `retired`) and `isArchived` flag.
- `DELETE /v1/inventory/items/:id`
  - Archives the item via `ItemService.archiveItem` (soft delete).

## Portion Presets

- Portion presets are reusable templates that define how a serving should consume inventory and how it should be priced before a menu/product exists. Each preset stores the primary `itemId`, `unitId`, descriptive labels (`name`, `label`, `code`), default/max portions, step size, pricing mode (linear/stepped), waste allowance, and a component list with `{ itemId, unitId, quantity, wasteAllowancePct }` entries. Optional stepped pricing tables let you describe “buy 2 portions for ₦X” scenarios.
- Catalog builders can supply only the `portionPresetId` when creating menus or products; the service copies the preset’s fields, validates they still align with the item family, and ensures every component has `menuPricing` configured on the linked item. This keeps pricing and consumption consistent even across bulk uploads.
- `GET /v1/inventory/portion-presets`
  - Supports standard listing parameters (`page`, `limit`, `order`) plus `family` to filter presets tied to a specific item family. Results populate family, component items, and units.
- `GET /v1/inventory/portion-presets/:id`
- `POST /v1/inventory/portion-presets`
  - Body: `{ name, description?, familyId, itemId, unitId, label?, code?, isDefault?, defaultPortions, maxPortions?, stepSize?, pricingMode?, components: [{ itemId, unitId, quantity, wasteAllowancePct? }], stepPricing?: [{ portions, price }], wasteAllowancePct? }`
  - `pricingMode` defaults to `linear`. When set to `stepped`, provide step pricing rows to describe each tier.
- `PUT /v1/inventory/portion-presets/:id`
  - Accepts the same fields as create, all optional.
- `DELETE /v1/inventory/portion-presets/:id`
  - Removes the preset. Existing menus/products retain their own recipe snapshots; they are not retroactively changed.

---

## Stock Lots

- Stock lots are the FIFO buckets that hold on-hand quantity and cost for a specific item and branch. Every purchase, adjustment, and consumption event manipulates lots. Allocations, orders, and recipes interact with lots through `StockLotService`.
- `GET /v1/inventory/lots`
- `GET /v1/inventory/lots/:id`
- `POST /v1/inventory/lots`
  - Body: `{ itemId, branchId, quantity, unitCost?, currency?, receivedAt?, expiryAt?, supplierName?, reference?, notes? }`
  - Creates an open lot and writes a `purchase` ledger entry.
- `PUT /v1/inventory/lots/adjust/:id`
  - Body: `{ quantityDelta, notes? }`
  - Positive `quantityDelta` adds stock; negative subtracts (fails if result < 0).
- `POST /v1/inventory/lots/consume`
  - Body: `{ itemId, branchId, quantity, notes?, reference? }`
  - Performs FIFO consumption, records `consumption` ledger rows, and returns the consumption plan.

---

## Stock Ledger

- The stock ledger is the immutable journal of all inventory movements (purchase, reservation, confirmation, release, adjustment). Reporting, audit, and the COGS pipeline read from this collection. Use the ledger endpoints to investigate discrepancies or to feed external BI tools.
- `GET /v1/inventory/ledgers`
  - Advanced results with item, branch, and lot populated.
  - Use query params `type`, `item`, `lot`, `branch`, `start`, `end` to filter (standard listing middleware parameters).
