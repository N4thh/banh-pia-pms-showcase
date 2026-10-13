# ADR-006: Capacity Policy (Soft Limit & Hard Limit Alignment)

**Date:** 13/10/2026
**Status:** Applied

---

## 1. Context

**Business context:**
In the bakery's business model, the need to separate standard production (Soft Limit) from the maximum extended capacity (Hard Limit) comes from the following reason: it makes it easier to manage and monitor the situation, to know how many orders go over the actual amount of cakes, and then to use this information to adjust the number of cakes for the following days.

The capacity buffer brings two values:

- **Better user experience:** In a real example, when a customer accidentally orders more than the standard cake capacity, without the extended slots the system would show an error immediately, even if the order is only a few cakes over.
- **Revenue optimization:** When running a bakery, pre-orders are sometimes cancelled for some reason, and as a result the cakes already made for that day are left over. A buffer can help reduce this waste and increase the shop's revenue.

**Risk scenario if there is no capacity management mechanism:**
If the system accepts orders beyond the real production ability, the baker (my mom) will have to work at a higher capacity than usual, which creates a bad experience for the person who is running the bakery. In addition, the service quality may go down if the quality of the cakes and the time to take care of customers are reduced.

If the system stops accepting orders too early, right when it reaches the standard limit, the bakery will miss two opportunities. The first is the chance to find out that it can sell more cakes than the planned amount; I see this as an opportunity cost. The second is the chance to get an extra source of revenue from those cakes.

**System invariants:**
An order that falls within the standard capacity is checked by the system to see whether it exceeds the limit. If it does not, the order is labeled "CONFIRMED": a normal order that meets the requirement.

An order that exceeds the standard capacity but not the maximum limit is labeled "WAITLIST": these are orders outside the plan. These orders are checked to see whether they go over the limit (more than 103% of the normal capacity). If they do, the system returns a Conflict Exception error; if they are valid, the order gets the matching status label.

The absolute constraint of the system when an order goes over the limit: the total number of cakes already ordered on that day, plus the number of cakes the customer wants to order, must not exceed the buffer (103% of the total amount of cakes the bakery will make that day). If it does, the order has broken the system rule and an error is returned immediately; on the FE, the number of cakes that exceeds the limit will be shown.

---

## 2. Options Considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| **A: Use only one hard limit** | Simple, the logic is easy to understand, and it does not take much time. | The bakery loses opportunities it could have. It lowers the user experience when a problem happens. | It makes the system too rigid and loses "valuable" data. |
| **B: Two levels, Soft Limit & Hard Limit with a buffer** | Improves the user experience, and also adds a small part of revenue for the bakery. | It has to check in several steps, and creates constraints that the system must have to keep the variables under control. | — (chosen option) |

**Comparison question about the capacity management mechanism:**
What is the difference between rejecting immediately when the standard limit is reached and allowing more orders in a controlled buffer?

It is the "softness" inside a "rigid" system: loosening constraints that are too strict because of the numbers.

---

## 3. Decision

**Formula & capacity configuration:**
The buffer limit formula = Max Capacity × 1.03 (rounded up to a whole number).

**Order of checks & blocking condition:**
When a new order comes in, the system calculates the accumulated total (`newBooked`) = the total number of cakes already ordered that day + the number of cakes in the new order.

The condition to throw an exception and reject the order is when the accumulated total > the buffer. At that point, the system returns this message: "Today we cannot process more orders. Please order for another day, thank you."

**Classification and reality:**
The "waiting list" (wait list) and "confirmed" statuses are created after the order has passed the constraints above. If the order does not exceed the available quantity, it is "confirmed"; if it is inside the buffer, it is "waiting list".

However, at the moment these statuses only carry information in the code; in reality, no function afterward uses them to run any business logic.

---

## 4. Trade-offs & Limitations

**Accepted trade-offs:**

- **Business trade-off:** Allowing more orders in the buffer brings revenue, but it can unintentionally put pressure on the baker (my mom). However, a 3% buffer is not a very large number, and it is acceptable in exchange for the opportunities and advantages it brings.
- **Technical trade-off:** I chose to hardcode the buffer because I wanted to keep everything simple and light while managing the bakery for my mom, so I did not want extra steps to adjust the buffer percentage.

**Limitations / inconsistencies in the current codebase:**
The statuses used to classify the buffer are not saved to the Database, so identifying which orders belong to the buffer can only be done by looking at the numbers. In reality, the system already makes this division but does not record it, which makes this step redundant.

---

## 5. What I'd do differently

- **Add an order status column to the Database:** Add a data column to the Database to record exactly which orders belong to the buffer, which makes operational data more transparent.
- **Flexible configuration:** Instead of hardcoding, I would add a screen so that, in the future, if I want to change the buffer percentage, I can do it directly in the system and do not have to find and change the exact line of code.
- **Operation & monitoring process:** Build an alert mechanism or a process to handle the waiting list, for example sending a notification when the number of orders reaches the allowed level in the buffer.
