# Delta Exchange Crypto Algorithmic Trading

Cross-venue execution between a crypto perpetual exchange and an MT5 broker — contract
normalisation, inverse-contract delta, and two-leg execution that survives the second leg
failing.

---

## delta-exchange-mt5-hedging — the hedging platform

Holds a position in a Delta-Exchange-style **perpetual** and the economically offsetting
position at an Exness-style **MT5 broker**, handling the fact that the two venues agree on
almost nothing about what a position is: contracts versus lots, linear versus inverse
settlement, funding versus swap points, and quote currencies that do not match the account
currency.

The problem it exists to solve, computed by the engine rather than asserted:

```
1 BTCUSDT-PERP contract (= 0.001 BTC)  = 0.001    BTCUSD lot (1 lot = 1 BTC)
1 ETHUSDT-PERP contract (= 0.01 ETH)   = 0.001    ETHUSD lot (1 lot = 10 ETH)
1 PAXGUSDT-PERP contract (= 0.01 PAXG) = 0.0001   XAUUSD lot (1 lot = 100 XAU)
1 SOLUSDT-PERP contract (= 1 SOL)      = 0.01     SOLUSD lot (1 lot = 100 SOL)
```

Four pairs, four ratios. Hedging 1000 contracts with "1000 lots" is a 1000× position error.
Note also that BTC and ETH share a ratio of 0.001 despite unrelated contract sizes — equal
ratios do not mean equal economics.

What is in it:

- **Nine hedge objectives** — base-asset-, notional-, quote-P&L- and account-currency-P&L-
  neutral; custom ratio; partial; funding- and cost-adjusted (mean-variance,
  `h* = β − k/(λσ_h²)`); risk-weighted. On an inverse perpetual in profit,
  `BASE_ASSET_NEUTRAL` and `QUOTE_PNL_NEUTRAL` differ by 25% on the same position, because
  an inverse contract's delta depends on its entry price rather than its mark.
- **A quantity optimiser** that searches the venue's lattice instead of rounding — on a live
  example, 10.03 bps of residual where naive rounding leaves 16.73 bps.
- **Measured statistics** — rolling volatility and correlation recovering the simulator's
  configured values within 4%, with β reported alongside its standard error and flagged when
  its distance from 1.0 is within sampling noise.
- **A 16-state execution machine** with write-ahead transition persistence, so restart
  recovery can distinguish "submitted, outcome unknown" from "never submitted".
- **Fourteen injectable faults and ten end-to-end scenarios** — leg-2 rejection, API timeout,
  partial fills, venue disconnects, stale data, price gaps, restart recovery.
- **Two-level risk** with closed-form linear *and* inverse liquidation, and a kill switch
  that verifies it actually reached flat rather than assuming it did.

**Worth knowing before cloning:** it is **paper trading only** and has never traded real
money. That is enforced structurally rather than by a flag — the live adapters raise on
construction, the venue factory refuses any non-paper mode, no setting holds an exchange
credential, and fills can only be created in one file. Five barriers, each covered by a test.
The instrument specifications are illustrative, modelled on the shape of public contract
specs rather than transcribed from live venue documentation, and the volatility estimator is
validated against the simulator's known answer rather than a real market.

Verified from a fresh clone: 402 tests, 25/25 acceptance steps, 10/10 scenarios, strict mypy
clean, 21 tables migrated on PostgreSQL 16.

https://github.com/pranay123-stack/delta-exchange-mt5-hedging

---

## Strategy, not execution

This portfolio is about the plumbing between two venues. The strategies that would run on it
live elsewhere:

| Portfolio | What it covers |
|---|---|
| [Crypto Trading Strategies](https://github.com/pranay123-stack/crypto-trading-strategies) | Statistical arbitrage, ICT smart-money, mean reversion, market making |
| [Crypto Exchange Development](https://github.com/pranay123-stack/crypto-exchange-development) | Order books, matching engines and venue simulators |
| [Forex Trading Strategies](https://github.com/pranay123-stack/forex-trading-strategies) | MQL5 Expert Advisors — the MT5 side, as a strategy platform rather than a hedge leg |

---

## On results

The backtest figures in this portfolio are simulation output, not a track record. A 500-step
run (≈21 simulated days) shows a full hedge removing 98.8% of price drawdown at a cost of
−5,413, a half hedge removing 49.7%, and no hedge removing 1.0% — risk reduction tracking the
hedge ratio linearly, which is the correctness check rather than a performance claim.

The hedge **costs money**. That is what a hedge does, and the platform reports it rather than
dressing it up. No live-traded track record is published here, because there is none.

**Tech:** Python 3.12, FastAPI, SQLAlchemy 2.0 async, PostgreSQL, Alembic, React, TypeScript,
Docker

---
