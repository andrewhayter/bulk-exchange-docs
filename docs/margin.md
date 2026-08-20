# Portfolio Margin

BULK Exchange uses portfolio margin by default: the risk engine evaluates the entire account as one risk unit instead of margining each position independently. Correlated hedges partially offset each other, up to a documented 70% efficiency gain over per-position margin.

> **Source:** [docs.bulk.trade](https://docs.bulk.trade) and BULK's "The Engine Room" post (July 16, 2026), the first public disclosure of the aggregation formula.

---

## The 9-Regime HMM

A Hidden Markov Model classifies each market into one of 9 regimes: 3 market regimes (bearish / neutral / bullish) × 3 volatility regimes (low / medium / high). Each regime maps to its own lambda surface — a 3D function of leverage, market impact, and volatility regime — via bilinear interpolation, so margin requirements change continuously with no discrete tier-jump cliffs.

---

## The Aggregation Formula

Confirmed SPAN-inspired (CME's 1988 methodology, the same lineage OKX and Bybit use for portfolio margin, gated behind VIP tiers there — BULK runs it for every account from the first dollar).

**Step 1 — standalone margin per position:** `Mi = notional × rate`

**Step 2 — correlation-weighted aggregation, not addition:**

```
Mp = √( ΣMi² + ΣΣ 2·Mi·Mj·ρij )
```

`ρ` is the pairwise correlation between positions. Correlated same-direction positions reinforce; correlated opposite-direction positions partially cancel. Summing margins assumes every position blows up in the same direction at once — this formula weights that by how likely it actually is.

**Worked example — $100K long BTC / $100K short ETH, ρ = 0.81:**

| Step | Value |
| --- | --- |
| BTC leg margin (2% floor) | $2,004 |
| ETH leg margin (2% floor) | $2,001 |
| Per-position sum | $4,005 |
| Portfolio margin: `√(2,004² + 2,001² − 2×2,004×2,001×0.81)` | **$1,234** |
| Reduction | **69%** |

**Regime-break protection:** measured correlation is blended with a worst-case correlation, so the offset a hedged book gets is the one that survives a correlation breakdown in a crisis, not the one from a calm month.

Margin rates (the 2% floor above, hard caps, leverage limits) are model outputs — a function of volatility regime, portfolio leverage, and depth-adjusted cost of force-unwinding into the live book — not a static percentage table.

---

## Cascade Adjustment

Before flagging a liquidation, the engine estimates the price impact of liquidating the position and recalculates whether the portfolio survives the cascade. If not, the engine may act earlier to prevent it. Liquidations are netted across accounts before hitting the book, slippage-capped, with the insurance vault and ADL as backstops in that order.

---

## Isolated Margin

Add `i=true` to any order (see [api-reference.md](./api-reference.md)) to route it to a per-instrument isolated account instead of the main portfolio. One instrument per isolated account, created automatically on first use. Maximum loss is capped at the capital in that isolated account.

---

## Sub-Account Isolation

Each of up to 64 sub-accounts runs a fully independent portfolio margin evaluation — a liquidation in one sub-account doesn't touch the others.

---

_Resource: [builtonbulk.xyz/bulk-margin-types](https://builtonbulk.xyz/bulk-margin-types)_
