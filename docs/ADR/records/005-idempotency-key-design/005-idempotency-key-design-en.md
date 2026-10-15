# ADR-005: Idempotency Key Design

**Date:** 08/10/2026
**Status:** Applied

---

## 1. Context

**Business context:**
A customer may click the submit-order button several times, or a network problem may cause the server to receive duplicate requests.

**Risk scenario if there is no defense mechanism:**
If duplicate requests get into the system, the worst data inconsistency happens in the Order, OrderItem and Availability tables: the system creates an extra "ghost" order and deducts the matching cake slots. When this happens, if customers see that the shop has sold out, revenue drops. In addition, if the baker receives this "ghost" order, they will make more cakes than are actually needed.

**System invariants:**
For the same Idempotency Key (the same ordering intention), the API must make sure the request is executed only once and returns the result of that one execution.

When a user refreshes the page or resends a request with an old key, the Guard only checks the status of the key in Redis. If the key is `PENDING`, the request is rejected with status code `409`. If the key is `SUCCESS:<orderId>`, the Guard returns `409` together with the order ID, so the frontend can load the order and redirect the user according to the payment method.

**Status verification (Applied and Assumed):**
In the Bánh Pía PMS system, there is already a mechanism to prevent duplicates, using an Idempotency Key hashed from these fields: `phone`, `items`, `receiveDate`, `paymentMethod`. Each combination of information can only create one order while the key is still valid. Hashing these fields into a key also allows the same user to place another order while the key still exists, as long as the new order does not have the same information as the old one. This approach handles most orders, but it is only suitable for the small-to-medium traffic of the current system.

If the system grows, this approach is not enough: the remaining cases need to be handled, and if Redis goes down, the key also needs to be stored in the database to act as the "source of truth".

---

## 2. Options Considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| **A: The FE creates a new uuid every time the user submits an order request** | Easy to implement, light on performance | It only prevents duplicate orders when the user does not refresh the page. If the page is refreshed, the FE creates a new uuid, so the problem is not solved. | Not safe enough, many duplicate cases still get through. |
| **B: Hash the fields `phone`, `items`, `receiveDate`, `paymentMethod` as the Idempotency Key, combined with Redis GET, SET to check the status of the key** | If the user refreshes the page, each order still has its own separate key, so duplicate orders are prevented. Also, during the 10-minute TTL, the user can still place another order. | It cannot handle the case where, within those 10 minutes, the user places an order with exactly the same information as the old one. | — (chosen option) |

---

## 3. Decision

I chose to hash the order fields (`phone`, `items`, `receiveDate`, `paymentMethod`) as the Idempotency Key with a 10-minute TTL, combined with Redis GET, SET to quickly access the status of the key.

The FE uses the fields above as the payload and hashes them with SHA-256 to get a hash. The key is sent to the backend through the `x-idempotency-key` header when calling the `POST /booking/create` API. The Guard on the backend takes the key from the header, builds the Redis key, and then reads the status of the key. If the Guard returns `409` with the order ID, the FE redirects the user according to the payment method instead of showing an error, so the user has a better experience. If the status is `PENDING`, the system shows "Processing".

---

## 4. Trade-offs & Limitations

**Accepted trade-off:**
To get safety against duplicates, the system accepts higher code complexity, and every create-order request has to run additional Redis commands (latency has not been measured).

**Real limitations in the current codebase:**

- The system cannot yet handle the case where an error or failure happens halfway through. At the moment, only errors inside the transaction are handled — for example, if a cake is not found or the delivery address is missing, the transaction is rolled back. However, the Guard has already written `PENDING` before calling the service, and no part of the system cleans up the key when an error happens.
- Two requests that arrive at the same time both GET and see that the key does not exist yet, then both continue — this is exactly the double-click situation. So using only GET and SET on Redis is not enough.
- Because the key is not saved in the database (unique), if Redis fails, the system has no source to check the key again, even after Redis has recovered.
- The current system is only suitable for small-to-medium customer traffic with few variables. If it grows, cases such as orders with the same information, Redis going down, or the server going down will be a big weakness for revenue and customer experience.

---

## 5. What I'd do differently

The ideas below came up after I reviewed the system again. They were not among the options I considered when I wrote the code.

After rewriting the pseudocode based on the current codebase and researching more about the limitations of the system, if I could do it again, I would do the following:

- Stop hashing the order fields as the Idempotency Key. Instead, the FE will create a uuid when the purchase session starts (when the customer is on the payment information page). This way, even if the customer refreshes the page, they stay in the same purchase session and reuse the same key, unless they leave the payment page.
- Add functions to delete the key in the necessary cases, for example when the system encounters an error.
- Use `SET key value NX EX 600` in Redis instead of the basic GET, SET as it is now.
- Combine this with saving the key in the database with a unique constraint, so that even if Redis goes down, the database is still the source of truth and acts as a lifesaver when the system has problems.