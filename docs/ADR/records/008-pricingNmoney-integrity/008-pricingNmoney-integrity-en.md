# ADR-008: Pricing & Money Integrity

**Date:** 22/10/2026
**Status:** Applied

---

## 1. Context

**Business context:**
The total money of an order is the total value of its items:

`total = Σ ((base price + number of salted eggs × 5,000) × quantity)`

From the moment the customer chooses the cakes until PayOS confirms the payment, the money goes through three steps: putting products in the cart, confirming the order information, and paying.

- In the cart step, the FE is the place that holds and shows the total money of the products and the order that the customer has to pay. The information is stored in localStorage, and the products in the cart are kept for 30 minutes.
- In the confirmation step, the FE reuses the information stored in localStorage to show the customer the total amount to pay.
- In the payment step, the page loads the amount of the order that was already created on the BE; this amount is stored in the `Order` table.

**Risk scenario if the money is incorrect:**
If the money is incorrect at any step, the amount the customer confirmed and the amount the customer has to pay may be different.

**System invariants:**

- The place that decides the final amount of an order is the data that the BE has saved in the database.
- The order amount, the amount sent to PayOS, and the amount in the webhook must be the same.

---

## 2. Options Considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| **A: Store as `Int`** | Cake prices have no decimals | If taxes are added later, calculations in percentages will be troublesome | Not certain enough |
| **B: Store as `Decimal`** | More certain | It must be configured correctly: Prisma returns `Decimal` as an object, not a number, so `Number()` is needed every time we calculate. If the precision is not declared, Postgres creates data with 30 decimal places | — (chosen option) |

---

## 3. Decision

**How the price is calculated:**

- **Price of one cake:** the BE gets the `basePrice` of the cake from the database, then adds 5,000 VND for each salted egg: `unitPrice = basePrice + eggCount × 5,000`. The function is in `pricing.constants.ts`.
- **Total money of an order:** in the `createOrder` step, the BE loops through the items, calculates the price of each cake, then adds `unitPrice × quantity` to the total: `totalMoney = Σ (unitPrice × quantity)`.

**How it is stored and the data types:**
Money is stored in these tables and columns: `Order.totalMoney`, `Cake.basePrice`, `OrderItem.priceAtPurchase`, `PaymentTransaction.amount`. The type conversion and rounding are done in the functions that use this data. I chose this way because it is clearer and more organized if taxes are applied later.

**Amount sent to PayOS and webhook comparison:**

- The amount sent to PayOS is taken from the database, converted to a number with `Number()`, and rounded with `Math.round()`.
- When the webhook comes back, the amount is compared with the `totalMoney` column in the `Order` table. Both are rounded with `Math.round()` and compared with `!==`. When they do not match, the BE logs an error and returns `success: false` with the message "Webhook received but amount does not match".

---

## 4. Trade-offs & Limitations

**Accepted trade-offs:**

- **Technical trade-off:** I accepted that I have to write every command that uses these values very carefully. Because Prisma `Decimal` returns an object, `Number()` must be used every time. The money columns do not declare precision, so Postgres creates data with 30 decimal places. But to be sure about the numbers used for taxes, and to make rounding easier, I chose `Decimal` instead of `Int`.
- **Business trade-off:** I did not trade off much, because I try to show round numbers to my mom.

**Limitations / inconsistencies in the current codebase:**

- When the price shown on the FE is different from the price calculated on the BE, the customer sees the FE price but has to pay the price calculated by the BE.
- When the admin changes the price, the products in the cart do not update their price automatically. At payment time, the customer will see a completely new price, because the BE has just taken the price the admin changed to calculate.
- When important problems happen, such as the amount in the webhook and the amount in the system being different, the system does not yet have a way to tell the admin that an error has appeared.

---

## 5. What I'd do differently

- I would add a step to compare the price between the FE and the BE, maybe like this: the FE sends the amount the customer has seen, and the BE calculates it again; if they are different, the BE rejects the request and returns the new price so the customer can review it.
- When the customer enters the order confirmation page, I would get the latest cake price again and tell the customer if the price has changed.
- The system already has Telegram notifications. The reason I did not build error notifications is that I was afraid my mom would get too many notifications and be confused. If I did it again, I would split them into two different Telegram chat rooms, and the error notifications would only go to the admin, the person who manages the system, so that only the admin knows about them.