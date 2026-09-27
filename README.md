# Signal Engine — Architecture

A real-time signal engine for a single liquid instrument, built solo: data ingestion,
scoring, guard rails, paper-trade evaluation, and a live dashboard.

> This document describes the **architecture only**. Strategy parameters, research
> findings and edge logic are intentionally omitted.

## System overview

```
   market data API
        |
        v
 [ ingestion + retry/alignment ]  ->  [ candle store (PostgreSQL) ]
        |                                        |
        v                                        v
 [ engine loop (fixed interval) ] <------- [ closed candles only ]
        |
        |-- indicator + structure scoring (multi-timeframe)
        |-- guard rails: session calendar | economic-news window | higher-timeframe bias
        |-- execution model: staged exits, break-even move, bar-clock timeout
        |
        v
 [ signals table ] -> [ paper-trade evaluator ] -> [ REST API ] -> [ React dashboard ]
```

## Components

**Ingestion**
- Pulls OHLC candles per timeframe on an adaptive schedule
- Retries when the newest bar is not yet published; aligns feed timestamps to the
  venue's server time so bar boundaries match the terminal
- Maintains a daily-candle set with throttled refresh for regime analytics

**Candle store**
- PostgreSQL; the engine only ever reads **closed** bars — no look-ahead

**Scoring engine**
- Multi-timeframe structure score built from confluence primitives
  (zone retests, sweeps, displacement, trend context)
- A candidate must clear a per-timeframe score threshold plus a valid trigger
  (fresh structure retest) before it can emit
- Optional higher-timeframe bias filter: the arbitrating timeframe's trend must not
  oppose the trade direction

**Guard rails**
- Market calendar: weekend window and a daily exchange break — the engine idles when
  the market is closed and never evaluates bars from a closed session
- Economic-news guard: suppresses new entries around high-impact scheduled events,
  records the suppressed setups to a shadow table for later analysis
- Catch-up guard: when the engine restarts, historical bars are replayed, but setups
  that already resolved (stop hit, target touched) are skipped — no phantom signals

**Execution model** (implemented identically in live and research)
- Risk-based sizing from the stop distance
- Staged exit: partial at the first target, stop moved to break-even, full target after
- Timeout measured in **trading bars**, not wall-clock, so sessions and breaks don't
  consume holding time

**Evaluation & research**
- Paper-trade evaluator resolves every emitted signal against subsequent candles
- A Python research pipeline mirrors the live logic bar-for-bar (parity-tested) and
  runs the same execution model, so backtest, walk-forward and live numbers are
  directly comparable
- Validation includes walk-forward folds, slippage/cost stress, and drift checks

**Delivery**
- REST API + WebSocket updates to a React dashboard: active signals, bias matrix,
  performance breakdowns, regime context

## Engineering notes

- Every rule added to the engine was validated in research first; the dashboard exposes
  live state (why a setup was skipped, score breakdowns, guard status) so behavior is
  auditable rather than opaque.
- The same code paths are used for backtest and live emission; differences are limited
  to data source and lateness rules.
