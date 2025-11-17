# Event Ticket Flow Verification

This checklist exercises the end‑to‑end event purchase → ticket delivery → check‑in → rollback paths after the latest backend changes.

> Prerequisites  
> - MongoDB running locally with the `latonilux-backend` `.env` values loaded.  
> - Queue workers started (`yarn workers` or `npm run workers`) so `sendEventTicket` jobs are processed.  
> - Optional: a REST client (Postman, Bruno, Thunder Client) for the HTTP calls shown below.

## 1. Seed minimum data

1. Create (or reset) an event that has capacity:
   ```bash
   # Inspect existing events
   mongo --eval 'db.events.find({}, {eventID:1, "ticket.slot":1}).pretty()'

   # Optional: delete test data
   mongo --eval 'db.orders.deleteMany({ "summary.item": /TEST/ })'
   mongo --eval 'db.transactions.deleteMany({ "metadata.displayName": "action", "metadata.value": "order-test" })'
   ```
2. Make sure a member exists (`members` collection) and capture the `_id` for upcoming API requests.

## 2. Place an event order

- Endpoint: `POST /v1/orders/buy-event/:memberId`
- Example payload:
  ```json
  {
    "code": "EVT123",
    "quantity": 2,
    "medium": "digital",
    "participants": [
      { "firstName": "Ada", "lastName": "Lovelace", "email": "ada.test+1@example.com" },
      { "firstName": "Grace", "lastName": "Hopper", "email": "grace.test+1@example.com" }
    ]
  }
  ```
- Expectation:
  - 200 response containing the order summary.
  - Mongo `orders` document now includes the `participants` array with the same people.
  - `events.members[].participants` contains sanitized entries (ticket codes will be assigned later).

Verification query:
```bash
mongo --eval 'db.orders.find({orderID: "<ORDER_ID>"}, {participants:1, summary:1}).pretty()'
mongo --eval 'db.events.find({eventID: "EVT123"}, {"members.participants":1, "ticket":1}).pretty()'
```

## 3. Simulate payment success (queues & ticket generation)

For cash settlements, invoke:
```bash
curl -X POST http://localhost:5000/api/identity/v1/orders/pay/<orderId> \
  -H "Content-Type: application/json" \
  -H "x-access-token: <admin-session-token>" \
  -H "channel: web" \
  -d '{ "medium": "cash" }'
```

Once the queue worker processes `latonilux:send-event-ticket`:
- Each participant should now have:
  - `ticketCode` (unique `EVxxxxxx` code).
  - `isSent = true`, `sendAt`, and `ticketUrl`.
- Email attachments / generated PNG include the ticket code.

Confirm via:
```bash
mongo --eval 'db.events.find({eventID:"EVT123"}, {"members.participants":1}).pretty()'
```

Optional: inspect worker logs to ensure the job succeeded.

## 4. Check-in endpoint

- Endpoint: `PUT /v1/events/check-in/:eventId`
- Payload (one of `participantId`, `ticketCode`, or `email` is required):
  ```json
  {
    "ticketCode": "EVABC123",
    "email": "ada.test+1@example.com"
  }
  ```
- Expected response: participant detail with `isCheckedIn: true`.

Re-query participants to confirm check-in metadata:
```bash
mongo --eval 'db.events.find({eventID:"EVT123"}, {"members.participants":1}).pretty()'
```

Also test the filtered listing:
```bash
curl -H "x-access-token: <admin>" -H "channel: web" \
  http://localhost:5000/api/identity/v1/events/participants/<eventId>?status=checked-in
```

## 5. Payment failure rollback

Simulate a webhook failure (Paystack) by POSTing a crafted payload to the webhook endpoint (or temporarily calling `TransactionService.failTransaction` in a script). After the failure handler runs:
- Order status should be `payment_failed`.
- Event slot count should be restored (`ticket.slot` increments).
- `event.members` no longer lists the unpaid participants (or the array becomes empty).

Validate:
```bash
mongo --eval 'db.orders.find({orderID:"<ORDER_ID>"}, {status:1, participants:1}).pretty()'
mongo --eval 'db.events.find({eventID:"EVT123"}, {"members":1, "ticket":1}).pretty()'
```

## 6. Regression & automation

1. Run existing unit/integration tests if configured:
   ```bash
   npm test
   # or
   yarn test
   ```
2. Lint/build for sanity:
   ```bash
   npm run lint && npm run build
   ```

## 7. Cleanup

Remove test data as needed:
```bash
mongo --eval 'db.events.updateOne({eventID:"EVT123"}, {$set: {"members": []}})'
mongo --eval 'db.orders.deleteMany({orderID: "<ORDER_ID>"})'
mongo --eval 'db.transactions.deleteMany({order: ObjectId("<ORDER_ID_OBJECT>")})'
```

---

Repeat steps 2–5 with different mediums (digital vs cash), participant counts, and failure scenarios to cover edge cases such as duplicate ticket codes or partial check-ins.***
