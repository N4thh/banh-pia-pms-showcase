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
| 001 | Concurrent double-booking on slot reservation | Pessimistic locking + 3% overbooking buffer | Applied | [VI](./001-concurrency-control-vi.md) · [EN](./001-concurrency-control-en.md) |
