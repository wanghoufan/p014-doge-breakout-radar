# PEPE / DOGE / ETHFI Breakout Radar

[简体中文](./README.md) | English

![Homepage, live radar (real screenshot from the production site)](docs/qa/ethfi-acceptance/01-home.png)

Production site: https://pepe-doge-breakout-radar.vercel.app (auto-deployed from the `main` branch)

This is a market-observation web app (V2) built on top of a quantitative research package. It turns the researched quantitative features into an observation radar that is strictly point-in-time aligned (each moment is computed only from data that had already closed by then, so future data never leaks into a signal), works in both live and backtest mode, and can tell a successful breakout apart from a failed one. It continuously tracks the build-up, brink, breakout, follow-through, and failure states of PEPE and DOGE, replacing fuzzy judgement calls with six independent scoring layers (environment gate / setup / trigger / follow-through / risk / hard veto) plus a ten-state machine.

> This project **is not investment advice** and **does not output any win rate, accuracy, false-breakout probability, or return promise for trading decisions**. Backtest metrics serve only as evidence of research honesty.

---

## What it can do

- **Live radar**: for PEPE and DOGE separately, it outputs layered signals — environment gate (ALLOW/CAUTION/BLOCK) → setup score → trigger (breakout) score → follow-through score → risk score → hard veto — and then classifies the result into one of ten states (market blocked / no setup / setup forming / brink of breakout / breakout confirmed / awaiting follow-through / healthy breakout / pullback confirmation / breakout failed / structure broken).
- **No future-data leakage**: breakouts use a Rolling Breakout definition — `rollingHigh(t) = MAX(high[t-42..t-1])`, and the breakout only counts when `close > rollingHigh` on fully closed 4-hour candles. The setup layer uses only pre-breakout data; the follow-through layer uses only post-breakout data. MFE/MAE (maximum favorable/adverse excursion — how far price later moved for or against the breakout) are measured from `breakoutClose` and used only as after-the-fact labels.
- **Explainable evidence**: every state and every scoring layer shows "value + evidence source + timestamp + freshness". Missing data is explicitly marked `WAITING` / `PENDING` / `DATA_UNAVAILABLE` and never disguised as a 0 score.
- **Historical events**: 21 research waves (11 PEPE + 10 DOGE) recomputed under V2; each one can be opened to see the breakout level and distance, MFE/MAE, strength relative to BTC, the window's candles, and the original screenshot.
- **Success/failure samples and backtest**: the frozen sample baseline is Raw layer 290 (PEPE 140 + DOGE 150 — kept only as evidence of signal inflation, forbidden as a sample) / Episode layer 116 independent episodes (41 successes + 75 failures, frozen label rule `72H MFE ≥ 10% AND MAE ≥ −8%`) / Normal layer 768 windows (760 evaluable: PEPE 379 / DOGE 381), with anomalous windows recorded as separate events. Walk-forward out-of-sample backtesting (the model is only ever tested on data after its training window) honestly reports the conclusion that "a naive breakout signal ≈ no excess return". The old figure of "217 (75 successes / 142 failures)" came from a legacy raw-cooldown method, is void, and must not be used as a sample baseline (the frozen rules live in `PRE_ALERT_DATA_PROTOCOL_V1.md` §4).
- **Similarity engine**: each wave is turned into a five-dimension feature vector (ATR contraction ratio / volume contraction ratio / pre-launch return / pre-launch ATR / pre-launch funding rate — ATR measures typical price swing size, and the funding rate is the periodic payment between perpetual-futures longs and shorts; all five dimensions use only pre-breakout data, no future information), answering "which historical wave does the current market most resemble".
- **Graceful degradation**: when `overview` is unavailable it is marked `unavailable`; `candles` / `funding` failures return 503 with real diagnostics (a historical snapshot is never passed off as live data); historical mode is always fully available.

## Pages

| Route | Description |
|---|---|
| `/` | Overview: live radar + historical samples |
| `/asset/pepe`, `/asset/doge` | Single-asset detail |
| `/history` | Historical sample archive (filterable by asset and wave period) |
| `/history/:id` | Event detail: layered metrics + MFE/MAE + window chart + original screenshot |
| `/similarity` | Similarity: historical feature space + which wave the present resembles |
| `/methodology` | Methodology: ten-state machine + layered scoring + evidence rules + backtest |

## Tech stack

Next.js 16 (App Router) · React 19 · TypeScript 5 · Tailwind CSS 4 · shadcn/ui · hand-drawn SVG candlestick charts · server-side data proxy.

## Quick start (running locally)

```bash
pnpm install
bash scripts/dev.sh          # development mode (tsx watch src/server.ts)
```

Run the tests (73 unit tests + a 21-event regression):

```bash
pnpm test
```

## Directory overview

```
src/lib/          ten-state machine, layered engine (v2/engine.ts), Rolling Breakout, MFE/MAE, relative strength, historical events, data proxy, snapshot loading
src/app/          page routes + REST endpoints (/api/*)
src/components/   UI components (SVG candlestick chart, ten-state badges, layered signal cards, radar cards, etc.)
src/data/         research snapshot JSON (metrics for 21 events + 4H candles for 3 assets + funding rates)
public/screenshots/  21 original screenshots (P01..P11 / D01..D10)
scripts/recompute-v2.ts  offline recomputation of the 21 events + full scan + walk-forward backtest
```

## Data sources

- Historical snapshots: `event_metrics.csv` (21 events) from the research package, OKX 4-hour candles, Binance funding rates.
- Live data: OKX (candles / index price) and Binance (funding rates), proxied server-side by `src/lib/market-client.ts` with timeouts, in-memory caching, rate-limit deduplication, and error classification.
- Time: stored in UTC, displayed as Asia/Shanghai on the pages.

## Known gaps (stated as-is, no fill-in fabrication)

- Derivatives metrics are incomplete: funding is integrated, but open interest and liquidations are not yet, so the risk layer and `LIQUIDITY_VETO` currently lack those dimensions.
- Limited sample size: only two assets, about 883 days of 4-hour data, frozen baseline of 290 raw / 116 episodes (41 successes / 75 failures) / 768 normal windows (760 evaluable); walk-forward folds are sample-sparse, so precision/recall fluctuate a lot. The old "217 breakouts" figure uses the legacy raw-cooldown method and is void.
- The MFE/MAE label thresholds (MFE ≥ 10% / MAE ≥ −8%) come as a single parameter set; no sensitivity analysis has been run.
- In sandbox or partially deployed environments without outbound access to OKX / Binance, the live radar shows "real-time data unavailable"; the historical dimension remains fully usable.

## Changes in this push (2026-09-10 to 09-11, not the first push)

> Based on the 2 commits after `origin/main` (`39a63f3` ETHFI three-asset integration) plus this README update.

- `73d6a18` feat(research): P0 data-source probes + collection infrastructure + derive completion (the 24h baseline is pending formalization; the live pipeline consumes none of it — offline research only).
  Added `research/source-probe/` (OKX/Binance probes + 24h baseline script), `src/lib/research/` (schema/store/ingest/venue-adapters/field-freshness/instrument-registry/isolation) plus the `research-infra` / `research-schema` unit tests; added `ETHFI_PRODUCTION_CLOSEOUT.md` and `PRE_ALERT_DATA_PROTOCOL_V1.md` (frozen sample rules: Raw 290 / Episode 116 〈41 successes/75 failures〉 / Normal 768 〈760 evaluable〉, the old 217 figure is void); included production read-only re-verification evidence under `docs/qa/ethfi-production/`.
- `ecced00` feat(research): forward collector + baseline clarification (QA PASS; zero consumption by the live path).
  Added `src/lib/research/collector.ts`, `research/forward-collect/`, and `research/ELIGIBLE_COVERAGE_V1.md`; `SOURCE_CADENCE_BASELINE_V1.md` baseline formalized.
- This update: added the homepage product screenshot to the top of the README (`docs/qa/ethfi-acceptance/01-home.png`, a real production capture) and the production URL; aligned the title and sample wording with the ETHFI three-asset scope and the frozen baseline.

## Disclaimer

This project is a market-observation and research-methodology showcase. All output is quantitative description based on historical data and public indicators, with no risk-return promise of any kind.
