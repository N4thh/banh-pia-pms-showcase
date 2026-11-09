# ADR-010: JWT Rotation & Redis Blacklist

**Date:** 02/11/2026
**Status:** Applied

## 1. Context

Some functions and data in the system can only be used and seen by the admin. For example: adding cake slots, viewing all orders, viewing revenue details, changing the order status, and viewing weekly revenue reports.

The information of the shop and the customers is the main thing to protect. We also need to make sure that only a user with the admin role can do important tasks. So the system needs authentication and authorization for the people who access it.

The admin logs in with a username and a password. The password is stored as a hash in the database. At login, the system uses the `compare` function of `bcrypt` to check the password against the hash, and the username must match too. If both match, the system gives out tokens. After that, admin requests are checked with the token (see section 3), not with the password again.

## 2. Options Considered

When I wrote the code, I did not consider any other option. I used JWT to verify the identity of the user, stored in `localStorage`, together with a blacklist stored in Redis.

## 3. Decision

The system issues an Access Token (AT) and a Refresh Token (RT). The AT lives for 15 minutes and the RT lives for 7 days. The AT works like a "ticket" to do admin tasks. The RT is used to get a new AT when the AT expires.

An admin request is checked in this order: first, whether the endpoint is `public`; if not, the signature and expiry of the AT are verified; then the `jti` of the AT is checked against the blacklist; finally `AdminGuard` checks whether the endpoint requires admin (`IS_ADMIN_KEY`) and whether `role` is `ADMIN`. Only when all checks pass does the business function run.

When the admin logs out, the AT and the RT (if sent) are added to the blacklist in Redis, with the key `blacklist:${jti}` and a TTL equal to the remaining time of the token. When tokens are refreshed, the system issues a new AT and RT pair, and the old RT is added to the blacklist.

The FE stores the tokens in `localStorage`. When it receives a `401` error, the FE tries to refresh the token with the RT. If that fails, it removes the tokens from `localStorage`.

## 4. Trade-offs & Limitations

After a token refresh, I only add the old RT to the blacklist, not the old AT. As a result, the old AT can still be used to access the system while it is valid, for up to 15 minutes.

If 2 refresh requests with the same RT arrive at almost the same time, and the token is still valid, both of them read that the token is not in the blacklist, so both pass. As a result, both create a new token pair, and one pair is extra.

When a used RT is sent again, the system finds its `jti` in the blacklist and returns an error right away. This can happen because someone reuses an old RT (the RT may have leaked), but it can also happen because of two requests at the same time or a client retry. With the current code, the system cannot tell these cases apart.

When Redis cannot be reached during the permission check or the token refresh, the request cannot continue because the blacklist cannot be read. There is no try/catch for this case, so the error is not handled the way I want. The final response that the client receives has not been tested.

The blacklist data is not stored durably: the Compose files do not declare a volume or any persistence setting for Redis. Inferred from the configuration (not verified): if Redis loses its data, tokens that were logged out can be used again until they expire (AT up to 15 minutes, RT up to 7 days).

When an expired RT, or an RT with a wrong signature, is sent to the refresh endpoint, `verifyAsync` is outside a try/catch, and the code does not turn the error into an `UnauthorizedException`. Inferred from the code (not verified): the client receives `500` instead of `401`.

Tokens are stored in `localStorage`, and the repo has no CSP or security header configuration. If the admin page is injected with a malicious script, the script can read the tokens.

About login and authorization: the login route has no limit on password attempts; `MinLength(6)` in `AdminDto` is not enforced because this route has no `ValidationPipe`; and `AdminGuard` lets a request pass when an endpoint has no `IS_ADMIN_KEY`, so access control depends on whether each route is marked.

During an authorization review, I found that three cake routes and `/availability/book` had no effective guard: putting a guard in a service does not protect its routes. I moved the guard to the affected routes. Initially, 12 of the 14 test cases failed; after the fix, all 14 passed.

The tests still only have `should be defined` for `AuthService`, `AuthController` and `RedisService`. `JwtBlacklistGruard` is tested with a public handler, a request with no token (`401`), a wrong role (`403`), and a blacklisted token (`401`). There is no test for login, refresh, logout or Redis errors.

## 5. What I'd do differently

The guard-placement issue noted in section 4 has been fixed. The directions below have not been implemented in the code.

1. **The old AT still works after a refresh.** Send the AT with the refresh request, or read the `jti` of the AT from the `Authorization` header, and add that `jti` to the blacklist together with the old RT. The change is in the `/auth/refresh` endpoint and `AuthService`. The cost is one more Redis write for each refresh.

2. **Two refresh requests at the same time.** Merge the read step and the write step of the blacklist into one Redis command (`SET ... EX ... NX`). If the command says the key already exists, reject the request. The change is in `RedisService` and the `refreshToken` function. This idea came to me after I read the code again; I did not consider it when I wrote the code.

3. **A used RT is sent again.** Write a warning log when this happens, so I have data to tell the causes apart. I would not revoke the whole token chain automatically, because with one admin account and one shop, locking the admin out by mistake because of two simultaneous requests is worse than the benefit. The change is in `AuthService`.

4. **The blacklist is not durable.** Turn on persistence for Redis (`appendonly`) and mount a volume for the Redis container in `docker-compose.production.yml`, so a restart does not lose the revocation list.

5. **Redis errors.** For admin routes, keep blocking the request when the blacklist cannot be read. These routes hold customer data and there is only one admin, so a short outage is acceptable. Add try/catch to return a clear error (for example `503`) instead of an uncontrolled error. The change is in `JwtBlacklistGruard` and `AuthService`. This is different from the "let it pass" direction in ADR-009, because the data here is more sensitive.

6. **Error code for an expired or badly signed RT.** Wrap `verifyAsync` in a try/catch and turn the error into an `UnauthorizedException`, so the client receives `401`. The change is in the `refreshToken` function. Before that, I would run a test on the local environment to confirm the real error code.

7. **Tokens in `localStorage`.** At the current scale (one admin account, used from the shop owner's device), I accept this. If the system grows, I would add a CSP in Nginx and consider storing the RT in an `HttpOnly` cookie. This idea came to me after I read the code again.