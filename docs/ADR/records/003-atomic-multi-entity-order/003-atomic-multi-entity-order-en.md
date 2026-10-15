## ADR-[003]: Atomic Multi-Entity Order Creation & Boundary Design

**Date Written:** [25/09/2026] **Status:** [Implemented]

---

### 1. Context

1. **How is this problem different from ADR-001 (only 1 Availability table)?**
2. **What specific scenario shows that if this is not handled correctly, the system can break?**
3. **Why can't this problem be solved by writing to each table independently and checking errors manually after each step?**

Answers:

1.

When the user clicks the place order button, related data tables such as: order, address, order items, user, and availability will be created at the same time with their corresponding information.

Why does writing to multiple tables at the same time create risks? Because the level of dependency between these tables affects data consistency. If any table is missing or contains incorrect information, it can cause data inconsistency between the order and the customer.

In addition, data types must also be handled correctly because different tables may use different data types, which requires a higher level of accuracy.

2.

A real example is when the Availability table is successfully written to the database, but the order creation step right after that fails. As a result, cake slots are deducted, but the order does not actually exist.

This violates the atomicity of the transaction.

If this happens many times, the system may incorrectly show products as sold out even though no actual sales happened, causing real revenue loss. It can also confuse administrators and negatively affect the user experience for both admins and customers.

3.

In reality, manually creating records in each table is still possible, but it only works well if every step always succeeds.

If any step fails, we have to manually clean up orphan records, and that is not an optimal approach.

The solution here is to wrap all creation operations inside a transaction so that all information is written consistently and remains synchronized across tables.

---

### 2. Options Considered

| **Option** | **Advantages** | **Disadvantages** | **Why Not Chosen** |
|------------|---------------|------------------|-------------------|
| A: Execute separate statements and write each table independently | Easy to implement | If any step fails, orphan records may be created and must be cleaned up to keep data consistent | Requires additional manual cleanup and creates risk if inconsistent data is not handled in time. |
| B: Put both DB transaction and Redis hold key inside one transaction | Matches the original requirement (everything succeeds or fails together), and the code looks cleaner | Does not guarantee that the Redis hold key works together with database transaction operations | Because Redis and DB transactions operate at different layers. Wrapping Redis inside a DB transaction is meaningless because Redis cannot participate in the database rollback mechanism. |
| C: Use DB transaction for all database operations and keep Redis hold key outside | Meets all requirements we need | More complex code | |

---

### 3. Decision

I chose to use a DB transaction to group all related database write operations together, ensuring that all related tables are written at the same time.

If any database operation fails, all completed and pending operations will be rolled back, and the process will stop immediately.

At the same time, I keep the Redis hold key outside the transaction because DB and Redis operate at two different layers. Therefore, if the transaction succeeds but Redis fails afterward, the order is still created because Redis failure should not stop the transaction.

a) Why keep Redis outside the transaction:

I consider the database as the source of truth, while Redis is only a temporary supporting layer. Therefore, even if Redis fails, the main business data still remains correct in the database.

Also, Redis cannot participate in the rollback mechanism of the database transaction.

b) Why use Redis hold instead of querying the DB continuously (performance reason):

We should remember that database data is read from storage devices (SSD/HDD), while Redis data is stored in RAM. Therefore, the access speed is significantly different.

If the system continuously queries the database to calculate the remaining payment time for an order (10 minutes), the query cost becomes much higher, especially when many users place orders at the same time.

This is why Redis was chosen as a temporary state layer from the beginning, and this decision is unrelated to rollback behavior.

---

### 4. Trade-offs & Limitations

- At first, I planned to put the Redis hold key inside the transaction because I expected that if Redis failed, the transaction would also be rolled back. However, after reviewing and researching more about how DB and Redis work, I realized that approach was incorrect and changed to the current solution.
- For bank transfer payments, the slot hold only exists for 10 minutes. To clean up expired orders, I use a compensation mechanism: a cron job scans orders with status 'NEW' that have passed the allowed payment time, compares the order time with the current time, and updates them to 'CANCELLED' with the reason 'payment expired'.
- While rewriting the pseudocode for the createOrder function, the biggest thing I discovered was that the redis-hold-key should not be placed inside the try-catch block that handles Redis failures.

---

### 5. What I'd Do Differently

If I could do it again, I would list all possible consequences more systematically so I could handle and report errors in greater detail.

One example is the situation where the Redis hold key fails but the DB transaction still succeeds. This made me realize that I had missed a potential issue.

In addition, the createOrder function is currently quite long and complex. If I were to redesign it, I would organize it better because even when rewriting the pseudocode, I had to remind myself several times to remember all the required steps.