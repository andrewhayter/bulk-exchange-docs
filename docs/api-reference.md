# BULK Exchange API Reference

Community reference compiled from official sources:
- [docs.bulk.trade/api-reference](https://docs.bulk.trade/api-reference/introduction)
- [github.com/Bulk-trade/bulk-keychain](https://github.com/Bulk-trade/bulk-keychain)
- [github.com/Bulk-trade/bulk-client](https://github.com/Bulk-trade/bulk-client)

> Community-maintained. Not official documentation.

---

## Base URLs

| Environment | HTTP REST | WebSocket |
|-------------|-----------|-----------|
| Staging | `https://staging-api.bulk.trade/api/v1` | `wss://staging-ws.bulk.trade` |
| Production | `https://exchange-api.bulk.trade/api/v1` | `wss://exchange-ws1.bulk.trade` |

> Production endpoints may be paused during trading competitions. Check [docs.bulk.trade](https://docs.bulk.trade) for current status.

---

## Authentication

| Endpoint category | Auth required |
|-------------------|---------------|
| Market Data (HTTP) | None |
| Account queries (HTTP) | None |
| Trading (HTTP) | Ed25519 signature via [bulk-keychain](https://github.com/Bulk-trade/bulk-keychain) |
| WebSocket market data | None |
| WebSocket account stream | None (public key in subscription) |

---

## Signing Library

Official packages:

```bash
npm install bulk-keychain        # TypeScript / Node.js
npm install bulk-keychain-wasm   # TypeScript / Browser
pip install bulk-keychain        # Python
cargo add bulk-keychain          # Rust
```

Source: [github.com/Bulk-trade/bulk-keychain](https://github.com/Bulk-trade/bulk-keychain)

### Nonce

Nonces are nanosecond-precision integers:

```typescript
const nonce = BigInt(Date.now()) * 1_000_000n;
```

### Signing message format

```
action_count (u64 LE) + serialized_actions + nonce (u64 LE) + account_pubkey (32 bytes)
```

The `signer` and `signature` fields are **not** included in the signed bytes.

### Signature modes (request header)

```
x-bulk-sig-mode: raw       # canonical binary (default)
x-bulk-sig-mode: offchain  # Solana v0 offchain-message envelope
x-bulk-sig-mode: base58    # legacy wallet compatibility
```

### Agent wallets

When using an agent wallet: `account` = user pubkey, `signer` = agent pubkey. The agent must be pre-authorized via `agentWalletCreation`.

### Field encoding

| Field type | Encoding |
|-----------|---------|
| Price, size, trigger price (`px`, `sz`, `tr`, `pmin`, `pmax`, `lim`) | `round(value × 1e8)` as u64 LE |
| Raw float fields (`mod.sz`, `faucet.amount`, `transfer.marginAmount`) | IEEE-754 double, 8 bytes LE |
| Enum variants | u32 LE discriminant |
| Pubkeys / hashes | 32 raw bytes |
| Strings | u64 LE length + UTF-8 bytes |

---

## Market Data (HTTP — No Auth)

### GET /exchangeInfo

Returns all available markets.

**Response:**
```json
[
  {
    "symbol": "BTC-USD",
    "baseAsset": "BTC",
    "quoteAsset": "USDC",
    "status": "TRADING",
    "pricePrecision": 1,
    "sizePrecision": 4,
    "tickSize": 0.1,
    "lotSize": 0.0001,
    "minNotional": 10,
    "maxLeverage": 50,
    "orderTypes": ["LIMIT","MARKET","STOP","STOP_LIMIT","TAKE_PROFIT","RANGE","TRIGGER","TRAILING"],
    "timeInForces": ["GTC","IOC","ALO"]
  }
]
```

---

### GET /ticker/{symbol}

24-hour stats for one market.

**Path parameter:** `symbol` e.g. `BTC-USD`

**Response fields:**
```
symbol, priceChange, priceChangePercent, lastPrice, highPrice, lowPrice,
volume, quoteVolume, markPrice, oraclePrice, openInterest, fundingRate,
regime (-1|0|1), regimeDt, regimeVol, regimeMv,
fairBookPx, fairVol, fairBias,
timestamp (nanoseconds)
```

---

### GET /klines

OHLCV candle data.

**Query parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `symbol` | Yes | e.g. `BTC-USD` |
| `interval` | Yes | `10s\|1m\|3m\|5m\|15m\|30m\|1h\|2h\|4h\|6h\|8h\|12h\|1d\|3d\|1w\|1M` |
| `startTime` | No | int64 milliseconds |
| `endTime` | No | int64 milliseconds |

**Response:**
```json
[
  {
    "t": 1717300800000,
    "T": 1717300860000,
    "o": 68000.0,
    "h": 68200.0,
    "l": 67900.0,
    "c": 68100.0,
    "v": 12.5,
    "n": 143
  }
]
```

---

### GET /l2book

L2 order book snapshot.

**Query parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `type` | Yes | Must be `"l2book"` |
| `coin` | Yes | Market symbol e.g. `BTC-USD` |
| `nlevels` | No | Price levels per side (integer) |
| `aggregation` | No | Price increment in quote currency |

**Response:**
```json
{
  "updateType": "snapshot",
  "symbol": "BTC-USD",
  "levels": [
    [{ "px": 68000.0, "sz": 1.5, "n": 3 }],
    [{ "px": 68100.0, "sz": 0.8, "n": 2 }]
  ],
  "timestamp": 1717300800000
}
```

---

### GET /stats

Exchange-wide statistics.

**Query parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `period` | No | `1d\|7d\|30d\|90d\|1y\|all` (default `1d`) |
| `symbol` | No | Omit for aggregate |

**Response:**
```json
{
  "timestamp": 1717300800000,
  "period": "1d",
  "volume": { "totalUsd": 45000000 },
  "openInterest": { "totalUsd": 12000000 },
  "funding": {
    "rates": {
      "BTC-USD": { "current": 0.0001, "annualized": 0.1095 }
    }
  },
  "markets": [
    {
      "symbol": "BTC-USD",
      "volume": 30000000,
      "quoteVolume": 30000000,
      "openInterest": 8000000,
      "fundingRate": 0.0001,
      "fundingRateAnnualized": 0.1095,
      "lastPrice": 68100.0,
      "markPrice": 68050.0
    }
  ]
}
```

> Note: All periods return 24h rolling stats. Aggregate response is cached with a 600s TTL. Markets sorted by `quoteVolume` descending.

---

### GET /riskSurfaces

Risk surface configuration for a market.

**Query parameter:** `market` — market or coin symbol (e.g. `BTC-USD` or `BTC`)

---

### GET /feeState

Current fee policy state. No parameters. No auth required.

---

## Account Queries (HTTP — No Auth)

### POST /account

Read-only account data. No signing required.

**Request body:**
```json
{
  "type": "fullAccount|openOrders|fills|positions|fundingHistory|orderHistory|activityHistory|riskHistory|feeTier",
  "user": "<base58_pubkey>",
  "symbol": "<optional, for feeTier type only>"
}
```

**Type options:**

| Type | Returns |
|------|---------|
| `fullAccount` | Complete account state — margin, positions, orders, leverage |
| `openOrders` | Up to 5,000 resting live orders |
| `fills` | Up to 5,000 recent trade fills |
| `positions` | Up to 5,000 closed position history with P&L |
| `fundingHistory` | Up to 5,000 funding payments |
| `orderHistory` | Up to 5,000 terminal order records |
| `activityHistory` | Up to 5,000 account activity events |
| `riskHistory` | Up to 5,000 liquidation / ADL events |
| `feeTier` | Fee tier snapshot for a symbol |

---

## Trading (HTTP — Signed)

### POST /order

Place, cancel, or modify orders. Requires Ed25519 signature.

**Request body:**
```json
{
  "actions": [ ... ],
  "nonce": 1717300800000000000,
  "account": "<base58_user_pubkey>",
  "signer": "<base58_signer_pubkey>",
  "signature": "<base58_ed25519_signature>"
}
```

### Action types

**Limit order:**
```json
{ "l": { "c": "BTC-USD", "b": true, "px": 68000, "sz": 0.1, "tif": "GTC", "r": false, "i": false } }
```
- `c` = symbol, `b` = isBuy, `px` = price, `sz` = size, `tif` = GTC|IOC|ALO, `r` = reduceOnly, `i` = isolated

**Market order:**
```json
{ "m": { "c": "BTC-USD", "b": true, "sz": 0.1, "r": false, "i": false } }
```

**Cancel single order:**
```json
{ "cx": { "c": "BTC-USD", "oid": "<order_id_base58>" } }
```

**Cancel all orders:**
```json
{ "cxa": { "c": "BTC-USD" } }
```

**Conditional types:** `st` (stop-loss), `tp` (take-profit), `rng` (OCO range), `trl` (trailing stop), `trig` (trigger basket), `of` (on-fill)

### Response

```json
{
  "status": "ok",
  "response": {
    "type": "order",
    "data": {
      "statuses": [
        { "resting": { "oid": "<base58_hash>" } }
      ]
    }
  }
}
```

**Status variants:**

| Non-terminal | Terminal |
|-------------|---------|
| `resting` | `filled` |
| `working` | `partiallyFilled` |
| `triggered` | `cancelled`, `cancelledRiskLimit`, `cancelledSelfCrossing`, `cancelledReduceOnly`, `cancelledIoc` |
| | `rejectedCrossing`, `rejectedDuplicate`, `rejectedRiskLimit`, `rejectedInvalid` |
| | `siblingCancelled`, `triggerFailed`, `error` |

---

## WebSocket API

**Production URL:** `wss://exchange-ws1.bulk.trade`
**Staging URL:** `wss://staging-ws.bulk.trade`

**Limits:** Max 100 subscriptions per connection, 1,000 messages/second. Server pings every 30 seconds — respond with pong within 10 seconds.

### Subscribe

```json
{
  "method": "subscribe",
  "subscription": [
    { "type": "ticker", "symbol": "BTC-USD" },
    { "type": "candle", "symbol": "BTC-USD", "interval": "1m" },
    { "type": "trades", "symbol": "BTC-USD" },
    { "type": "l2Snapshot", "symbol": "BTC-USD", "nlevels": 10 },
    { "type": "l2Delta", "symbol": "BTC-USD" },
    { "type": "account", "user": "<base58_pubkey>" }
  ]
}
```

### Unsubscribe

```json
{ "method": "unsubscribe", "topic": "ticker.BTC-USD" }
```

### Update frequencies

| Stream | Frequency |
|--------|-----------|
| `ticker` | 200ms |
| `candle` | ~10–20s |
| `trades` | Real-time |
| `l2Snapshot` | 200ms |
| `l2Delta` | Real-time |
| `account` | Event-driven |

### Account stream

Subscribe with one or multiple pubkeys:
```json
{
  "method": "subscribe",
  "subscription": [{
    "type": "account",
    "user": ["<pubkey1>", "<pubkey2>"]
  }]
}
```

First message is a full `accountSnapshot`. Subsequent messages are delta updates.

---

## Order IDs

Order IDs are computed client-side. bulk-keychain computes them automatically:

```typescript
const signed = signer.sign(order);
console.log(signed.orderId);
```
```python
signed = signer.sign(order)
print(signed.get("order_id"))
```
```rust
let signed = signer.sign(order.into(), None)?;
println!("{:?}", signed.order_id);
```

**Formula:**
```
base58( SHA256( seqno_le_u64 + bincode(single_action) + account_bytes_32 + nonce_le_u64 ) )
```
- `seqno`: zero-based action index within the transaction (0 for single orders)
- Signer pubkey is excluded from the hash

---

*Resource maintained by [builtonbulk.xyz](https://builtonbulk.xyz)*
