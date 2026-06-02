# Fair Ordering — BULK Exchange 4-Layer MEV Shield

BULK Exchange publishes a formal fair ordering specification. This document summarises the four layers as described in official documentation.

> **Source:** [docs.bulk.trade/architecture/fair-ordering](https://docs.bulk.trade). All layer descriptions are from official BULK Exchange documentation.

---

## Layer 1 — Quorum Admission

The transaction batch is formed from the intersection of pending transaction sets across a supermajority (>2/3) of validators, using minisketch set reconciliation.

A transaction only enters a batch if it appears across the required validator majority. No single validator can selectively include or exclude a transaction.

---

## Layer 2 — Deterministic Shuffle

All transactions in the batch are reordered using the Fisher-Yates algorithm with a WyRand PRNG seeded by the consensus-agreed batch timestamp.

The batch timestamp is only finalised after BFT consensus completes — after all orders are submitted. No participant can predict the shuffle outcome before submitting their order.

This layer eliminates sandwich attacks and latency-based front-running that depend on knowing the final transaction sequence in advance.

---

## Layer 3 — Structural Priority Queues

Within the shuffled batch, transactions are processed in fixed order:

1. Cancellations
2. Post-only / ALO (maker) orders
3. Regular orders

This ensures cancellations always execute before fills, preventing cancel-racing. ALO orders are processed before regular orders so market makers cannot be unintentionally filled as taker.

---

## Layer 4 — Price-Time Priority

Standard CLOB matching within the ordered set. Buy orders sorted descending by limit price; sell orders ascending. Ties broken by submission time within the shuffled batch.

---

## Comparison

| Property | BULK Exchange | Hyperliquid |
|----------|:------------:|:-----------:|
| Fair ordering specification | Published (4-layer) | Not published |
| Consensus | Leaderless BulkBFT | Leader-based HyperBFT |
| Deterministic shuffle | Fisher-Yates / WyRand | Not specified |
| Quorum admission | Yes (>2/3 supermajority) | Not specified |
| Cancel priority | Yes (Layer 3) | Not specified |

This comparison reflects published documentation as of June 2026. Hyperliquid has not published a comparable fair-ordering specification.

---

*Resource: [builtonbulk.xyz/fair-ordering-explained](https://builtonbulk.xyz/fair-ordering-explained)*
