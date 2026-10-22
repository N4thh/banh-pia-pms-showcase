# ADR-007: PayOS OrderCode Abstraction

**Date:** 20/10/2026
**Status:** Applied

---

## 1. Context

**Business context:**
From the moment the customer chooses bank transfer until the order is recorded as paid, there is a step that sends a request with a code (`payosOrderCode`) to PayOS to create a payment link. In this step, a new record is written to the `PaymentLink` table, and the system uses the `orderId` code to identify the order.

My system and PayOS each identify the order with a different code: my system uses `orderId`, while PayOS uses `payosOrderCode`.

**Risk scenario if there is no separate mechanism for the code sent to PayOS:**

- The internal code and the code sent to PayOS serve two different purposes, so keeping them separate makes the code clear and avoids confusion.
- The code sent to PayOS is generated randomly, so an outsider who only has this code cannot know which order it belongs to.

**System invariants:**

- Each order has at most one `PaymentLink` record, because a `@unique` constraint in the database guarantees this.
- The code sent to PayOS must be an integer and must be unique; our database requires the same.

---

## 2. Options Considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| **A: Use `orderId` as `payosOrderCode`** | Convenient, can be used right away | An outsider who reads the payment code may know which order it belongs to | It does not meet the original requirement |
| **B: Clearly separate `orderId` and `payosOrderCode`** | Even if an outsider gets the `payosOrderCode`, they cannot know which order it belongs to | The invariants and the way to generate the code must be designed separately | — (chosen option) |

---

## 3. Decision

I chose to fully separate `orderId` and `payosOrderCode`. The code sent to PayOS is generated randomly from 100,000,000 to 999,999,999. Each order has only one PayOS code, and this code is stored in the `PaymentLink` table.

The reason is that the purpose of each data field becomes clear, and each field serves its own purpose. In addition, if an outsider gets the PayOS code, they still cannot know which order the code was used for.

**Flow when the create-payment-link function is called:**

- If the order does not have a link yet, the function creates a new link.
- If the order already has a link, the function returns data such as the URL, QR code, and so on, so the FE can use it and move to the page that matches the order status.

**Order of steps:**

1. Find the order and check whether the order exists.
2. Check whether the FE base address has been set.
3. Check whether the link already exists. If it does, return the old link; if not, continue to create the link: generate a random `payosOrderCode` from 100,000,000 to 999,999,999.
4. From this step, use `try/catch`: call PayOS to create the payment link; if it succeeds, write the data returned by PayOS as a new record in the `PaymentLink` table.
5. If one of these two steps fails, return a 500 error.

**When PayOS sends a webhook:**
The system uses the `payosOrderCode` in the webhook data to find the `PaymentLink` record, and from it finds the matching order.

---

## 4. Trade-offs & Limitations

**Accepted trade-offs:**

- **Technical trade-off:** The invariants and the way to generate the code have to be designed separately.
- **Business trade-off:** There is no clear trade-off.

**Limitations / inconsistencies in the current codebase:**

- At the moment, when an order is cancelled, the system only does a soft cancel of the order (changing its status to CANCELLED) and does not touch `PaymentLink`, so the payment link still exists, both on the system side and on the PayOS side. If a customer still pays through that link, the system will handle it incorrectly, even though the order has been paid.
- The random generation from 100,000,000 to 999,999,999 can still produce the same code twice, and the system does not have a check for this yet.

---

## 5. What I'd do differently

- I would make the payment link cancellation flow better by cancelling on both sides: the system and PayOS. This keeps both sides in sync and prevents the case where a customer still pays through PayOS's own link.
- I would check for duplicates of `payosOrderCode` to make sure each generated link is unique.