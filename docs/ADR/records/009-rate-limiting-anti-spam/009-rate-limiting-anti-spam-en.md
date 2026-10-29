# ADR-009: Rate Limiting & Anti-Spam

**Date:** 27/10/2026
**Status:** Partially applied

## 1. Context

The public entry points of the system are:

- **Customer ordering:** this is the only group with a protection mechanism. The risks here are repeated requests, taking up cake slots, and wasting resources.
- **Customer and order lookup:** the risk is that someone looks up users by phone number again and again to get customer information.
- **Payment and login:** creating payment links and admin login. The risks are calling repeatedly to create links on PayOS, and trying passwords again and again.
- **Public data writes:** access logs and slot holds. The risk is unlimited writes to Redis or the database.

Sending too many requests to the server can take up cake slots, waste resources, and lead to wrong final results. So I added a limit to stop users from sending requests continuously.

ADR-005 already covers duplicate order prevention with the Idempotency Key. The mechanism in this ADR reads the IP and builds a key. It counts the requests sent from the same IP in a period of time, and stops accepting requests when the limit is passed.

## 2. Options Considered

When I wrote the code, I did not consider any other option. I built it directly at the application layer with a guard and Redis.

## 3. Decision

I read the IP when a request comes in and use it to build a Redis key. I check whether the key exists. If it does not, the count is 0. When the count is 0, I run the Redis `SET` command with value = 1 and TTL = 60 seconds. My original wish was to let only 1 request run the order creation logic, so I set `MAX_REQUEST = 1`. If the count is more than 1, the guard returns error 429. If the count is equal to 1, I use the Redis `INCR` command to increase the key's value by 1.

When the FE receives error 429, it shows the message returned by the BE: "Bạn thao tác quá nhanh. Vui lòng đợi chúng tôi 1 phút rồi tiếp tục đặt nhé" (You are acting too fast. Please wait 1 minute and then continue ordering).

## 4. Trade-offs & Limitations

When requests arrive one after another within 60 seconds, 2 requests can pass. The first request runs `SET` and sets the TTL. The second request sees that the request count is no longer 0, so it runs `INCR`. When many requests arrive at the same time, they all read the value 0, so they all pass and all run `SET`. Compared with `MAX_REQUEST = 1`, this result is not correct, and this is my mistake.

When the key expires between the two Redis commands (`GET` and `INCR`): after `GET` sees a request count of 1, the code runs `INCR`, but the key has just expired. `INCR` creates a new key with value 1 and no TTL, so the key stays forever unless someone deletes it by hand.

The code cannot handle many customers who share the same IP.

There is no try/catch to delete the key when `createOrder` throws an error (failure, no slots left, etc.). Result: if the user sends the request again with the same `phone`, `items`, `receiveDate` and `paymentMethod` (so the hash gives the same key), the system still returns error 409 with the message "Đơn hàng của bạn đang được xử lý. Vui lòng đợi chúng tôi trong giây lát" (Your order is being processed. Please wait a moment).

## 5. What I'd do differently

Handle the gap between the steps better, so that no key is left hanging, by using try/catch. Make sure only one request can pass, matching the name of the constant `MAX_REQUEST`.

Handle failed requests by using the `DEL` command to delete the key, by catching the error that the BE returns and deleting the key right after.

Find a way to handle many customers who share the same IP.