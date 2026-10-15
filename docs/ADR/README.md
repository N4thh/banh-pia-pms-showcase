# Architecture Decision Records — Banh Pia PMS

This folder documents the key engineering decisions made while building and
operating Banh Pía PMS in production — including the trade-offs considered,
what was rejected, and why. Written primarily in Vietnamese (my working
language during development); English versions are provided for the most
significant decisions.

## Format

Each ADR follows a fixed structure: Context → Options Considered →
Decision → Trade-offs & Limitations → What I'd do differently.

## Index

| # | Problem | Decision | Status | Docs |
|---|---|---|---|---|
| 001 | Concurrent double-booking on slot reservation | Pessimistic locking + 3% overbooking buffer | Applied | [VI](./records/001-concurrency-control/001-concurrency-control-vi.md) · [EN](./records/001-concurrency-control/001-concurrency-control-en.md) |
| 002 | PayOS webhook security | Signature verification + idempotent processing | Applied | [VI](./records/002-payos-webhook-security/002-payos-webhook-security-vi.md) · [EN](./records/002-payos-webhook-security/002-payos-webhook-security-en.md) |
| 003 | Atomic multi-entity order creation | DB transaction wraps writes, Redis hold kept outside | Applied | [VI](./records/003-atomic-multi-entity-order/003-atomic-multi-entity-order-vi.md) · [EN](./records/003-atomic-multi-entity-order/003-atomic-multi-entity-order-en.md) |
| 004 | Order cancellation concurrency | Pessimistic locking in a shared transaction | Applied | [VI](./records/004-order-cancellation-concurrency-control/004-order-cancellation-concurrency-control-vi.md) · [EN](./records/004-order-cancellation-concurrency-control/004-order-cancellation-concurrency-control-en.md) |
| 005 | Duplicate order requests | Hashed order fields as a Redis idempotency key with a 10-minute TTL | Applied | [VI](./records/005-idempotency-key-design/005-idempotency-key-design-vi.md) · [EN](./records/005-idempotency-key-design/005-idempotency-key-design-en.md) |
| 006 | Daily production capacity limits | Soft limit and hard limit with a 3% buffer | Applied | [EN](./records/006-capacity-policy/006-capacity-policy-en.md) |