# Basis — Product Requirements Document (Agile)

**Version 1.0 · Owner: Founder/PM · Status: Ready for Sprint 0**

This PRD supersedes the exploratory design docs as the *build* reference. Those docs remain valid as deep-dives (traceability in §14), but where they conflict with this document, **this document wins** — because most of them were written to explore what's possible, and this one is written to decide what's actually getting built, in what order.

The single biggest change from the design phase: **the architecture has been deliberately shrunk.** The nine-service, Kafka-backed, four-datastore design was correct for a mature product and wrong for a pre-revenue one. §5 collapses it. If you read only one section before starting, read that one.

---

## 1. Problem & Vision

**Problem.** Retail and semi-professional traders build strategies against backtests that assume perfect fills, zero fees, and no market impact. Those backtests produce Sharpe ratios and equity curves that do not survive contact with a live order book. SEBI's own FY26 study found roughly 9 in 10 individual F&O traders in India lost money. The tools they're paying ₹300–3,000/month for are not the cause of that, but they are not preventing it either — because they hide the exact variable that kills live performance.

**Vision.** A backtesting platform where realistic execution modelling is the default, not an upgrade — and where every fill can be traced to a stated reason. Easy enough for a no-code options trader, honest enough for a quant.

**Positioning statement.** *For* retail and semi-pro traders who don't trust their backtests, *Basis* is a strategy design and backtesting platform *that* models real fills, fees, and slippage by default and shows you why each fill happened — *unlike* no-code tools that hide execution cost or code-first engines that make you model it yourself.

**Non-goals (v1).** Not a broker. Not a signal-selling marketplace. Not an HFT platform. Not a portfolio management or accounting system.

---

## 2. Success Metrics

Leading indicators come first deliberately — the lagging ones can't move for months, and steering by them early leads to building on guesses.

| Horizon | Metric | Target | Why this one |
|---|---|---|---|
| R1 (Demo) | Demo → waitlist conversion | ≥ 8% of unique visitors | Tests whether the honesty positioning lands at all |
| R1 | Qualitative: unprompted "I'd trust this more" in ≥ 5 of 10 user interviews | 5/10 | The whole thesis in one sentence |
| R2 (Beta) | Week-4 retention of activated users | ≥ 25% | The single strongest PMF signal available pre-revenue |
| R2 | Activation: signup → first completed backtest | ≥ 40% | Measures whether the builder is actually usable |
| R2 | Median time signup → first backtest result | < 10 min | Ease claim, made falsifiable |
| R3 | Backtest-vs-paper-trade divergence on matched strategies | median < 5% | **The product's core claim, measured.** If this is bad, nothing else matters |
| R4 | Free → paid conversion | ≥ 4% | Most sensitive assumption in the financial model; currently a guess |

**Counter-metric (watch for gaming):** if backtest-vs-live divergence looks great but users churn, the model is probably over-penalising and making every strategy look unprofitable. Track both together.

---

## 3. Personas

| Persona | Context | Primary job-to-be-done | Success looks like |
|---|---|---|---|
| **Priya — India retail options trader** | Trades NIFTY/BANKNIFTY weeklies on Zerodha. Uses Streak. Doesn't code. | "Tell me if this strangle actually made money after costs." | Builds a 4-leg combo without code, sees a realistic P&L, understands *why* it differs from what she expected |
| **Daniel — US quant / small prop** | Writes Python. Uses Backtrader or QuantConnect. Frustrated by both. | "Prove this edge isn't curve-fit before I allocate." | Runs a parameter sweep, gets an overfitting score, exports a report he trusts |
| **Ops (you, initially)** | Runs the system | "Know what's broken before a user tells me." | Health visible, logs greppable, config changeable without a deploy |

Priya is the **primary** persona for v1. Where a tradeoff exists between Priya's needs and Daniel's, Priya wins — she's the larger, more underserved, more monetisable segment, and she's the one no competitor serves honestly.

---

## 4. Decisions Resolved

These were left open at the end of the design phase. Leaving them open blocks Sprint 1, so each gets a decision, a rationale, and a review trigger.

| # | Decision | Rationale | Revisit if |
|---|---|---|---|
| **D1** | **Build the engine in-house (C++), do NOT adopt NautilusTrader for v1.** | Nautilus is excellent but its Python/Rust strategy model doesn't accept a compiled-graph WASM module, has no India broker adapters, and pulls you into an LGPL boundary review before you've validated demand. The v1 engine (§5) is small — bar-level, single-instrument, statistical slippage — genuinely weeks, not months. | Engine work exceeds 6 weeks, or a customer demands tick-level multi-venue before R4. Run the Nautilus spike in R3 as a de-risk, not a blocker. |
| **D2** | **v1 is a modular monolith, not microservices.** | Nine services + Kafka + ClickHouse + S3 + Postgres is roughly 4× the operational surface of the actual v1 feature set. Module boundaries are preserved in-code so the split stays cheap later. | Any single module needs independent scaling, or team size exceeds ~4 engineers. |
| **D3** | **Graph compiles via template codegen to WASM — but v1 ships a tree-walking interpreter first.** | The interpreter is ~1 sprint; the compiler is ~3 and needs a Rust toolchain in the deploy path. The engine dominates runtime anyway. Ship interpreted, measure, then compile. | Sweep throughput becomes the measured bottleneck (instrument this from day one so the question is answerable with data). |
| **D4** | **Design direction: CRED-inspired dark/warm palette with lime accent and the "product panel" component system.** | It tested best in review, it's differentiated in the fintech landscape, and it's already prototyped. **Locked — no further theme exploration until R2 ships.** Theme churn was the single largest time sink of the design phase. | Post-R2 user research says the aesthetic reads as untrustworthy for a financial tool. |
| **D5** | **Skip the WASM sandbox in R1; add it before any external user runs code.** | R1 is single-tenant (you + a handful of pilot users on invite). Isolation is a hard gate for R2, not R1. | Any non-trusted user gets access earlier than planned. |
| **D6** | **Postgres only in v1. No ClickHouse, no Kafka.** | Postgres handles hundreds of millions of bars with partitioning and a BRIN index. ClickHouse earns its place when scan latency is measurably a problem, not before. | p95 backtest data-load exceeds 2s, or bar volume exceeds ~500M rows. |
| **D7** | **Defer OpCon, the matching engine, NBBO multi-venue, marketplace, and SEBI compliance beyond v1.** | None are needed to test the core hypothesis. Each is fully designed and can be picked up later without rework — that's what the adapter/registry patterns were for. | India live-trading launch is committed (compliance becomes a hard blocker at that point — see R4). |

---

## 5. v1 Architecture (Simplified)

This is the most important change in this document. The full design remains the target state; this is the path to it that doesn't sink the project first.

```
┌──────────────────────────────────────────────────────────┐
│  Web App (React + TypeScript, React Flow canvas)          │
└───────────────────────────┬──────────────────────────────┘
                            │ REST /v1
┌───────────────────────────▼──────────────────────────────┐
│  Basis Server — single Go binary, modular internally       │
│                                                            │
│   ├── api/          auth, routing, validation              │
│   ├── strategy/     graph CRUD, versioning, checksums      │
│   ├── ingest/       vendor adapters, CSV import            │
│   ├── runner/       job queue (Postgres-backed), dispatch  │
│   └── results/      metrics, walk-forward, reports         │
└───────────┬────────────────────────────┬──────────────────┘
            │ gRPC (streamed)             │ SQL
┌───────────▼──────────────┐  ┌───────────▼──────────────┐
│  Engine (C++ binary)      │  │  Postgres                 │
│  · event loop             │  │  · metadata + bars        │
│  · graph interpreter      │  │  · partitioned, BRIN idx  │
│  · IExecutionModel        │  └───────────────────────────┘
│  · order/fill sim         │  ┌───────────────────────────┐
└───────────────────────────┘  │  Local FS / S3            │
                                │  · trade logs (parquet)   │
                                └───────────────────────────┘
```

**What survives from the original design, unchanged and non-negotiable:**
- Event-driven engine with strict time-ordering (look-ahead bias prevention is architectural, not a test)
- `IExecutionModel` / `ISlippageModel` registry pattern — new models are one file + one registration
- `IMarketDataAdapter` / `IBrokerAdapter` ports — vendor and broker variation never leaks into core logic
- Immutable, checksummed `strategy_versions` — reports stay reproducible
- Per-fill `reason` string — the product promise, implemented at row level
- Typed DAG graph model with topological sort

**What's explicitly deferred:** Kafka (Postgres `SELECT ... FOR UPDATE SKIP LOCKED` is a fine job queue at this scale), ClickHouse, WASM sandbox, the Rust compiler service, OpCon, the compliance service, the matching engine.

**Migration path stays open** because module boundaries are real: each Go package has its own interface and no cross-package struct sharing. Splitting `runner/` into its own service later is a deployment change, not a rewrite.

---

## 6. Scope — MoSCoW by Release

| | R1 · Proof (8 wks) | R2 · Private Beta (12 wks) | R3 · Live (10 wks) | R4 · India GA (12 wks) |
|---|---|---|---|---|
| **Must** | C++ engine (bar-level, 1 instrument), 2 slippage + 2 fee models, graph interpreter, ~22 nodes, React Flow canvas, two-curve result chart, CSV + 1 vendor adapter, EMA-crossover + iron-butterfly worked examples | Multi-tenant auth, WASM sandbox, multi-leg engine support (time-ordered merge), options combo nodes, walk-forward + overfitting score, parameter sweeps, ~60 nodes, saved/versioned strategies, results dashboard | Broker adapter interface + 1 broker, paper trading, live bridge reusing sandbox module, order state machine, idempotency keys, live-vs-backtest divergence dashboard | Algo-ID tagging, hash-chained audit log, static-IP/OAuth enforcement, billing, ~90 nodes, OpCon MVP |
| **Should** | Deployed demo URL, landing page | Strategy templates, report export (PDF/CSV), Custom Formula node | Second broker, alerts/webhooks | Rate-threshold detection, black-box flagging |
| **Could** | — | Graph→WASM compiler (if D3 trigger fires) | Nautilus spike (de-risk D1) | Marketplace with verified backtest reports |
| **Won't** | Auth, multi-user, live trading, options | Live trading, compliance | Compliance, marketplace | Matching engine, NBBO multi-venue, on-prem |

---

## 7. Epics

| ID | Epic | Release | Points | Outcome |
|---|---|---|---|---|
| **E1** | Foundations & CI | R1 | 13 | Repo, CI, container builds, Postgres migrations |
| **E2** | Market data ingestion | R1 | 21 | Bars from one vendor + CSV, queryable, corporate actions applied |
| **E3** | Execution engine core | R1 | 34 | Event loop, portfolio accounting, look-ahead-free, gRPC-streamed |
| **E4** | Execution realism | R1 | 21 | Slippage/fee registries, order sim, per-fill reasons |
| **E5** | Graph model & interpreter | R1 | 21 | Typed DAG, validation, topological sort, node evaluation |
| **E6** | Node library — core set | R1/R2 | 34 | 22 nodes in R1 → 60 in R2 |
| **E7** | Strategy builder UI | R1/R2 | 34 | React Flow canvas, typed ports, inline config, save/validate |
| **E8** | Results & honesty layer | R1/R2 | 21 | Metrics, two-curve chart, walk-forward, overfitting score |
| **E9** | Multi-tenancy & sandbox | R2 | 21 | Auth, WASM isolation, per-tenant quotas |
| **E10** | Multi-leg & options | R2 | 34 | Time-ordered merge, portfolio context, combo actions, strike selectors |
| **E11** | Sweeps & optimisation | R2 | 13 | Fan-out, walk-forward windows, overfitting warnings |
| **E12** | Live trading | R3 | 34 | Broker port, paper mode, order state machine, divergence tracking |
| **E13** | Compliance & billing | R4 | 34 | Audit chain, Algo-ID, OAuth/IP, subscriptions |
| **E14** | Ops tooling (OpCon) | R4 | 21 | Status/start/stop/logs, candidate-commit config |

Points are relative (Fibonacci), calibrated so **1 point ≈ half a focused day** for one developer with AI assistance. Assumed velocity: **20 points/sprint, 2-week sprints, one developer.** Re-baseline after Sprint 2 using actuals — the first two sprints are always wrong.

---

## 8. User Stories — R1 (the sprints you'll actually start)

Format: *As a [persona], I want [capability], so that [outcome].* Acceptance criteria in Given/When/Then. Only R1 is expanded to story level here; R2–R4 stories get written at their release-planning session, per agile practice — writing them now would be waste, since R1 learnings will change them.

### E1 — Foundations (13 pts)

**S1.1 — Monorepo & CI** · 5 pts
> As a developer, I want a monorepo with per-language CI so that a broken build blocks merge.
- **Given** a PR touching `/engine`, **when** CI runs, **then** the C++ build and unit tests execute and a failure blocks merge.
- **Given** a PR touching `/server` or `/web`, **then** the corresponding Go/TS build, lint, and tests run.
- Structure: `/engine` (C++), `/server` (Go), `/web` (React), `/proto` (shared contracts), `/docs`.

**S1.2 — Postgres schema & migrations** · 5 pts
> As a developer, I want versioned migrations so that schema changes are reproducible.
- **Given** a clean database, **when** migrations run, **then** `strategies`, `strategy_versions`, `backtest_runs`, `run_results`, `market_bars` exist with documented indexes.
- **Given** `market_bars`, **then** it is partitioned by month with a BRIN index on `(symbol, ts)`.
- `strategy_versions` is append-only, carries `checksum`, and has no UPDATE path in application code.

**S1.3 — Local dev environment** · 3 pts
> As a developer, I want one command to run the whole stack locally.
- **Given** a clean machine with Docker, **when** `make dev` runs, **then** Postgres, the Go server, and the web app start and the health endpoint returns 200.

### E2 — Market Data (21 pts)

**S2.1 — `IMarketDataAdapter` interface + first vendor** · 8 pts
> As Priya, I want historical NIFTY/BANKNIFTY data available so that I can backtest without sourcing data myself.
- **Given** a symbol, timeframe, and date range, **when** ingestion runs, **then** bars are normalised to `CanonicalBar` and persisted.
- **Given** a re-run over an already-ingested range, **then** no duplicate rows are created (idempotent on `(vendor, symbol, timeframe, ts)`).
- **Given** a vendor that lacks a requested timeframe, **then** `VendorCapabilities` causes a clear pre-flight error, not a partial ingest.

**S2.2 — Golden-file contract tests** · 3 pts
> As a developer, I want recorded vendor fixtures in CI so that a vendor changing their format fails the build, not a backtest.
- **Given** a stored fixture response, **when** the adapter parses it, **then** output matches the expected `CanonicalBar` set exactly.

**S2.3 — CSV import with column mapping** · 8 pts
> As Daniel, I want to upload my own CSV so that I can test against data I already have.
- **Given** an uploaded CSV, **when** headers are read, **then** a suggested column mapping with per-field confidence is returned.
- **Given** a confirmed mapping, **then** it is saved as a reusable named profile, and re-uploads from that source require no re-mapping.
- **Given** 3 malformed rows in 50,000, **then** 49,997 import and a rejection report lists each bad row with a specific reason.
- Timezone and decimal separator are **explicit fields**, never inferred.

**S2.4 — Corporate actions adjustment** · 3 pts
> As a trader, I want splits and dividends handled so that a corporate action isn't mistaken for a return.
- **Given** a split on date D, **when** bars spanning D are loaded, **then** pre-split prices are adjusted and the adjustment is recorded.

### E3 — Engine Core (34 pts)

**S3.1 — Event-driven simulation loop** · 13 pts
> As a trader, I want my strategy evaluated bar-by-bar in strict time order so that results aren't contaminated by future data.
- **Given** a bar series, **when** the engine runs, **then** the strategy is invoked once per bar with access only to data at or before that bar's timestamp.
- **Given** any attempt to access a future bar, **then** it is impossible by construction (API exposes no forward access), not merely untested.

**S3.2 — Portfolio & P&L accounting** · 8 pts
- **Given** a sequence of fills, **then** position, cash, equity, and realised/unrealised P&L are correct to 2dp against a hand-verified fixture.
- **Given** an equity curve, **then** drawdown is computed per bar.

**S3.3 — Look-ahead bias test suite** · 5 pts
> As a developer, I want automated proof of no look-ahead so that the core claim is defended by CI.
- A property test with a deliberately cheating strategy (references bar t+1) **fails to compile or is rejected at validation**.
- A known-signal fixture produces the exact expected trade sequence.

**S3.4 — gRPC contract, streamed** · 8 pts
- **Given** a `BacktestJobRequest`, **when** the engine runs, **then** it streams `Progress`, `TradeFill`, and `Completed` events.
- **Given** a 6-month 15m backtest, **then** first progress event arrives < 1s.

### E4 — Execution Realism (21 pts) — *the differentiator; do not compress this epic*

**S4.1 — `ISlippageModel` registry** · 5 pts
- **Given** a new model file with `REGISTER_SLIPPAGE_MODEL(...)`, **when** the engine builds, **then** it is selectable by string ID from run config with **zero changes** to engine or orchestrator code.

**S4.2 — Fixed + volume-impact models** · 5 pts
- Both models registered, parameterised, and covered by the conformance suite.
- **Given** identical config and seed, **then** results are bit-identical across runs.

**S4.3 — Fee model registry + flat/percentage** · 3 pts
- Fees applied per fill, not averaged across the run.

**S4.4 — Order simulation with fill probability** · 5 pts
- Market, limit, and stop orders each behave per spec; partial fills occur when requested size exceeds available volume at price.

**S4.5 — Per-fill `reason` string** · 3 pts
> As Priya, I want to know why a fill happened at that price so that I can trust the number.
- **Given** any fill, **then** `trade_fills.reason` contains a human-readable explanation (e.g. "partial fill: 40% of avg 1m volume at limit").
- **Given** the results UI, **then** the reason is visible per row without an extra request.

### E5 — Graph Model & Interpreter (21 pts)

**S5.1 — Graph JSON schema + typed ports** · 5 pts
- **Given** a graph, **when** validated, **then** every edge's source output type matches the destination input type, or a specific error names the offending edge.

**S5.2 — Cycle detection & topological sort** · 5 pts
- **Given** a cyclic graph, **then** save succeeds but validate fails, naming the nodes in the cycle.
- **Given** a valid DAG, **then** a deterministic execution order is produced.

**S5.3 — Node evaluation interpreter** · 8 pts
- **Given** a sorted graph, **when** the engine runs a bar, **then** each node evaluates once, reading cached upstream outputs.
- Stateful nodes (EMA, RSI) persist state across bars keyed by node ID.
- A node output referenced twice (e.g. one EMA feeding entry and exit) is computed **once**.

**S5.4 — Param schema validation** · 3 pts
- **Given** a param outside its declared min/max, **then** the run is rejected pre-dispatch with a field-level error.

### E6 — Node Library, R1 subset (13 of 34 pts)

**S6.1 — 22 core nodes** · 13 pts
> As Priya, I want enough building blocks to express a real strategy without code.

R1 set, chosen to make both worked examples buildable and cover one node per category:
`Bar Data Source` · `SMA` · `EMA` · `RSI` · `ATR` · `Bollinger Bands` · `Rolling High` · `Rolling Low` · `Swing Low` · `Bullish Engulfing` · `Crosses Above` · `Crosses Below` · `Greater Than` · `Less Than` · `Breaks Above Level` · `AND` · `OR` · `NOT` · `Position Size (% equity)` · `Stop-Loss (%)` · `Enter Long` · `Exit Position`
- Each node ships with its param schema, a unit test, and an entry in the palette.

### E7 — Builder UI, R1 subset (21 of 34 pts)

**S7.1 — React Flow canvas with typed ports** · 8 pts
- Drag from palette to canvas; drag between ports to connect.
- **Given** an attempted type-incompatible connection, **then** the UI refuses it during the drag, with a reason on hover.

**S7.2 — Node palette + inline config panel** · 5 pts
- Palette grouped by category and searchable.
- **Given** a selected node, **then** its config form is **generated from its param schema**, not hand-written per node type.

**S7.3 — Save / validate / compile as distinct actions** · 5 pts
- Save works on an invalid graph (no lost work). Validate surfaces errors inline on offending nodes. Compile only proceeds when validation passes.

**S7.4 — Design system implementation** · 3 pts
- Palette, typography, and the reusable "product panel" component per D4, applied consistently across builder and results.

### E8 — Results & Honesty Layer, R1 subset (13 of 21 pts)

**S8.1 — Core metrics** · 5 pts
- Sharpe, Sortino, max drawdown, win rate, CAGR, profit factor — each verified against a hand-computed fixture.

**S8.2 — Two-curve honesty chart** · 5 pts
> As Priya, I want to see the naive result next to the modelled result so that I understand what execution cost me.
- **Given** a completed run, **then** both curves render with the gap explicitly labelled as a percentage.
- This is the hero visual — treat its polish as a feature, not styling.

**S8.3 — Trade log view with reasons** · 3 pts
- Sortable table: time, side, requested price, fill price, slippage, fee, status, **reason**.

---

## 9. Sprint Plan

**Cadence:** 2-week sprints · 20 pts/sprint · Sprint 0 is setup-only.

### Release 1 — Proof (Sprints 1–4, 8 weeks)

| Sprint | Goal (one sentence — if it needs two, the sprint is overloaded) | Stories | Pts |
|---|---|---|---|
| **S0** | Environment, repo, decisions ratified, backlog groomed | S1.1, S1.2, S1.3 | 13 |
| **S1** | Data lands: real bars queryable from a real vendor | S2.1, S2.2, S2.4, S3.1 (start) | 20 |
| **S2** | A hardcoded strategy backtests end-to-end with realistic fills | S3.1 (finish), S3.2, S4.1, S4.2 | 21 |
| **S3** | A graph — not hardcoded C++ — drives a backtest | S5.1, S5.2, S5.3, S4.5 | 21 |
| **S4** | A human can build and run a strategy in a browser | S6.1, S7.1, S7.2, S8.2 | 21 |

**R1 exit criteria (all must be true):**
- EMA-crossover strategy built in the UI, backtested, two-curve chart rendered
- Every fill carries a reason, visible in the trade log
- Adding a third slippage model requires one file and one registration line (demonstrate it)
- Deployed to a URL someone else can open
- Remaining R1 stories (S2.3 CSV, S3.3, S3.4, S4.3, S4.4, S5.4, S7.3, S7.4, S8.1, S8.3) land in the R1 buffer sprint or slip to R2 — **flagged now** because velocity assumptions are unproven

> **Buffer:** insert **Sprint 4.5** (2 weeks) before declaring R1 done. First-time velocity estimates are wrong roughly always; planning without a buffer converts that into a missed date instead of a known one.

### Release 2 — Private Beta (Sprints 5–10, 12 weeks)

| Sprint | Goal |
|---|---|
| **S5** | R1 debt cleared; CSV import and full metrics shipped |
| **S6** | Auth + WASM sandbox: a second person can safely run a strategy |
| **S7** | Multi-leg engine: time-ordered merge, portfolio context, pairs example works |
| **S8** | Options: strike selectors, option legs, combo actions — iron butterfly runs end-to-end |
| **S9** | Walk-forward + overfitting score + parameter sweeps |
| **S10** | Node library to ~60, results dashboard polish, beta onboarding |

**R2 exit criteria:** 10 external beta users onboarded; ≥40% activation; iron butterfly buildable by a non-developer without help; week-4 retention instrumented and reporting.

### Release 3 — Live (Sprints 11–16) · Release 4 — India GA (Sprints 17–22)

Goals only; stories written at their release planning:
- **R3:** broker port + 1 broker → paper trading → live bridge reusing the sandbox module → **divergence dashboard** (the metric that validates the entire product thesis). Nautilus spike here as a de-risk on D1.
- **R4:** audit chain → Algo-ID → OAuth/static-IP → billing → OpCon MVP → India launch.

---

## 10. Definition of Ready / Done

**Ready** (a story can't enter a sprint without these): user story format with a named persona · acceptance criteria in Given/When/Then · estimated by whoever will build it · dependencies identified · no open decision blocking it · UI stories have a reference design.

**Done** (system-wide): merged to main · unit tests pass in CI · acceptance criteria demonstrably met · anything touching a registry (slippage model, adapter, node type) passes its conformance suite · anything touching fills, orders, or money has a fixture test with hand-verified expected values · no new lint or type errors · docs updated if the public contract changed · **demoable in the sprint review** — if it can't be shown, it isn't done.

---

## 11. Ceremonies

Solo or two-person team — keep the loop, drop the theatre.

| Ceremony | When | Timebox | Purpose |
|---|---|---|---|
| Sprint planning | Day 1 | 60 min | Pick the sprint goal first, then pull stories to ~20 pts |
| Daily check-in | Daily | 5 min (written) | Blockers only. A written log beats a meeting at this size |
| Backlog grooming | Mid-sprint | 45 min | Refine the *next* sprint's stories to Ready |
| Sprint review/demo | Last day | 30 min | Run the demo. Record it — these become your LinkedIn/investor material for free |
| Retrospective | Last day | 30 min | One thing to keep, one to change. Adjust velocity from actuals |
| Release planning | Per release | 2–3 hrs | Write the next release's stories; re-baseline estimates |

---

## 12. Risks

| Risk | Impact | Likelihood | Mitigation | Trigger to act |
|---|---|---|---|---|
| **Silent correctness bug in slippage/P&L** — wrong numbers that look plausible | Fatal (destroys the one differentiator) | Medium | Hand-verified fixtures; property tests; cross-check against a second implementation; R3 divergence dashboard | Any fixture drift; divergence > 10% |
| **Scope creep** (this project's demonstrated failure mode — see the design-phase theme churn) | High | **High** | MoSCoW enforced per release; D4 locks design; "Won't" column is binding, not aspirational | Any story not traceable to a release Must/Should |
| Velocity assumption (20 pts) is wrong | Medium | High | Re-baseline after S2; buffer sprint before each release gate | Two consecutive sprints under 15 pts |
| Vendor data quality/cost | Medium | Medium | CSV path as fallback (S2.3); adapter interface makes swapping cheap | Vendor gaps block a demo |
| Nautilus (or a competitor) ships hosted no-code first | High | Low-Medium | Speed on R1/R2; differentiation is India + no-code + options, not the engine | They announce a no-code/hosted product |
| Solo-dev bus factor | High | — | Everything decided lives in this doc + §14 sources; commit conventions; ADRs for future decisions | — |

---

## 13. Instrumentation (build in R1, not later)

Because the metrics in §2 can't be measured retroactively: event tracking on signup → strategy created → backtest run → result viewed (the activation funnel); backtest wall-clock and data-load p50/p95 (answers D3 and D6 with data instead of opinion); per-run execution model and param set (needed to explain divergence in R3); error rates by node type (shows which nodes confuse users — the highest-signal input to the UX backlog).

---

## 14. Traceability

| This PRD section | Source document |
|---|---|
| §1, §3 problem/personas | `backtesting-saas-market-research.md`, `slippage-pmf-analysis.md` |
| §2, §9 metrics/financials | `slippage-financial-model.xlsx` |
| §5 architecture | `slippage-system-design.html`, `slippage-reimagined-system.md` |
| §5 data/API contracts | `slippage-data-model-and-api.md` |
| §7 E4 execution realism | `slippage-matching-engine-analysis.md` |
| §7 E5/E6/E7 graph & nodes | `slippage-node-designer-analysis.md`, `slippage-node-library-spec.md`, `slippage-strategy-types-node-requirements.md` |
| §7 adapters/registries | `slippage-configurability-playbook.md`, `slippage-extensibility-playbook.md` |
| §6 R4 compliance, E14 | `slippage-opcon-design.md`, `slippage-master-spec.md` §8 |
| §4 D1 | `slippage-nautilustrader-buildvsbuy.md` |
| §4 D4 design | `slippage-cred-inspired-theme.html`, `slippage-premium-fintech-researched.html` |
| §7 E12 broker/live | `slippage-configurability-playbook.md` §6 |

**Deferred-but-designed** (pick up as-is when triggered, no redesign needed): matching engine & NBBO (`slippage-matching-engine-analysis.md`), OpCon (`slippage-opcon-design.md`), full 113-node library, marketplace, on-prem.

---

## Appendix — Start Here

1. Ratify §4 decisions (or overrule with reasons recorded as ADRs).
2. Create the board: 14 epics from §7, R1 stories from §8.
3. Run Sprint 0 (§9) — environment and schema only.
4. Set the Sprint 1 goal: *"Real bars from a real vendor, queryable."*
5. Do not open a design tool until R2 ships (D4).
