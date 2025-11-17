# Orders API

Reference for the order endpoints updated during SaleTrack Steps 7 and 10, covering order creation, payment, fulfillment, and cancellation with the new inventory hooks.

## Common Notes

- Base route: `/v1/orders`.
- Authentication required. Creation endpoints are open to admin/member roles as indicated; payment/fulfillment requires `superadmin` or `admin` unless otherwise noted.
- Order statuses: `pending`, `processing`, `paid`, `payment-failed`, `cancelled`, `completed`.
- All responses use the standard envelope: `{ error, errors, data, message, status }`.

---

## Create Orders

These endpoints orchestrate the initial capture of demand. They validate catalogue references, create provisional orders, and reserve inventory (for menus/products) or seats (for events/facilities). Orders remain in a `pending` state until payment succeeds.
### POST `/v1/orders/buy-menu/:memberId`

- Roles: `superadmin`, `admin`, `member`.
- Body example:
  ```json
  {
    "medium": "digital",
    "tip": 1000,
    "items": [
      { "menuID": "MENU-BAR-001", "quantity": 2 },
      { "menuID": "MENU-BAR-002", "quantity": 1 }
    ]
  }
  ```
- Creates a `menu` order tied to the specified member (or generates a guest ID if none exists). Portions are resolved into inventory reservations automatically.

### POST `/v1/orders/buy-product/:memberId`

- Body structure mirrors menu orders but uses `productID`.
- Each product line reserves stock via `ProductService.reserveStock` to lock FIFO lots until payment succeeds.

### POST `/v1/orders/buy-event/:memberId`

- Body:
  ```json
  {
    "medium": "digital",
    "code": "EVENT-202",
    "quantity": 4,
    "participants": [
      { "firstName": "Ada", "lastName": "Lovelace", "email": "ada@example.com" },
      { "firstName": "Grace", "lastName": "Hopper", "email": "grace@example.com" }
    ]
  }
  ```
- Validates participants and ensures quantity matches participant count.

### POST `/v1/orders/buy-facility/:memberId`

- Same contract as event orders but targets facility bookings.

Each create endpoint returns the provisional order plus payment metadata (for digital medium) so the frontend can redirect to the provider checkout.

---

## Payment

Payment endpoints trigger the financial workflow. Digital payments create Paystack transactions and rely on webhooks to finalise inventory deductions. Cash payments shortcut the flow by immediately marking the transaction successful.
### POST `/v1/orders/pay/:id`

- Body: `{ "medium": "digital" | "cash" }`
- Rules:
  - `medium = digital`: order must be `pending` or `payment-failed`. The controller calls `OrderService.initOrderPayment`, which creates a Paystack transaction and returns payment instructions.
  - `medium = cash`: allowed only for admins; immediately completes payment (`OrderService.payCashOrder`).
- On success, response `data` contains either the paid order (cash) or the Paystack init payload (digital).

### Webhook Side-Effects

- Paystack webhooks (handled by `/v1/providers/paystack/webhook`) finalize the transaction, call `OrderService.finalizeOrderPayment`, and confirm stock. No additional client action required beyond verifying status.
- Successful finalisation writes an `inventory` snapshot onto the order. Each menu line now includes `menuAutoPrice`, `menuManualAdjustment`, `menuFinalPrice`, `menuFinalTotal`, and every component carries `menuUnitPrice`, `priceLabel`, `currency`, and the consumed lot references. Use this data for back-office reconciliation or audit reports without recomputing prices.

---

## Fulfillment & Completion

Once payment clears, operational staff can fulfil or complete orders. Fulfilment records consumption/production milestones, while completion flips the order to its terminal state and triggers reporting pipelines.
- `PUT /v1/orders/fulfill/:id`
  - Body: `{ "code": "FUL-REF-123" }`
  - Marks the order as fulfilled and triggers downstream jobs (`OrderService.fulfillOrder`).
- `PUT /v1/orders/complete/:id`
  - Body not required.
  - Only admins can complete orders; requires status `paid`.

---

## Cancellation / Deletion

Cancellation safeguards inventory integrity. Before deletion the platform releases any reserved stock so FIFO totals remain accurate. Use this path for abandoned or test orders.
- `DELETE /v1/orders/:id`
  - Role: `superadmin`.
  - Before deletion the controller calls `OrderService.releaseProductReservations(order)` to ensure any outstanding product reservations are released.
  - Useful for aborting pending orders; reservations are freed immediately, preventing stock locks.

- Failed transactions (`TransactionService.failTransaction`) also invoke `releaseProductReservations`, so explicit deletion is typically only needed for manual admin cancellations.

---

## Fulfillment Jobs & Hooks (FYI)

Background jobs keep associated collections (transactions, branches, ticket queues) in sync. Ensure the queue workers are running whenever new code is deployed so these hooks execute promptly.
- Successful payments queue the following jobs:
  - `AddTransactionsToResource` – links transactions to staff/branches/departments.
  - `AddOrdersToResource` – links orders to staff/branches.
  - `AddTransactionToItems` – propagates transaction references to ordered items.
- Event/facility orders queue ticket delivery jobs (`sendEventTicket`, `sendFacilityTicket`). Ensure workers are online before deploying.
