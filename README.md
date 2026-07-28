# Binance USDⓈ-M Futures Trend-Following Bot

A perpetual-futures trading bot for Binance USDⓈ-M Futures that scans a
liquidity-filtered universe of coins for 4h-timeframe trend continuation
setups, sizes and manages positions with fixed risk-per-trade, and runs
paper (simulated) trades against live market data — with a full backtest
engine to validate the strategy before ever risking capital.

## Problem

Most small-account retail trading bots either overfit a backtest until it
looks perfect, or start trading live with no falsifiable evidence the edge
is real. This project's goal was the opposite: build a single, disciplined
trend-following strategy, validate it out-of-sample on a year of real
Binance data, and only then run it forward as paper trades — with every
subsequent change (risk sizing, universe size, execution timeframe,
experimental sub-strategies) A/B-tested against a fixed baseline before it's
allowed to touch the default configuration.

The strategy itself: trade *with* the trend, not against it. Enter only
when trend (EMA200/EMA50), strength (ADX), direction (Supertrend), and
momentum (RSI/MACD) all agree, confirmed by a volume spike and either an RSI
cross or a breakout trigger. Size every trade off a fixed dollar risk, scale
out half the position at +2R, and manage the rest with a trailing exit —
straightforward, mechanical, and (per the validation below) genuinely
backtested rather than curve-fit.

## Architecture

```
scanner.js          universe selection (volume/listing-age/open-interest
                     filters, correlation guard) + per-symbol snapshot fetch
        |
strategyIndicators.js   pure indicator math (EMA/RSI/ADX/Supertrend/MACD/
                         ATR/Bollinger) over OHLCV candles — no I/O
        |
signalingEngine.js   entry/exit decision logic: trend/strength/direction/
                     momentum gates -> WATCH/READY classification, plus
                     exit rules (stop, TP1 scale-out, trailing, time exit)
        |
positionManager.js   risk-based position sizing (fixed $ or % of equity),
                     stop/TP computation, min-notional & leverage validation
        |
binanceClient.js     Binance USDⓈ-M REST wrapper — live market data (klines,
                     exchange info, funding, open interest) always real;
                     order placement is paper-trading-gated (see below)
        |
main.js              orchestrates the 4h scan loop (+ 1h light pass), or
                     replays historical candles through the identical
                     signalingEngine/positionManager/tradeLogger path for
                     backtesting — live and backtest share the same decision
                     logic by construction, not by convention
        |
tradeLogger.js        appends every closed trade to a JSONL log, computes
                       win rate / profit factor / expectancy / drawdown
        |
dashboardServer.js + dashboard/   a small Express + vanilla-React dashboard
                                   reading the live trade log — positions,
                                   closed trades, alerts, per-symbol P&L
```

Everything downstream of `signalingEngine.js` and `positionManager.js` is
config-flag-gated: features like percent-of-equity sizing, the 1h execution
timeframe, or the experimental mean-reversion sleep all default to *off*,
reproducing the validated baseline unless explicitly enabled.

**Paper trading by default.** `binanceClient.placeEntryOrder`/`placeStopOrder`
are gated on `PAPER_TRADING` (`true` unless explicitly set to `false`) — in
paper mode they log a simulated fill and never place a real order. Live
order routing is intentionally left as a TODO stub throughout the codebase;
this repo has never placed a real order.

## Validation

365-day backtest, long-only, default configuration (fixed $ risk per trade,
4h execution timeframe, ~35-40 symbol universe):

| Metric | Result |
|---|---|
| Trades | 177 |
| Win rate | 58.8% |
| Profit factor | 1.85 |
| Expectancy | +0.24R per trade |
| Max drawdown | 18.1% |

This is the baseline every subsequent change is measured against. Changes
that degraded this baseline in testing (universe expansion to 80 coins,
symmetric short-side entries, a tighter entry-extension filter) were
identified and reverted — see `CHANGELOG.md` for the specific before/after
numbers. Reproduce it yourself:

```bash
npm run backtest -- --days=365
```

Results are written to `backtest_results_365d.json` (gitignored — it's a
regeneratable artifact, not source) and printed to the console, including a
go-live gate check (`expectancy >= 0.10R`, `profit factor >= 1.30`).

## Setup

**Requirements:** Node.js >= 18, a Binance account (testnet keys are fine —
see below).

```bash
git clone <this repo>
cd <this repo>
npm install
cp .env.example .env
```

Edit `.env` and fill in your Binance API credentials. For paper trading you
don't need real trading permissions — [Binance Futures
Testnet](https://testnet.binancefuture.com/) keys work fine, since this bot
never places real orders unless `PAPER_TRADING=false` (unimplemented — see
Architecture above). Leave every other `.env.example` value at its default
unless you know you want to change it; the advanced flags section is
optional and documented inline.

**Run the bot live (paper trading):**
```bash
npm start
```
Scans the universe every 4h bar close (00/04/08/12/16/20 UTC), with a
lighter hourly pass to catch triggers firing intra-bar. Logs simulated
trades to `logs/trades_paper.jsonl`.

**Run the dashboard** (separate process, reads the live trade log):
```bash
npm run dashboard
```
Then open `http://localhost:3001`.

**Run the test suite:**
```bash
npm test
```
Tests run under an isolated `TEST_MODE` that redirects all trade/position
logs to `*_test` files — they never read or write your live trading state.

**Run a backtest:**
```bash
npm run backtest -- --days=365
```

**Run the A/B harness** (compares two configs over identical historical
data):
```bash
npm run ab -- --days=365 --ab-flag=EXECUTION_TIMEFRAME --ab-value=1h
```

**Deploying for continuous operation:** the bot and dashboard are designed
to run under a process manager (this repo was developed against
[PM2](https://pm2.keymetrics.io/) with `ecosystem.config.js`) so a machine
restart doesn't lose state — see `recoverOpenPositions`/`recoverRealizedEquity`
in `main.js` for how open positions and realized P&L survive a restart.

---

*Not financial advice. This is a paper-trading research project; nothing
here places real orders as shipped, and past backtest performance is not a
guarantee of future results.*
