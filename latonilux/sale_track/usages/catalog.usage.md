# Catalog API (Menus & Products)

This guide documents the menu and product endpoints enhanced during SaleTrack Steps 2 and 6. Both surfaces now embed recipe/portion data so menu items and sellable products are tied to inventory items and units.

## Common

- Base route: `/v1/menus` and `/v1/products`.
- Authentication required. Create/update/delete operations are limited to `superadmin` and `admin` roles.
- Portion definitions must reference valid `itemId` and `unitId` pairs from the inventory module.
- All create/update endpoints return the hydrated document with active recipe details populated.

---

## Menu Endpoints

Menus represent the sellable offerings shown to guests. Each menu entry owns a recipe/portion definition that tells the inventory subsystem which items and units to deduct. Whenever you create or modify a menu, the recipe cache (`RecipeService` and Redis) is refreshed so order fulfilment stays accurate.
- `GET /v1/menus`
  - Paginated list; supports query params from the advanced results middleware (`page`, `limit`, `sort`, etc.).
- `GET /v1/menus/all`
  - Public listing used by consumer apps (no auth required but still channel validation).
- `GET /v1/menus/:id`
- `POST /v1/menus`
  - Body summary:
    ```json
    {
      "name": "House Mojito",
      "price": 4800,
      "code": "BAR-001",
      "description": "Rum, mint, lime",
      "branchID": "BRANCH-LAG",
      "type": "beverage",
      "unit": "glass",
      "serving": "bar",
      "branches": [{ "branchID": "BRANCH-LAG", "code": "BAR" }],
      "portions": [
        {
          "portionPresetId": "PRESET-RUM-SHOT",
          "itemId": "ITEM-RUM",
          "unitId": "UNIT-ML",
          "label": "Single",
          "code": "SGL",
          "defaultPortions": 1,
          "maxPortions": 2,
          "stepSize": 1,
          "pricingMode": "linear",
          "wasteAllowancePct": 3,
          "isDefault": true,
          "components": [
            { "itemId": "ITEM-RUM", "unitId": "UNIT-ML", "quantity": 50 },
            { "itemId": "ITEM-MINT", "unitId": "UNIT-GRAM", "quantity": 5 }
          ]
        }
      ]
    }
    ```
  - `price` is optional. When present it is treated as a manual adjustment added on top of the auto-calculated ingredient cost.
- `POST /v1/menus/bulk`
  - Accepts CSV-derived payload. `portions` field can be JSON string per row.
- Each portion entry must either include a `portionPresetId` (to auto-fill the rest) or explicitly define `itemId`, `unitId`, and `components`. Missing ingredient price configuration on the linked item raises a `422 item menu pricing not configured...` error.
- `POST /v1/menus/search`
  - Body: `{ key: string }`. Searches by `name`, `menuID`, etc.
- `POST /v1/menus/filter`
  - Body matches `FilterMenuDTO` (currently only `type`).
- `PUT /v1/menus/:id`
  - Supports updating price, description, portions, enabled status.
- `PUT /v1/menus/detach/:id`
  - Body: `{ branchID: string }`. Removes menu from a branch without deleting.
- `DELETE /v1/menus/:id`
  - Marks the menu as archived and purges branch attachments.

### Portion & Recipe Behaviour

- When a menu is created, the portion payload is normalised and saved as a versioned recipe. Orders pull this recipe via `RecipeService.getActiveRecipe` to determine ingredient usage. If a referenced item or unit is missing, the controller rejects the request with a descriptive error.
- Menu creation automatically seeds a `Recipe` document with the same portion data.
- Every update revalidates components and keeps the active recipe in sync. Invalid inventory references result in 400 errors.
- Menu responses include `autoPrice`, `manualAdjustment`, and `finalPrice`. Portion components echo `unitPrice`, `priceLabel`, and `currency` so UIs can display how the price was built.
- All create/update/bulk paths validate that each `{ itemId, unitId }` pair has a configured menu price on the item. Missing prices trigger a `422 menu pricing not configured for: ...` error and halt the request.
- Bulk (`/v1/menus/bulk`) and multi-branch (`/v1/menus/create-multiple`) operations reuse the same normalisation, so a single erroneous row cancels the batch and surfaces the offending menu.

---

## Product Endpoints

Products cover retail items or packaged goods that can be sold alongside menus. They share the same portion schema so that stock reservations, confirmations, and releases behave the same way for both menus and products. Product definitions link to `ProductService`, which coordinates with `StockLotService`.
- `GET /v1/products`
  - Supports filtering by `unit`, `status`, `type` (see `FilterProductDTO`).
- `GET /v1/products/:id`
- `POST /v1/products`
  - Body summary:
    ```json
    {
      "name": "Bottled Water",
      "price": 800,
      "description": "500ml PET",
      "branchID": "BRANCH-LAG",
      "categoryCode": "BEV",
      "quantity": 120,
      "unit": "bottle",
      "portions": [
        {
          "portionPresetId": "PRESET-WATER-BTL",
          "itemId": "ITEM-WATER",
          "unitId": "UNIT-ML",
          "defaultPortions": 1,
          "maxPortions": 1,
          "components": [
            { "itemId": "ITEM-WATER", "unitId": "UNIT-ML", "quantity": 500 }
          ]
        }
      ],
      "stock": [
        { "branchID": "BRANCH-LAG", "categoryCode": "BEV", "code": "BOTTLE-500", "quantity": 120 }
      ]
    }
    ```
  - Stock entries create initial lots per branch/category using the new inventory service.
- `POST /v1/products/bulk`
  - Accepts CSV rows; each row must provide a `portions` column containing a JSON array (same schema as the `portions` body field). Rows without valid portions are rejected so recipes stay connected to inventory.
- `POST /v1/products/search`
  - Body: `{ key: string }` to search across name/SKU.
- `POST /v1/products/filter`
  - Body uses `FilterProductDTO` fields.
- `PUT /v1/products/:id`
  - Allows price/unit/description changes and stock adjustments via `{ stock: { action: 'add'|'remove'|'restore', quantity } }`.
- `PUT /v1/products/detach/:id`
  - Body: `{ branchID: string }`. Removes product from an outlet (releases reservations as needed).
- `DELETE /v1/products/:id`
  - Archives the product.

### Portion Behaviour

- Behind the scenes `ProductService` converts the product portions into a recipe snapshot. This allows `ProductService.reserveStock` and `ProductService.confirmStock` to map orders to individual stock lots.
- Products share the menu portion schema. During creation/update the service normalises portions into recipes so the order pipeline can deduct inventory using `ProductService.confirmStock`.
- All create/update/bulk paths require at least one portion definition (direct JSON or via `portionPresetId`). If a referenced item's menu pricing does not include the specified unit, the API returns `422 item menu pricing not configured for: ...` so admins can update the item before retrying.
