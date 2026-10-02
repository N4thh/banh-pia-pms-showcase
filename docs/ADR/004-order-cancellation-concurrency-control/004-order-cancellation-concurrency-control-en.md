# ADR-004: Order Cancellation & Concurrency Control

**Date:** 01/10/2026
**Status:** Applied

---

## 1. Context

**Question 1.1:** In the Bánh Pía PMS system, which 3 independent actors can all change the status of an order? What is the worst-case conflict scenario between Admin and the BullMQ delayed worker happening at the same millisecond?

**Question 1.2:** What does the invariant about an order's final status require? If a payment webhook arrives late, after Cron/Admin has already cancelled the order, how should the system handle it so the order does not "come back to life"?

**Question 1.3:** What does the invariant about releasing cake slot capacity say about the maximum number of times a slot can be released for one cancelled order? If it is released twice, what is the business impact?

**Answer:**

**1.1:** In the current PMS system, there are 3 independent actors that can change an order's status: the Admin, the BullMQ delayed worker (which handles expired orders), and the PayOS webhook. The worst-case scenario is a conflict between Admin and the worker at the exact same millisecond — and this is not just a theoretical risk: after reviewing the code (see section 4.3), I confirmed that the worker originally updated the order status directly, without going through the same locking mechanism as `cancelOrderById`. This meant both could read the status as `'NEW'` at the same time and both release the slot, leading to overbooking from the slot being released twice.

**1.2:** The PMS system has an invariant about an order's final status: once an order is `"CANCELLED"` or `"COMPLETED"`, it can never go back to any earlier status. If a payment webhook arrives late, after Cron/Admin has already cancelled the order, the system checks whether the order is already in one of these two statuses, and if so, returns an error saying the order has already been cancelled/completed.

**1.3:** The current PMS system only allows a slot to be released exactly once per cancelled order. As mentioned in 1.1, releasing it twice would create extra slots that do not match reality, causing overbooking — a serious violation of capacity integrity.

---

## 2. Options Considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| **A: Check order status step by step, then call release-hold-slot and update the order sequentially, without wrapping in a transaction** | Easy to implement | If one of the two steps fails, it creates inconsistent data | Does not guarantee the safety the system requires, and can easily create data-consistency gaps in the order. |
| **B: Use `updateMany`; if the DB returns `count = 1`, then release-hold-slot is allowed to run** | Later flows do not need to wait, so they can run at the same time | `updateMany` only returns the number of updated records (a number), not the order details. But we need the order items to call release-hold-slot, so a separate `find` call is needed. It also loses the ability to return a detailed error to the User/Admin. | Adds several extra steps, and still has to be wrapped in a transaction in the end — so it does not actually reduce complexity. |
| **C: Use pessimistic locking and wrap both operations in one transaction** | Guarantees both operations happen correctly. Returns the order items directly, without needing a second check. Returns a detailed error to the User/Admin. | More complex to implement. Must make sure the transaction is passed correctly through child services (tx propagation). | — |

---

## 3. Decision

**3.1:** We use a parent transaction and pass the database client between services. Without passing `tx`, the child service (release hold slot), even if it opens its own transaction, would create a separate DB connection, outside the main transaction. Passing `tx` through the whole flow makes sure the entire `cancelOrderById` runs on the same connection, inside the same transaction boundary.

**3.2:** We use `SET LOCAL lock_timeout` of 3 seconds so that a `SELECT FOR UPDATE` does not wait too long to acquire a lock, which would otherwise make other flows hang indefinitely and exhaust the connection pool.

**3.3:** At the SQL level, the slot-release function uses the formula `GREATEST(0, expression)` to make sure the slot count never goes negative.

**3.4:** When the worker (BullMQ) automatically cancels an expired order, it calls `cancelOrderById` directly, with `adminId = null` to represent a system-initiated cancellation. This makes sure both Admin and the worker go through the same transaction boundary and the same locking mechanism.

---

## 4. Trade-offs & Limitations

**4.1 (Performance trade-off):** The chosen solution uses pessimistic locking with a 3-second window to guarantee safety. In exchange, later operations have to wait their turn, which reduces system throughput. However, 3 seconds is only the maximum wait time to acquire the lock, not the actual time each operation takes. In practice, one operation only takes a few milliseconds, and with the current small-to-medium user traffic, the queue that builds up is negligible.

**4.2 (Deliberately redundant lock on Availability):** In theory, the `UPDATE` statement that releases the slot is already atomic. However, I still chose to use a manual `SELECT FOR UPDATE` on the `Availability` table to keep the codebase consistent, so that anyone (including myself later on) reading the code would clearly see the intent behind this decision, instead of mistaking it for a missed step.

**4.3:** Originally, the worker that handles expired orders (BullMQ) updated the order directly, without using the same mechanism as the booking service's `cancelOrderById` — this created a hidden race condition: both the worker and the admin could read the `'NEW'` status at the same time and both release the slot. After reviewing this, I changed the worker to call the same `cancelOrderById` function as the admin, so the locking mechanism is consistent and the problem above is fully resolved.

---

## 5. What I'd do differently

**5.1:** My system already has BullMQ + Redis set up, with a worker branch already written to report orders cancelled due to expiry. However, because this notification was burying my mom's new-order notifications in the message flow, I decided not to turn this feature on. If I were to redo this, I would redesign how my mom receives notifications in a clearer way, so she still gets notified when an order is cancelled, without it getting mixed in with new-order notifications.