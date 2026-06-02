# BULK Exchange — SDK and Signing Reference

Community-maintained reference. For official documentation: [docs.bulk.trade](https://docs.bulk.trade)

> **Note:** The public REST API endpoints are not yet documented publicly. This doc covers the signing library (bulk-keychain) which is official and open source.

---

## bulk-keychain — Official Signing Library

GitHub: [github.com/bulk-trade/bulk-keychain](https://github.com/bulk-trade/bulk-keychain)

Available in:

| Language | Package |
|----------|---------|
| Python | `pip install bulk-keychain` |
| Node.js | `npm install bulk-keychain` |
| Browser/WASM | `npm install bulk-keychain-wasm` |
| Rust | `cargo add bulk-keychain` |

---

## Confirmed Signing Methods

From the official bulk-keychain README:

| Method | Description |
|--------|-------------|
| `sign(order)` | Sign a single order or cancel |
| `signAll([orders])` | Sign multiple orders as parallel transactions |
| `signGroup([orders])` | Sign multiple orders as one atomic transaction |
| `signOraclePrices([...])` | Sign oracle price updates |
| `signPythOracle([...])` | Sign Pyth oracle batch |

---

## External Wallet Flow (Confirmed)

For hardware wallets or browser wallets that can't expose private keys, bulk-keychain supports a Prepare → Sign → Finalize flow:

1. **Prepare** — creates an unsigned message without needing the private key
2. **Sign** — external wallet signs the prepared message bytes
3. **Finalize** — wraps the external signature into a SignedTransaction

Supported actions: orders, batches, agent wallets, faucet requests, user settings updates.

---

## Order ID Formula (Confirmed from bulk-keychain source)

```
order_id = base58( SHA256( seqno_le + bincode(single_action) + account_bytes + nonce_le ) )
```

- `seqno`: action index within a grouped transaction (`0` for single orders)
- `bincode(single_action)`: canonical binary encoding of the action
- `account_bytes`: 32-byte account public key
- `nonce_le`: u64 little-endian
- Signer pubkey is **excluded** from the hash

**Fixed-point serialization:** `round(value * 1e8)` as `u64` little-endian

---

## Confirmed Architecture Facts

| Property | Value | Source |
|----------|-------|--------|
| Consensus | BulkBFT — leaderless BFT | Official docs |
| Matching latency target | 5–20ms | Official docs |
| Fair ordering | Fisher-Yates shuffle via WyRand PRNG | Official docs |
| Margin model | SPAN-style portfolio margin | Official docs |
| Capital efficiency | 30–70% lower requirement vs isolated (varies by position correlation) | Official docs |
| Sub-accounts per master | 64 | Official docs |
| Signature scheme | Ed25519 | bulk-keychain source |

---

## What Is Not Yet Public

- REST API base URL and endpoint paths
- WebSocket stream format and URLs
- Full SDK client (bulk-keychain is a signing library only — not a full client)
- Mainnet trading API access (pre-mainnet as of June 2026)

Check [docs.bulk.trade](https://docs.bulk.trade) and the [official Discord](https://discord.gg/bulk) for updates.

---

*Resource maintained by [builtonbulk.xyz](https://builtonbulk.xyz)*
