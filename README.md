# Delta Exchange Crypto Algorithmic Trading

Cross-venue execution for crypto derivatives — contract normalisation, inverse-contract delta,
and two-leg execution that survives the second leg failing.

Two platforms, the same problem pointed in opposite directions: **hedging** a perpetual against
an MT5 broker across asset classes, and **arbitraging** perpetuals against each other across
three crypto venues. Both spend most of their code on the same awkward fact — that two venues
agree on almost nothing about what a position is.

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

## perp-funding-basis-arb — the arbitrage platform

The same problem pointed the other way. Instead of hedging one perpetual against a different
asset class, it holds offsetting perpetuals on **two crypto venues** — Binance Futures, Bybit
and OKX — and earns the funding differential and basis dislocation between them.

Same normalisation problem, different disagreements. These venues disagree about how big a
contract is, how long a funding interval is, and what units the interval is published in:

```
                          Binance      Bybit        OKX
BTC contract multiplier   1            1            0.01
SOL lot step              1 (whole)    0.1          1
SOL funding interval      4h           1h           4h
Interval published in     hours        MINUTES      hours
```

100 OKX contracts hedge 1 Binance BTC exactly; matching contract *counts* is a 100× position
error. And a quoted rate of 0.010% is **10.95% APR on an 8-hour venue and 87.60% APR on a
1-hour one** — the same number, eight times the economics. Subtracting raw quoted funding
rates is the standard error in this domain, so the API deliberately exposes
`naive_quoted_difference` — the wrong number — beside the right one.

The thesis is that **a funding differential is not an edge until the cost stack has been
subtracted from it**, which makes the cost model rather than the signal the centre of the
system. What the build then demonstrated, rather than assumed:

- **At retail (VIP0) fees the strategy trades nothing.** A fully-crossed round trip costs
  ~11–12 bp against typical cross-venue dispersion of a few bp. That preset ships as the
  control case rather than being tuned away.
- **Passive execution is not free.** Adverse selection on a resting quote is worth ~10 bp at a
  15-minute quote life, against a ~3 bp maker fee saving. Before that charge existed, a passive
  entry rested while the market fell 0.5%, filled at its stale limit price, and lost roughly
  forty times the fee it had saved.
- **"Price differential" and "basis edge" are one quantity, not two credits.** A hedge does not
  realise the entry dislocation at entry — it realises it on convergence. Reporting both and
  summing them roughly doubled gross edge, on exactly the trades the engine most wanted to take.

What is in it:

- **A full cost stack** — maker/taker fees per venue and tier, half-spread on each of four
  crossings, book-walked slippage plus square-root market impact, financing charged on posted
  margin rather than notional, currency conversion, and execution risk decomposed into
  inter-leg exposure, adverse selection and expected leg-failure cost.
- **A chosen holding period.** The horizon is not a constant: carry accumulates with time while
  basis convergence saturates and financing grows linearly, so the engine scans a grid and takes
  the argmax of expected net. Holding a two-hour-half-life basis trade for a fixed 48 hours pays
  48 hours of financing to collect three hours of convergence.
- **An optimiser whose risk model matches the position.** Mean-variance over the observed
  cross-venue *basis-spread* covariance, not asset prices — a delta-neutral position has no
  first-order price exposure, so asset covariance measures a risk it does not have while missing
  the one it does. SLSQP with a feasible-by-construction fallback, always feasibility-checked.
- **A 23-state execution machine** where the harder leg leads, the hedging leg always crosses,
  partial fills resize the second leg to what actually filled, and a timeout re-queries by
  client order id rather than re-submitting blind.
- **Reconciliation with two asymmetries** — comparison per venue+symbol aggregate rather than
  per leg, because a venue reports one net position for several cycles; and a surplus is flagged
  rather than absorbed, because adopting unexplained size attributes real position and P&L to a
  trade that never acquired it.
- **Twelve injectable faults and ten scripted scenarios**, asserting safe behaviour rather than
  profit — a scenario that ends in a loss can pass; one that ends with exposure nobody knows
  about cannot.

**Worth knowing before cloning:** it is **paper trading only**, enforced by validators that make
the application refuse to start if the safety flags are disabled — there is no live-execution
path and no setting holds an exchange credential. The market data is **synthetic and not
calibrated against a historical dataset**; a read-only public-data mode is a documented
extension point and is *not implemented*. A typical run produces tens of trades, which is enough
to see a tendency and not enough to size a strategy — the metrics say so themselves, and flag an
implausible Sharpe as a red flag rather than a finding.

Twenty-one real bugs were found by running the system and are logged with cause and fix. The
ones that mattered: the reconciler compared each leg against the venue's *net* position and
repaired healthy hedges into naked ones; the cycles table INSERTed every tick, so the database
path had never run past tick two; and a `reduce_only` exit against a netted venue position
stranded a cycle mid-close permanently.

Verified from a fresh clone: 484 tests, 27/27 demo steps, 10/10 scenarios, ruff and mypy clean
across 67 source files, 18 tables migrated on PostgreSQL 16, four-container Docker stack up from
one command.

https://github.com/pranay123-stack/perp-funding-basis-arb

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
dressing it up.

The arbitrage platform's headline result is likewise a negative one: priced through a full cost
stack at retail fee levels, cross-venue perpetual funding arbitrage does not clear its costs, and
the control preset trades nothing at all. It is a fee-tier and execution-quality business, and
the platform demonstrates that instead of tuning the costs until the strategy looks profitable.

No live-traded track record is published in either project, because there is none.

**Tech:** Python 3.12, FastAPI, SQLAlchemy 2.0 async, PostgreSQL, Alembic, React, TypeScript,
Docker

---
