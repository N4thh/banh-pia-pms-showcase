# ADR-011: Asynchronous Processing with BullMQ

**Date:** 12/11/2026
**Status:** Applied

## 1. Context

The system has two tasks that should not run while the customer is waiting for the order response.

**Telegram notification:** Telegram is a third-party service, and I cannot control its speed. Each call to Telegram is a call over the internet. If that side is slow or fails, the response to the customer is also slow. So this task should run after the result has been returned to the customer.

**Cancelling expired bank transfer orders:** this task only needs to run 10 minutes after the order is created, if the order has not been paid. So it needs a timer to run later. It cannot be done inside the order creation request.

No incident happened before this. I built it because I was worried that many events at the same time could overload the notification module, and because I wanted to learn more about message queues.

## 2. Options Considered

| Option | Pros | Cons | Why chosen or not chosen |
|---|---|---|---|
| A: Send the event directly to the notification module | Simple and easy to understand | I was worried that if all events arrive at the same time and go to the notification module, it could be overloaded and the server could become too busy | Not chosen: I wanted to learn more about message queues and I thought a queue would be safer |
| B: Use BullMQ and put notification jobs and expiry jobs in queues | Events are put in a line and handled separately, a job can be delayed by 10 minutes, and a failed job can be retried | Not considered | Chosen: same reason as above |

## 3. Decision

The system has 2 queues: one for expired orders (`order-expiry-queue`) and one for notifications (`notification_queue`).

When a bank transfer order is created, the system emits the event `ORDER_CREATED`. The listener in the notification module catches this event and adds the job `cancel-expired-order` to `order-expiry-queue`. After 10 minutes, the processor of `order-expiry-queue` receives the job, checks the order status, cancels the order with the reason "expired", and emits the event `ORDER_CANCELLED_AUTO`. For a CASH order, the listener does not create an expiry job. It calls Telegram directly, not through a queue.

For the notification queue: when the event `PAYMENT_SUCCESS` arrives, the listener adds the job `order.paid` to `notification_queue`. The processor receives the job and sends a Telegram message that the order has been paid. For the event `ORDER_CANCELLED_AUTO`, the processor already has a branch for an auto-cancel notification, but nothing adds a job for that branch, and the listener of this event only writes a log, because I do not want too many notifications.

Settings of `notification_queue`: `attempts: 3`, `backoff: { type: 'exponential', delay: 2000 }`, `removeOnComplete: true`.
Settings of `order-expiry-queue`: `delay: 10 * 60 * 1000` (10 minutes), `attempts: 5`, `backoff: { type: 'exponential', delay: 2000 }`, `removeOnComplete: true`.

For the notification queue, I allow a retry when an error happens, up to 3 times for each notification. For the expiry queue, the job runs after 10 minutes and is retried up to 5 times when an error happens. Both kinds of job are removed from Redis after they finish, and the first retry wait is 2 seconds, which gets longer after each retry.

I chose BullMQ because I wanted jobs to be handled separately, to be delayed by 10 minutes, and especially to be retried when an error happens. Later, I noticed a weakness: if Redis goes down, the jobs are lost too. So I added a cron job that runs every minute. It cancels `NEW` orders that are past the deadline, to fill the gap when Redis has a problem.

## 4. Trade-offs & Limitations

One order with many items creates many expiry jobs, because there is no `jobId`. The first job cancels the order and releases the slots for all items. The later jobs see that the order is already `CANCELLED`, get a `BadRequestException`, and are skipped. So slots are not released twice, but there are extra jobs.

The two cancel paths are different in locks and transactions. The BullMQ job uses `FOR UPDATE` on `Order`, and then on each `Availability` row. Cancelling the order, updating `paymentLink`, and releasing slots are in one transaction. The cron does not lock `Order`. It uses `updateMany` with the conditions `id` and `status: NEW`. Cancelling the order and updating `paymentLink` are in one transaction, but releasing slots happens after the commit, outside the transaction, and each release opens its own transaction. If releasing a slot fails in the cron, the order is already `CANCELLED` but the slot is not released. It is not retried, and there is only a log. Even though the cron runs every minute, the order is `CANCELLED` instead of `NEW`, so it is skipped and the slot is never released.

Redis is not durable. When the container is recreated, the waiting jobs and the old hold keys are lost. Because the cron runs every minute, even if Redis has a problem, cancelling expired bank transfer orders and releasing slots still continue. What is lost is the waiting notification jobs.

When adding the expiry job fails (for example, when Redis has an error), the error is only written to a log. The order is already committed but has no expiry job. Only the cron can cancel that order.

The notification processor catches Telegram errors and only writes a log. It does not throw the error again. So `attempts` of the notification job has no effect, and the message is not sent. Also, the call to Telegram has no timeout, so if Telegram does not respond, the job may hang (inferred from the code, not tested).

The system has no monitoring for failed jobs (no Bull Board or alert). There are tests for the cron. I did not find tests for the listener and the two processors.

If the payment webhook arrives close to 10 minutes or after 10 minutes, when the cron or the worker has already changed the order to `CANCELLED`, the webhook will update the cancelled order back to `PROCESSING`. This breaks the rule about the final state of a cancelled order (see ADR-004).

## 5. What I'd do differently

The directions below have not been implemented in the code.

1. **Extra expiry jobs.** Add only one job for each order, instead of one job for each item, and set `jobId` from `orderId` so BullMQ does not accept a duplicate job. The change is in the listener of `ORDER_CREATED`. This idea came to me after I read the code again; I did not consider it when I wrote the code.

2. **The cron releases slots outside the transaction.** Make the cron call the same `cancelOrderById` function that the worker uses, so that cancelling the order, updating `paymentLink`, and releasing slots are in one transaction. Then, if releasing a slot fails, the whole transaction rolls back, the order stays `NEW`, and the next scan tries again. The change is in `PaymentCronService`. I would still keep both the BullMQ job and the cron: the job cancels at the exact 10-minute mark and emits an event, and the cron is a safety net when Redis loses the job. Without the cron, nothing can save an order when its job is lost. This is also an idea that came to me later.

3. **Redis is not durable.** Turn on persistence for Redis (`appendonly`) and mount a volume for the Redis container in `docker-compose.production.yml`, so a restart does not lose the waiting jobs. This is the same direction as in ADR-010.

4. **Adding the job fails with only a log.** At the current scale, I accept this. Bank transfer orders are still cancelled by the cron, so the risk is low. I would only raise the log level to `error` and write the `orderId`, so I can find the order later.

5. **Telegram errors are swallowed and there is no timeout.** In the processor, throw again the network errors or errors from Telegram so that `attempts` has an effect. A missing token or chat ID does not need a retry. Set a clear timeout for `HttpService`, so a hanging call does not hold a worker slot. The change is in `NotificationProcessor` and `HttpModule`. This is an idea that came to me later.

6. **Monitoring and tests.** Add tests for the listener and the expiry processor, because they decide what happens to the order. For a dashboard or alerts for failed jobs, I accept not having them at the scale of one shop and 109 orders in a season.

7. **The webhook overwrites a cancelled order.** The fix, and the decision whether to fix it or only record the current state, belong to ADR-004. In this document I only record the limitation.