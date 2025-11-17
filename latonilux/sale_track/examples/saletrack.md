# SaleTrack End-to-End Example – Pantry & Bar Purchase

This example shows how an administrator configures SaleTrack, stocks inventory, creates catalog entries, processes an order, and reads the resulting reports using a simple scenario:

- Buy **2 bags of rice** for the kitchen (each bag produces ~180 portions).
- Sell **10 bottles of Hennessy** (product).
- Sell **20 shots of Tequila** where one bottle contains 20 shots (menu item).
- Sell **4 plates of rice**, each containing **2 portions of rice** and **2 chicken laps** sold per piece.

All requests use JSON bodies unless stated otherwise. Replace IDs with your environment’s values.

> Portions (or a `portionPresetId`) are required on every menu/product call and in CSV imports. If an item does not expose `menuPricing` for the referenced unit, the API rejects the request with `422 item menu pricing not configured ...`. The examples below show both inline component definitions and preset-driven flows so you can mirror them in automation.

### CSV row sample (products)

```
name,price,description,branch_id,category_code,quantity,unit,portions
"Kitchen Rice Bag",0,"Internal issue reference","BRANCH-KITCHEN","KITCHEN",0,"bag","[{\"portionPresetId\":\"PRESET-RICE-BAG\"}]"
```

`portions` is a JSON array encoded inside the CSV cell; rows that omit or misformat this column are rejected during import.

---

## 1. Configure Inventory

### 1.1 Item Families

```http
POST /v1/inventory/families
{
  "name": "Grains",
  "description": "Staple grains and cereals"
}

POST /v1/inventory/families
{
  "name": "Spirits",
  "description": "Alcoholic beverages"
}

POST /v1/inventory/families
{
  "name": "Frozen Meat",
  "description": "Frozen proteins"
}
```

### 1.2 Units in each family

```http
POST /v1/inventory/units
{
  "familyId": "FAM-GRAINS",
  "name": "Kilogram",
  "type": "mass",
  "isBase": true
}

POST /v1/inventory/units
{
  "familyId": "FAM-GRAINS",
  "name": "Bag (25kg)",
  "type": "mass",
  "toBaseFactor": 25
}

POST /v1/inventory/units
{
  "familyId": "FAM-GRAINS",
  "name": "Portion",
  "type": "mass",
  "toBaseFactor": 0.1389,
  "notes": "One portion ≈ 0.1389kg (bag yields ~180 portions)"
}

POST /v1/inventory/units
{
  "familyId": "FAM-SPIRITS",
  "name": "Bottle (750ml)",
  "type": "volume",
  "isBase": true
}

POST /v1/inventory/units
{
  "familyId": "FAM-SPIRITS",
  "name": "Shot (37.5ml)",
  "type": "volume",
  "toBaseFactor": 0.05
}

POST /v1/inventory/units
{
  "familyId": "FAM-FROZEN",
  "name": "Piece",
  "type": "count",
  "isBase": true
}

POST /v1/inventory/units
{
  "familyId": "FAM-FROZEN",
  "name": "Pack (50 pieces)",
  "type": "count",
  "toBaseFactor": 50
}
```

### 1.3 Unit Conversions

```http
POST /v1/inventory/conversions
{
  "familyId": "FAM-SPIRITS",
  "fromUnitId": "UNIT-BOTTLE",
  "toUnitId": "UNIT-SHOT",
  "factor": 20,
  "isBidirectional": true
}

POST /v1/inventory/conversions
{
  "familyId": "FAM-FROZEN",
  "fromUnitId": "UNIT-PACK",
  "toUnitId": "UNIT-PIECE",
  "factor": 50,
  "isBidirectional": true
}

POST /v1/inventory/conversions
{
  "familyId": "FAM-GRAINS",
  "fromUnitId": "UNIT-BAG",
  "toUnitId": "UNIT-PORTION",
  "factor": 180,
  "isBidirectional": true,
  "notes": "Bag makes 180 portions"
}
```

### 1.4 Create Items

```http
POST /v1/inventory/items
{
  "name": "White Rice",
  "familyId": "FAM-GRAINS",
  "baseUnitId": "UNIT-KG",
  "defaultPurchaseUnitId": "UNIT-BAG",
  "minReorderQty": 5,
  "defaultOutletIds": ["BRANCH-MAIN"],
  "departmentIds": ["KITCHEN"],
  "metadata": { "storage": "Dry pantry" }
}

POST /v1/inventory/items
{
  "name": "Hennessy VS",
  "familyId": "FAM-SPIRITS",
  "baseUnitId": "UNIT-BOTTLE",
  "minReorderQty": 10,
  "defaultOutletIds": ["BRANCH-BAR"],
  "departmentIds": ["BAR"],
  "metadata": { "storage": "Bar shelf" }
}

POST /v1/inventory/items
{
  "name": "Tequila Blanco",
  "familyId": "FAM-SPIRITS",
  "baseUnitId": "UNIT-BOTTLE",
  "defaultPurchaseUnitId": "UNIT-BOTTLE",
  "defaultOutletIds": ["BRANCH-BAR"],
  "departmentIds": ["BAR"],
  "metadata": { "storage": "Bar shelf" }
}

POST /v1/inventory/items
{
  "name": "Chicken Lap",
  "familyId": "FAM-FROZEN",
  "baseUnitId": "UNIT-PIECE",
  "defaultPurchaseUnitId": "UNIT-PACK",
  "defaultOutletIds": ["BRANCH-KITCHEN"],
  "departmentIds": ["KITCHEN"],
  "metadata": { "storage": "Frozen" }
}
```

### 1.5 Portion Presets

Create reusable portion templates that the catalog will hydrate later.

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

POST /v1/inventory/portion-presets
{
  "name": "Tequila Shot",
  "familyId": "FAM-SPIRITS",
  "defaultPortions": 1,
  "maxPortions": 5,
  "pricingMode": "stepped",
  "components": [
    { "itemId": "ITEM-TEQUILA", "unitId": "UNIT-SHOT", "quantity": 1 }
  ],
  "stepPricing": [
    { "portions": 1, "price": 500 },
    { "portions": 5, "price": 2300 }
  ]
}
```

The admin UI now lists these presets so future menus/products can select them instead of rebuilding their portion logic.

### 1.6 Configure Menu Pricing

Each sellable combination of item + unit must expose a menu price so auto-pricing can work. Configure these values immediately after creating the items.

```http
PUT /v1/inventory/items/ITEM-RICE
{
  "menuPricing": [
    { "unitId": "UNIT-PORTION", "price": 450, "label": "Rice Portion", "currency": "NGN" }
  ]
}

PUT /v1/inventory/items/ITEM-HENNESSY
{
  "menuPricing": [
    { "unitId": "UNIT-BOTTLE", "price": 2500, "label": "Bottle", "currency": "NGN" }
  ]
}

PUT /v1/inventory/items/ITEM-TEQUILA
{
  "menuPricing": [
    { "unitId": "UNIT-SHOT", "price": 1500, "label": "Shot", "currency": "NGN" }
  ]
}

PUT /v1/inventory/items/ITEM-CHICKEN
{
  "menuPricing": [
    { "unitId": "UNIT-PIECE", "price": 1500, "label": "Chicken Lap", "currency": "NGN" }
  ]
}
```

---

## 2. Manage Recipes in the Admin UI

With units, presets, and menu pricing in place, recipe authoring happens entirely from the admin dashboard:

1. **Menus** – open any menu and switch to the new **Recipes** tab. The panel shows the active recipe, lets you tweak portions with the builder, and save a new version. Versions remain in draft until you hit **Activate**, at which point new orders consume against that snapshot.
2. **Products** – inventory products reuse the same tab. Toggle to **Recipes** to describe how a kit, bottled drink, or bundle should deduct stock before it hits the POS.
3. **Builder behavior** – the builder hydrates portion presets, enforces component requirements, and previews the auto-priced total (sum of component menuPricing × quantity). Notes help operations explain adjustments or wastage assumptions.
4. **Versioning** – every save creates a new version with an effective date. You can preview older versions, clone their portions back into the builder, and re-activate if needed.

This workflow keeps the admin UI aligned with the backend recipe APIs referenced later in the spec.

---

## 2. Seed Stock Lots

```http
POST /v1/inventory/lots
{
  "itemId": "ITEM-RICE",
  "branchId": "BRANCH-KITCHEN",
  "quantity": 100,
  "unitCost": 480,
  "currency": "NGN",
  "receivedAt": "2024-11-01T09:00:00Z",
  "supplierName": "Grain Supply Co",
  "notes": "4 bags at NGN12,000 each"
}

POST /v1/inventory/lots
{
  "itemId": "ITEM-HENNESSY",
  "branchId": "BRANCH-BAR",
  "quantity": 50,
  "unitCost": 45000,
  "currency": "NGN",
  "receivedAt": "2024-11-01T09:00:00Z",
  "supplierName": "Liquor Hub"
}

POST /v1/inventory/lots
{
  "itemId": "ITEM-TEQUILA",
  "branchId": "BRANCH-BAR",
  "quantity": 30,
  "unitCost": 32000,
  "currency": "NGN",
  "receivedAt": "2024-11-01T09:00:00Z",
  "supplierName": "Liquor Hub"
}

POST /v1/inventory/lots
{
  "itemId": "ITEM-CHICKEN",
  "branchId": "BRANCH-KITCHEN",
  "quantity": 500,
  "unitCost": 500,
  "currency": "NGN",
  "receivedAt": "2024-11-01T09:00:00Z",
  "supplierName": "Poultry Farms",
  "notes": "10 packs x 50 pieces"
}
```

---

## 3. Build the Catalog

### 3.1 Menu – Tequila Shot

```http
POST /v1/menus
{
  "name": "Tequila Shot",
  "price": 300,
  "code": "BAR-TEQ-SHOT",
  "description": "Single shot of premium tequila",
  "branchID": "BRANCH-BAR",
  "type": "beverage",
  "unit": "shot",
  "serving": "bar",
  "branches": [{ "branchID": "BRANCH-BAR", "code": "BAR" }],
  "portions": [
    {
      "portionPresetId": "PRESET-TEQUILA-SHOT",
      "itemId": "ITEM-TEQUILA",
      "unitId": "UNIT-SHOT",
      "label": "Shot",
      "defaultPortions": 1,
      "maxPortions": 1,
      "components": [
        { "itemId": "ITEM-TEQUILA", "unitId": "UNIT-SHOT", "quantity": 1 }
      ]
    }
  ]
}
```

Response excerpt:

```json
{
  "data": {
    "autoPrice": 1500,
    "manualAdjustment": 300,
    "finalPrice": 1800,
    "portions": [
      {
        "label": "Shot",
        "components": [
          {
            "item": "ITEM-TEQUILA",
            "unitPrice": 1500,
            "currency": "NGN"
          }
        ]
      }
    ]
  }
}
```

The service derives the NGN1,500 auto price directly from the `Tequila Blanco` menu pricing. The submitted `price` becomes a manual adjustment, yielding a final menu price of NGN1,800.

### 3.2 Product – Hennessy Bottle

```http
POST /v1/products
{
  "name": "Hennessy Bottle",
  "price": 90000,
  "description": "750ml bottle",
  "branchID": "BRANCH-BAR",
  "categoryCode": "BEV",
  "quantity": 50,
  "unit": "bottle",
  "portions": [
    {
      "portionPresetId": "PRESET-HENNESSY-BOTTLE",
      "itemId": "ITEM-HENNESSY",
      "unitId": "UNIT-BOTTLE",
      "defaultPortions": 1,
      "components": [
        { "itemId": "ITEM-HENNESSY", "unitId": "UNIT-BOTTLE", "quantity": 1 }
      ]
    }
  ]
}
```

### 3.3 Product – Kitchen Rice Bag (internal use)

This product is not sold to guests; it acts as the allocation reference for issuing whole bags to the kitchen.

```http
POST /v1/products
{
  "name": "Kitchen Rice Bag",
  "price": 0,
  "description": "Internal reference for issuing rice",
  "branchID": "BRANCH-KITCHEN",
  "categoryCode": "KITCHEN",
  "quantity": 0,
  "unit": "bag",
  "portions": [
    {
      "portionPresetId": "PRESET-RICE-BAG",
      "itemId": "ITEM-RICE",
      "unitId": "UNIT-BAG",
      "defaultPortions": 1,
      "components": [
        { "itemId": "ITEM-RICE", "unitId": "UNIT-BAG", "quantity": 1 }
      ]
    }
  ]
}
```

### 3.4 Menu – Rice Plate (two portions per serving)

```http
POST /v1/menus
{
  "name": "Rice Plate",
  "price": 300,
  "code": "KIT-RICE-PLATE",
  "description": "Plate of white rice",
  "branchID": "BRANCH-KITCHEN",
  "type": "food",
  "unit": "plate",
  "serving": "kitchen",
  "branches": [{ "branchID": "BRANCH-KITCHEN", "code": "KIT" }],
  "portions": [
    {
      "portionPresetId": "PRESET-RICE-DOUBLE",
      "itemId": "ITEM-RICE",
      "unitId": "UNIT-PORTION",
      "label": "Double Portion",
      "defaultPortions": 2,
      "components": [
        { "itemId": "ITEM-RICE", "unitId": "UNIT-PORTION", "quantity": 2 },
        { "itemId": "ITEM-CHICKEN", "unitId": "UNIT-PIECE", "quantity": 2 }
      ]
    }
  ]
}
```

Response excerpt:

```json
{
  "data": {
    "autoPrice": 3900,
    "manualAdjustment": 300,
    "finalPrice": 4200,
    "portions": [
      {
        "label": "Double Portion",
        "components": [
          {
            "item": "ITEM-RICE",
            "unitPrice": 450,
            "menuTotalPrice": 900
          },
          {
            "item": "ITEM-CHICKEN",
            "unitPrice": 1500,
            "menuTotalPrice": 3000
          }
        ]
      }
    ]
  }
}
```

Auto price (NGN3,900) equals two rice portions (2 × NGN450) plus two chicken laps (2 × NGN1,500). The NGN300 manual adjustment nudges the selling price to NGN4,200.

> If any component in the requests above references an item-unit pair without a `menuPricing` entry, the API responds with `422 item menu pricing not configured for: <item (unit)>`. Update the item’s menu pricing before retrying.

---

## 4. Process Orders (Sales)

### 4.1 Create Orders

```http
POST /v1/orders/buy-product/MEMBER-001
{
  "medium": "digital",
  "tip": 0,
  "items": [
    { "productID": "PROD-HENNESSY", "quantity": 10 }
  ]
}

POST /v1/orders/buy-menu/MEMBER-001
{
  "medium": "digital",
  "tip": 0,
  "items": [
    { "menuID": "MENU-TEQ-SHOT", "quantity": 20 },
    { "menuID": "MENU-RICE-PLATE", "quantity": 4 }
  ]
}
```

### 4.2 Initiate Payment

```http
POST /v1/orders/pay/ORDER-PRODUCT-1
{ "medium": "digital" }

POST /v1/orders/pay/ORDER-MENU-1
{ "medium": "digital" }
```

### 4.3 Paystack Webhook snippet (success)

```http
POST /v1/providers/paystack/webhook
{
  "event": "charge.success",
  "data": {
    "reference": "PSK-REF-123",
    "amount": 13500000,
    "fees": 150000,
    "paid_at": "2024-11-02T10:00:00Z",
    "channel": "card",
    "authorization": {
      "authorization_code": "AUTH-12345",
      "bin": "506099",
      "last4": "1234",
      "exp_month": "10",
      "exp_year": "28"
    },
    "metadata": {
      "action": "order-payment"
    }
  }
}
```

Once the webhook processes, both orders move to `paid`, reservations convert into consumption, and the order document stores an inventory snapshot that carries both pricing and cost information.

```http
GET /v1/orders/ORDER-MENU-1
```

```json
"inventory": {
  "deductionId": "INV-DED-001",
  "totalCost": 68400,
  "items": [
    {
      "code": "MENU-TEQ-SHOT",
      "quantity": 20,
      "menuAutoPrice": 1500,
      "menuManualAdjustment": 300,
      "menuFinalPrice": 1800,
      "menuFinalTotal": 36000,
      "components": [
        {
          "item": "ITEM-TEQUILA",
          "menuUnitPrice": 1500,
          "menuTotalPrice": 30000,
          "unitCost": 800,
          "totalCost": 16000
        }
      ]
    },
    {
      "code": "MENU-RICE-PLATE",
      "quantity": 4,
      "menuAutoPrice": 3900,
      "menuManualAdjustment": 300,
      "menuFinalPrice": 4200,
      "menuFinalTotal": 16800,
      "components": [
        {
          "item": "ITEM-RICE",
          "menuUnitPrice": 450,
          "menuTotalPrice": 3600,
          "unitCost": 480,
          "totalCost": 3072
        },
        {
          "item": "ITEM-CHICKEN",
          "menuUnitPrice": 1500,
          "menuTotalPrice": 12000,
          "unitCost": 900,
          "totalCost": 8640
        }
      ]
    }
  ]
}
```

The snapshot now contains everything finance needs: ingredient selling price (`menuUnitPrice`), the menu-level totals, and the actual FIFO cost consumed (`totalCost`).

---

## 5. Allocation Use Case – 2 Bags of Rice

To issue rice to the kitchen for prep, create and use an allocation.

### 5.1 Create Allocation

```http
POST /v1/allocations/BRANCH-KITCHEN
{
  "staffID": "STAFF-KITCHEN-1",
  "description": "Daily rice prep",
  "items": [
    { "code": "PROD-RICE-BAG", "quantity": 2 }
  ]
}
```

### 5.2 Use Allocation (consume the reserved bags)

```http
PUT /v1/allocations/use/ALLOC-001
{
  "code": "PROD-RICE-BAG",
  "quantity": 2,
  "idempotencyKey": "kitchen-2024-11-02"
}
```

This confirms the reservation and deducts 2 bags (50kg) from the rice lots.

---

## 6. Reporting Snapshots

### 6.1 Cost of Goods Sold

```http
POST /v1/reports/cogs
{
  "start": "2024-11-01",
  "end": "2024-11-02",
  "branchID": "BRANCH-BAR",
  "groupBy": "item"
}
```

**Sample JSON Response (excerpt)**

```json
{
  "data": {
    "groupBy": "item",
    "totals": { "quantity": 39.11, "cogs": 486_533.33 },
    "rows": [
      {
        "itemId": "ITEM-HENNESSY",
        "itemName": "Hennessy VS",
        "quantity": 10,
        "cogs": 450_000
      },
      {
        "itemId": "ITEM-TEQUILA",
        "itemName": "Tequila Blanco",
        "quantity": 20,
        "cogs": 32_000
      },
      {
        "itemId": "ITEM-RICE",
        "itemName": "White Rice",
        "quantity": 1.11,
        "cogs": 533.33
      },
      {
        "itemId": "ITEM-CHICKEN",
        "itemName": "Chicken Lap",
        "quantity": 8,
        "cogs": 4_000
      }
    ]
  }
}
```

### 6.2 Inventory Valuation

```http
POST /v1/reports/valuation
{
  "branchID": "BRANCH-BAR",
  "groupBy": "item"
}
```

Returns remaining quantity × unit cost for each item after the sales above.

### 6.3 Allocation Variance

```http
POST /v1/reports/allocations/variance
{
  "branchID": "BRANCH-KITCHEN",
  "start": "2024-11-01",
  "end": "2024-11-02"
}
```

Shows the 2 bags reserved and confirmed against allocation `ALLOC-001`.

---

## 7. Order and Inventory Snapshots (for auditing)

### Order Example (after webhook)

```json
{
  "orderID": "ORDER-PRODUCT-1",
  "status": "paid",
  "inventory": {
    "deductionId": "INV-123456",
    "deductedAt": "2024-11-02T10:00:00Z",
    "totalCost": 450000,
    "items": [
      {
        "code": "PROD-HENNESSY",
        "type": "product",
        "quantity": 10,
        "components": [
          {
            "item": "ITEM-HENNESSY",
            "unit": "UNIT-BOTTLE",
            "lotId": "LOT-HEN-001",
            "lotQuantity": 10,
            "unitCost": 45000,
            "totalCost": 450000
          }
        ]
      }
    ]
  }
}
```

```json
{
  "orderID": "ORDER-MENU-1",
  "status": "paid",
  "inventory": {
    "deductionId": "INV-789012",
    "deductedAt": "2024-11-02T10:00:05Z",
    "totalCost": 36_533.33,
    "items": [
      {
        "code": "MENU-TEQ-SHOT",
        "type": "menu",
        "quantity": 20,
        "components": [
          {
            "item": "ITEM-TEQUILA",
            "unit": "UNIT-BOTTLE",
            "lotId": "LOT-TEQ-001",
            "lotQuantity": 1,
            "unitCost": 32000,
            "totalCost": 32000
          }
        ]
      },
      {
        "code": "MENU-RICE-PLATE",
        "type": "menu",
        "quantity": 4,
        "components": [
          {
            "item": "ITEM-RICE",
            "unit": "UNIT-PORTION",
            "lotId": "LOT-RICE-001",
            "lotQuantity": 8,
            "unitCost": 66.67,
            "totalCost": 533.33
          },
          {
            "item": "ITEM-CHICKEN",
            "unit": "UNIT-PIECE",
            "lotId": "LOT-CHK-001",
            "lotQuantity": 8,
            "unitCost": 500,
            "totalCost": 4000
          }
        ]
      }
    ]
  }
}
```

### Stock Lot Example (after order + allocation)

```json
{
  "item": "ITEM-RICE",
  "branch": "BRANCH-KITCHEN",
  "quantityInitial": 100,
  "quantityReserved": 0,
  "quantityRemaining": 48.89,
  "status": "open"
}

{
  "item": "ITEM-TEQUILA",
  "branch": "BRANCH-BAR",
  "quantityInitial": 30,
  "quantityReserved": 0,
  "quantityRemaining": 29,
  "status": "open"
}

{
  "item": "ITEM-CHICKEN",
  "branch": "BRANCH-KITCHEN",
  "quantityInitial": 500,
  "quantityReserved": 0,
  "quantityRemaining": 492,
  "status": "open"
}
```

---

## Summary

By following these steps the admin has:

1. Established the required inventory metadata (families, units, conversions, items).
2. Seeded opening stock lots.
3. Published catalog entries for tequila shots, Hennessy bottles, and rice plates (with chicken laps).
4. Processed orders for 10 Hennessy bottles, 20 tequila shots, and 4 rice plates with accompanying chicken laps.
5. Issued 2 bags of rice via the allocation workflow for kitchen prep.
6. Generated reporting snapshots confirming consumption and remaining stock.

This end-to-end walkthrough demonstrates how each SaleTrack module connects—from configuration and fulfilment to analytics.
