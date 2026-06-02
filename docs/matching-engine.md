# BULK Exchange Matching Engine

The BULK Exchange matching engine is a deterministic central limit order book (CLOB) running on the BULK Net execution layer.

> **Source:** [docs.bulk.trade](https://docs.bulk.trade). All specifications below are from official documentation unless noted.

---

## Design Principles

**Deterministic execution:** Every validator runs the same matching algorithm against the same input and produces byte-identical output. After execution, validators compare state hashes — any discrepancy immediately flags a faulty or malicious validator.

**Latency target:** 5–20ms within regional validator clusters. Source: docs.bulk.trade.

**No liquidation fees:** Confirmed in official fee schedule.

---

## Priority Queues

Transactions in each batch are processed in fixed priority order:

| Priority | Queue |
|----------|-------|
| 1 (highest) | Config / oracle updates |
| 2 | Liquidations |
| 3 | Cancellations |
| 4 | ALO (post-only / maker) orders |
| 5 | Regular orders |

This ordering is part of Layer 3 of the fair ordering system. Cancellations always execute before new orders in the same batch, preventing cancel-racing attacks. ALO orders are processed before regular orders so market makers are never unintentionally filled as taker.

---

## CLOB Mechanics

Standard price-time priority within each queue. Buy orders sorted descending by limit price; sell orders ascending. Orders at the same price matched first-in-first-out within the shuffled batch.

The Fisher-Yates shuffle (Layer 2 of fair ordering) randomises transaction ordering before the CLOB processes them, using a seed from the consensus-agreed batch timestamp.

---

## Fee Structure

Confirmed from [docs.bulk.trade/bulk-exchange/fees](https://docs.bulk.trade):

| | Genesis Phase (first 30 days) | Post-Genesis |
|--|-------------------------------|-------------|
| Maker (Tier 1) | 0 bps | 2.0 bps |
| Maker (Tier 8) | 0 bps | −2.5 bps rebate |
| Taker (Tier 1) | 3.5 bps | 3.5 bps |
| Taker (Tier 8) | 2.2 bps | 2.2 bps |
| Liquidation fee | None | None |

Fee tiers are based on a rolling 14-day volume window.

**Fee distribution (from official docs):**
- 12.5% of taker fees → validators (accrues to BulkSOL holders)
- 7.5% of taker fees → Alpha Program (qualifying market makers, 30-day epochs)

---

## Order Types

Confirmed supported order types:

**Standard:** Limit (GTC, IOC, ALO), Market, Cancel, Cancel All

**Conditional:** Stop-Loss, Take-Profit, Range/OCO, Trigger Basket, Trailing Stop, On-Fill

---

*Resource: [builtonbulk.xyz/bulk-matching-engine](https://builtonbulk.xyz/bulk-matching-engine)*
