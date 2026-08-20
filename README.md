# BULK Exchange — Developer Resources and Protocol Documentation

> Community-maintained resource. For official documentation, see [docs.bulk.trade](https://docs.bulk.trade).

---

# BULK Exchange Protocol Documentation

**BULK Exchange** is a Solana-native decentralized perpetuals exchange with a custom L0 execution layer (BULK Net) that targets **5–20ms matching latency** and eliminates structural front-running through leaderless BFT consensus.

Settlement and asset custody remain on Solana. Trading executes on BULK Net, a separate consensus environment running alongside the Solana validator set.

---

## Overview

| Property             | Value                                                                                    |
| -------------------- | ---------------------------------------------------------------------------------------- |
| Type                 | Solana-native perpetuals DEX                                                             |
| Model                | Central limit order book (CLOB)                                                          |
| Execution layer      | BULK Net (L0, separate from Solana programs)                                             |
| Consensus            | BulkBFT — leaderless Byzantine fault-tolerant                                            |
| Matching latency     | 5–20ms target (vs ~400ms Solana, ~200ms Hyperliquid)                                     |
| Fair ordering        | 4-layer MEV shield                                                                       |
| Margin               | Portfolio margin — SPAN-inspired, up to 70% lower requirement vs isolated (see [docs/margin.md](./docs/margin.md)) |
| Fee attribution      | Builder Codes — permissionless, user-approved (see [docs/builder-codes.md](./docs/builder-codes.md))     |
| LST                  | BulkSOL — earns 12.5% of all exchange trading fees                                       |
| Community allocation | 30% of total BULK token supply confirmed                                                 |
| Season 1 status      | Live — pre-deposit Aura points at early.bulk.trade/deposit                               |

---

## Architecture

### BULK Net — The L0 Execution Layer

Every BULK validator runs a `bulk-agave` binary alongside their standard Solana (Agave) validator. These share identity keys and Solana stake weights.

```
┌─────────────────────────────────────────────────┐
│                  BULK Validator                  │
│                                                  │
│  ┌─────────────────┐  ┌────────────────────────┐ │
│  │  Agave (Solana) │  │    bulk-agave (BULK)   │ │
│  │  - Settlement   │  │    - Order matching    │ │
│  │  - Custody PDAs │  │    - BulkBFT consensus │ │
│  │  - Withdrawals  │  │    - Fair ordering     │ │
│  └─────────────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### BulkBFT — Leaderless Consensus

BulkBFT is a leaderless Byzantine fault-tolerant protocol with no designated block proposer per round. Key properties:

- Fast path achieves commitment in 2 message delays (theoretical minimum for BFT)
- Requires >2/3 stake supermajority
- No single validator can front-run or censor transactions

### 4-Layer Fair Ordering

| Layer | Mechanism                                            | Attack Prevented                  |
| ----- | ---------------------------------------------------- | --------------------------------- |
| 1     | Quorum admission (>2/3 validators)                   | Single-validator censorship       |
| 2     | Fisher-Yates shuffle (WyRand PRNG, consensus-seeded) | Transaction ordering manipulation |
| 3     | Structural priority queues (cancels first)           | Cancel racing                     |
| 4     | Price-time CLOB matching                             | —                                 |

### Asset Security

All user funds are held in per-user Solana PDAs. Withdrawals require FROST threshold signatures from the validator supermajority. The Solana program upgrade authority is a Squads 3-of-5 multisig.

---

## Quick Start: Route an Order with a Builder Code

Fastest path from zero to a routed order. Full mechanism, SDK details, and seven use-case examples (frontends, bots, vaults, wallet-native perps, etc.) in [docs/builder-codes.md](./docs/builder-codes.md).

```bash
npm install bulk-client
```

```typescript
import { BulkClient } from "bulk-client";

const client = new BulkClient({
  env: "production",
  builderCode: "45uQ6xmDCewvQR2TdMLjGNRTdrzmMo12JuyZFZn8U6hK",
});

await client.approveBuilderFee({ builderCode: "45uQ6xmDCewvQR2TdMLjGNRTdrzmMo12JuyZFZn8U6hK", maxFeeBps: 5 });
await client.placeOrder({ symbol: "BTC-USD", side: "buy", type: "limit", price: 68000, size: 0.1, tif: "GTC" });
```

---

## Key Resources

| Resource               | URL                                                                                              |
| ---------------------- | ------------------------------------------------------------------------------------------------ |
| Protocol overview      | [builtonbulk.xyz/bulk-exchange](https://builtonbulk.xyz/bulk-exchange)                           |
| SDK guide              | [builtonbulk.xyz/bulk-sdk-guide](https://builtonbulk.xyz/bulk-sdk-guide)                         |
| Architecture deep dive | [builtonbulk.xyz/bulk-exchange-architecture](https://builtonbulk.xyz/bulk-exchange-architecture) |
| BulkBFT explained      | [builtonbulk.xyz/bulkbft-explained](https://builtonbulk.xyz/bulkbft-explained)                   |
| Fair ordering          | [builtonbulk.xyz/fair-ordering-explained](https://builtonbulk.xyz/fair-ordering-explained)       |
| Fee structure          | [builtonbulk.xyz/fees](https://builtonbulk.xyz/fees)                                             |
| Aura points guide      | [builtonbulk.xyz/aura-points-guide](https://builtonbulk.xyz/aura-points-guide)                   |
| Official docs          | [docs.bulk.trade](https://docs.bulk.trade)                                                       |
| Pre-deposit (Season 1) | [early.bulk.trade/deposit](https://builtonbulk.xyz/go/bulk-app)                                  |

---

## Documentation Index

- [Aura Points](./docs/aura-points.md) — Season 1 mechanics, earning, dilution
- [BulkSOL](./docs/bulksol.md) — four yield streams, how to get it
- [Matching Engine](./docs/matching-engine.md) — CLOB, priority queues, deterministic execution
- [Fair Ordering](./docs/fair-ordering.md) — MEV shield, Fisher-Yates, quorum admission
- [Portfolio Margin](./docs/margin.md) — 9-regime HMM, SPAN-inspired aggregation formula, worked example
- [Builder Codes](./docs/builder-codes.md) — permissionless fee-attribution for developers, SDK quick start, 7 worked use cases
- [API Reference](./docs/api-reference.md) — all endpoints, order types, signature format, margin system

---

## Official Signing Library

```bash
pip install bulk-keychain    # Python
npm install bulk-keychain    # Node.js
cargo add bulk-keychain       # Rust
```

Official repo: [github.com/bulk-trade/bulk-keychain](https://github.com/bulk-trade/bulk-keychain)

The public REST API endpoints are not yet documented. Check [docs.bulk.trade](https://docs.bulk.trade) for updates.

---

## AURA Tracker Tools

For AURA projection calculators and wallet trackers, see the companion repo:
[bulk-aura-tracker](https://github.com/andrewhayter/bulk-aura-tracker)

---

_Resource maintained by [builtonbulk.xyz](https://builtonbulk.xyz) — community guide site, not official BULK Exchange documentation._
