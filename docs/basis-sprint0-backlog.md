# Basis — Sprint 0 & 1 Backlog (BASIS-1 … BASIS-10)

**Project key:** `SLIP` · **Cadence:** 2-week sprints · **Point scale:** 1 pt ≈ half a focused day

These ten tickets cover Sprint 0 (foundations) and the first half of Sprint 1 (data ingestion). They end at a demonstrable milestone: **real market bars from a real vendor, in Postgres, queryable, with contract tests guarding the adapter.**

Ordering is dependency-correct — work them roughly in sequence. BASIS-3 and BASIS-4 are the two that are catastrophically expensive to retrofit, so they come before anything that would consume them.

**Global Definition of Done** (applies to every ticket, not repeated below): merged to `main` · CI green · acceptance criteria demonstrably met · no new lint/type errors · public contract changes documented · demoable.

---

## BASIS-1 — Monorepo scaffolding and build system

**Type:** Task · **Points:** 5 · **Priority:** P0 · **Depends on:** — · **Component:** infra

### Context
Four languages need to coexist without their build systems fighting. Getting the skeleton right on day one avoids a painful restructure once there are hundreds of files.

### Requirements
Create the directory structure exactly as specified in the technical plan:

```
/engine        C++20 — CMake with presets
  /include/basis/     public headers (the extension surface)
  /src/                  implementation
  /tests/                unit, property, golden
/proto         .proto files — single source of truth for the Go↔C++ contract
/server        Go — modular monolith
  /cmd/basis-server/
  /internal/{api,strategy,ingest,runner,results,domain}/
/web           TypeScript + React + Vite
/fixtures      recorded vendor payloads, golden outputs, hand-verified P&L cases
/docs/adr      numbered Architecture Decision Records
Makefile       top-level orchestration
```

- CMake presets: `debug`, `release`, `asan`, `tsan`, `ubsan`. Release must **not** use `-ffast-math` and must pin `-march=x86-64-v2` (determinism baseline — see BASIS-3).
- Go module at `/server` with Go 1.22+.
- `buf` configured at `/proto` with lint rules enabled.
- Vite + TypeScript + React at `/web`.
- Top-level `Makefile` targets: `build`, `test`, `lint`, `fmt`.

### Acceptance criteria
- **Given** a clean checkout, **when** `make build` runs, **then** all four sub-projects build without error.
- **Given** a clean checkout, **when** `make test` runs, **then** placeholder test suites in each language execute and pass.
- **Given** the release preset, **when** the engine builds, **then** compiler flags contain neither `-ffast-math` nor `-march=native`.

### Deliverables
- ADR-000: monorepo structure and build tooling choices.

---

## BASIS-2 — CI pipeline with per-language gating

**Type:** Task · **Points:** 5 · **Priority:** P0 · **Depends on:** BASIS-1 · **Component:** infra

### Context
Red builds must block merge from the first commit. Retrofitting CI discipline onto an existing codebase never fully takes.

### Requirements
- Path-filtered jobs: changes under `/engine` trigger C++ build + tests; `/server` triggers Go build + `go vet` + tests; `/web` triggers tsc + lint + tests; `/proto` triggers `buf lint` **and** `buf breaking` against `main`.
- Branch protection: no merge to `main` with a failing required check.
- **Nightly scheduled job**: full test suite under the `asan` and `ubsan` presets.
- Build caching so the common path stays under ~5 minutes.

### Acceptance criteria
- **Given** a PR introducing a failing C++ test, **then** the merge button is blocked.
- **Given** a PR that removes a field from a `.proto` message, **then** `buf breaking` fails the build.
- **Given** the nightly job, **then** sanitiser output is captured in the run artifacts.

### Technical notes
Sanitisers run nightly rather than per-PR deliberately — they're 5–10× slower, and the engine is where memory bugs silently corrupt financial output, so nightly coverage is the right trade.

---

## BASIS-3 — Core value types: fixed-point money, price, quantity, and time

**Type:** Story · **Points:** 5 · **Priority:** P0 · **Depends on:** BASIS-1 · **Component:** engine

> As a developer, I want money and time represented exactly and unambiguously, so that accumulated P&L never drifts and no timezone assumption is ever implicit.

### Context
**This is the single most expensive ticket to defer.** A `double` cannot represent `0.1` exactly; accumulate P&L across 50,000 fills and drift appears that no test catches. Worse, floating-point addition isn't associative, which silently breaks the reproducibility promise. Retrofitting this after 40 files depend on `double` is a multi-week refactor with silent-corruption risk. Doing it now is an afternoon.

### Requirements
Create `/engine/include/basis/types.hpp`:

- `Price { int64_t ticks; }` — instrument tick units
- `Money { int64_t minor; }` — paise/cents
- `Qty   { int64_t units; }`
- `Timestamp { int64_t ns; }` — nanoseconds since Unix epoch, **UTC always**

Rules:
- Strong types — **not** typedefs. Adding a `Price` to a `Qty` must be a compile error.
- Explicit arithmetic only where it makes domain sense (`Price × Qty → Money`; `Price + Price` is meaningless and should not compile).
- No implicit conversion to or from `double`. Conversion helpers must be named and explicit (`to_double_for_analytics()`).
- `InstrumentSpec { int64_t tick_size; int64_t lot_size; int32_t price_scale; }` for conversions at the boundary.
- Timezone conversion lives in Go/TS only. The engine has no timezone code.

### Acceptance criteria
- **Given** `Price + Qty`, **then** compilation fails.
- **Given** 100,000 sequential `Money` additions of a repeating decimal, **then** the result is exact, and identical regardless of summation order.
- **Given** any engine source file, **when** grepped, **then** no `double` appears in a cash, price, position, or fill-price context.
- Unit tests cover overflow behaviour at `int64` boundaries and round-trip conversion via `InstrumentSpec`.

### Deliverables
- ADR-001: fixed-point money and price representation.
- ADR-002: UTC nanosecond time; no timezone logic in the engine.

### Out of scope
Currency conversion, multi-currency portfolios.

---

## BASIS-4 — Bar data structures and the look-ahead-safe accessor

**Type:** Story · **Points:** 5 · **Priority:** P0 · **Depends on:** BASIS-3 · **Component:** engine

> As a developer, I want an API in which future bars are unreachable, so that look-ahead bias is structurally impossible rather than merely tested against.

### Context
Don't hand strategies an API that *can* see the future and then write tests hoping it doesn't. Make the future unreachable. This is roughly twenty lines and is the highest-leverage code in the engine.

### Requirements

**`BarSeries`** — struct-of-arrays (columnar), not array-of-structs. The hot loop touches `close` far more than `volume`:

```cpp
struct BarSeries {
  std::vector<int64_t> ts;
  std::vector<int64_t> open, high, low, close;   // fixed-point ticks
  std::vector<int64_t> volume;
  std::unordered_map<std::string, std::vector<int64_t>> attrs;  // extension hatch
};
```

**`BarView`** — the only accessor handed to node evaluation:

```cpp
class BarView {
  const BarSeries* s_; std::size_t i_;
public:
  const int64_t& close(std::size_t back = 0) const;  // back=0 is current
  // ... open/high/low/volume/ts equivalents
  std::optional<int64_t> attr(AttrHandle h, std::size_t back = 0) const;
  // deliberately NO forward accessor and NO raw pointer escape
};
```

- `attrs` access must resolve to a column pointer **once at graph-compile time** via an `AttrHandle`. String lookup inside the bar loop is forbidden.
- **Bar timestamp convention: interval-close.** A 15m bar stamped `09:30` covers `09:15–09:30`. Document this in the header and enforce it at ingestion — vendors disagree, and this is a classic off-by-one look-ahead bug.
- `back > i_` must assert in debug and be documented as UB in release.

### Acceptance criteria
- **Given** the `BarView` public API, **then** there is no method returning data at an index greater than the current position.
- **Given** `back = 0`, **then** the current bar's values are returned; **given** `back = 1`, the previous bar's.
- **Given** an `AttrHandle` resolved at setup, **then** attribute access performs no string comparison.
- Header comment documents the interval-close convention explicitly.

### Deliverables
- ADR-003: look-ahead prevention as a structural property.

---

## BASIS-5 — Postgres schema and migration tooling

**Type:** Story · **Points:** 5 · **Priority:** P0 · **Depends on:** BASIS-1 · **Component:** server

> As a developer, I want versioned, reproducible schema migrations so that database state is never hand-managed.

### Context
Postgres only for v1 — no ClickHouse (PRD D6). Postgres handles hundreds of millions of bars with correct partitioning; ClickHouse earns its place when scan latency is *measurably* a problem.

### Requirements
Use `golang-migrate`. Initial migration creates:

- `strategies` — id, owner_id, name, kind, instrument_class, timestamps
- `strategy_versions` — id, strategy_id, version_number, definition (JSONB), checksum, created_at. **Append-only: no UPDATE path in application code.** Unique on `(strategy_id, version_number)`
- `backtest_runs` — id, strategy_version_id, status, config (JSONB), timestamps, parent_sweep_id (nullable self-FK)
- `run_results` — run_id (PK/FK), metrics (JSONB), trade_log_ref, equity_curve_ref, computed_at
- `market_bars` — exchange, symbol, timeframe, ts, open/high/low/close/volume (all `BIGINT`, fixed-point), vendor_id, ingested_at
- `instruments` — symbol, exchange, tick_size, lot_size, price_scale
- `ingest_jobs` — for BASIS-9's idempotency tracking

Critical details:
- `market_bars` **partitioned by month** on `ts`, with a **BRIN index** on `(symbol, ts)` — BRIN, not B-tree, because bar data is naturally time-ordered and BRIN is orders of magnitude smaller.
- Unique constraint on `market_bars (vendor_id, exchange, symbol, timeframe, ts)` — this is what makes ingestion idempotent.
- All monetary/price columns `BIGINT`. **No `FLOAT`, `REAL`, or `DOUBLE PRECISION` anywhere in the schema.**
- `sqlc` configured; queries hand-written in `/server/internal/*/queries.sql`.

### Acceptance criteria
- **Given** a clean database, **when** `migrate up` runs, **then** all tables and indexes exist.
- **Given** `migrate down` then `up`, **then** the schema is identical (verify via `pg_dump --schema-only` diff).
- **Given** an attempted duplicate bar insert, **then** the unique constraint rejects it.
- **Given** `sqlc generate`, **then** type-safe Go is produced and compiles.

### Deliverables
- ADR-007: Postgres-only persistence; documented triggers for reconsidering ClickHouse.

---

## BASIS-6 — One-command local development environment

**Type:** Task · **Points:** 3 · **Priority:** P0 · **Depends on:** BASIS-1, BASIS-5 · **Component:** infra

### Requirements
- `docker-compose.yml`: Postgres 16 with a seeded dev database and a persistent volume.
- `make dev`: starts Postgres, runs migrations, starts the Go server, starts the Vite dev server.
- `make seed`: loads a small fixture dataset (a few hundred bars) for local work.
- `GET /healthz` returns 200 with build version and DB connectivity status.
- `.env.example` documenting every required variable.

### Acceptance criteria
- **Given** a clean machine with Docker and Go installed, **when** `make dev` runs, **then** `curl localhost:8080/healthz` returns 200 within 60 seconds.
- **Given** `make seed`, **then** `market_bars` contains the fixture rows.

---

## BASIS-7 — Proto contracts for the engine boundary

**Type:** Story · **Points:** 5 · **Priority:** P0 · **Depends on:** BASIS-1, BASIS-3 · **Component:** proto

> As a developer, I want the Go↔C++ contract defined once and generated for both sides, so the two can never silently drift.

### Context
Process boundary with gRPC, **not cgo** — for crash isolation, resource control, and because the process boundary is already the service boundary when the monolith splits later.

### Requirements
`/proto/basis/engine/v1/engine.proto`:

```protobuf
service BacktestEngine {
  rpc RunBacktest (BacktestJobRequest) returns (stream BacktestJobEvent);
  rpc HealthCheck (HealthRequest) returns (HealthStatus);
}
```

- `BacktestJobRequest`: run_id, graph (bytes/JSON), instrument, timeframe, start_ts, end_ts, capital, `ExecutionModelSpec`, `FeeModelSpec`, seed.
- `ExecutionModelSpec` / `FeeModelSpec`: `{ string type; map<string,int64> params; }` — **the typed-plugin-params pattern.** A new model must add zero fields to any shared message.
- `BacktestJobEvent`: `oneof { Progress, TradeFill, Completed, JobError }`.
- `TradeFill` carries a **structured** reason: `{ ReasonCode code; int64 p0; int64 p1; }` — not a formatted string. Text is rendered in Go at the edge.
- All prices/quantities as `int64` fixed-point, matching BASIS-3.
- Codegen for Go (`protoc-gen-go`, `protoc-gen-go-grpc`) and C++ wired into the build.

### Acceptance criteria
- **Given** `buf generate`, **then** Go and C++ stubs are produced and both compile.
- **Given** a `.proto` change removing or renumbering a field, **then** `buf breaking` fails CI.
- **Given** the `ReasonCode` enum, **then** it includes at minimum: `FILLED_AT_TOUCH`, `PARTIAL_VOLUME_CAP`, `SLIPPED_IMPACT`, `REJECTED_INSUFFICIENT_LIQUIDITY`.

### Technical notes
Structured reasons (rather than prose) keep the hot loop allocation-free, make reasons queryable and aggregatable, and make them translatable for the India market later.

### Deliverables
- ADR-004: process boundary + gRPC rather than cgo.

---

## BASIS-8 — `IMarketDataAdapter` interface and capability negotiation

**Type:** Story · **Points:** 5 · **Priority:** P0 · **Depends on:** BASIS-5 · **Component:** server/ingest

> As a developer, I want one interface per data vendor so that adding a vendor never touches ingestion or engine logic.

### Requirements
In `/server/internal/ingest`:

```go
type MarketDataAdapter interface {
    Fetch(ctx context.Context, req FetchRequest) ([]domain.CanonicalBar, error)
    Capabilities() VendorCapabilities
    ID() string
}

type VendorCapabilities struct {
    Timeframes         []string   // {"1m","5m","15m","1d"}
    HasTickData        bool
    HasCorporateActions bool
    HistoryYears       int
    Exchanges          []string
}
```

- A registry mapping vendor ID → adapter, with **explicit registration** in one place (not import-side-effect magic — explicit is greppable and debuggable).
- `Capabilities()` is checked **before** a fetch is attempted: requesting a timeframe a vendor doesn't support returns a specific pre-flight error, never a partial ingest.
- `domain.CanonicalBar` uses fixed-point `int64` fields matching BASIS-3.
- Adapters must never touch the database — they return bars, the writer persists them. Keeps them trivially testable.

### Acceptance criteria
- **Given** a request for a timeframe not in `Capabilities().Timeframes`, **then** a typed `ErrUnsupportedTimeframe` is returned before any network call.
- **Given** a stub adapter registered in tests, **then** the ingestion pipeline works end-to-end without a real vendor.
- **Given** a second adapter added, **then** the diff touches only the new adapter file plus one registration line.

---

## BASIS-9 — First vendor adapter and idempotent ingestion writer

**Type:** Story · **Points:** 8 · **Priority:** P0 · **Depends on:** BASIS-8 · **Component:** server/ingest

> As Priya, I want historical NIFTY/BANKNIFTY data available in the platform, so that I can backtest without sourcing data myself.

### Requirements
- Implement one real adapter. Vendor choice is yours; pick one with a free tier covering Indian equity/index data at daily and intraday resolution. Document the choice and its limits in the adapter's header comment.
- Normalise vendor response → `CanonicalBar`, converting float prices to fixed-point via `instruments.price_scale` **at the boundary, once**.
- Enforce the interval-close timestamp convention from BASIS-4 — if the vendor stamps bars at interval-open, shift during normalisation and note it in the adapter comment.
- Writer batches inserts with `ON CONFLICT DO NOTHING` on the unique constraint from BASIS-5.
- Track ingestion in `ingest_jobs`: requested range, rows written, rows skipped, vendor, completed_at.
- CLI entry point: `basis-server ingest --vendor=X --symbol=BANKNIFTY --timeframe=15m --from=2025-01-01 --to=2025-06-30`.

### Acceptance criteria
- **Given** a symbol, timeframe, and date range, **when** ingestion runs, **then** bars are persisted with correct fixed-point values.
- **Given** the same command run twice, **then** the second run writes zero new rows and reports the skip count — **no duplicates**.
- **Given** a vendor API error mid-range, **then** already-fetched bars are committed, the job is marked failed with the failure point, and a re-run resumes without duplicating.
- **Given** 6 months of 15m BANKNIFTY data, **then** the row count matches the expected trading-session count within a documented tolerance (holidays accounted for).
- Prices spot-checked against a second public source for at least 5 random bars.

### Out of scope
Corporate actions (separate ticket), tick data, options chains.

---

## BASIS-10 — Golden-file contract test harness

**Type:** Story · **Points:** 3 · **Priority:** P0 · **Depends on:** BASIS-9 · **Component:** server/ingest

> As a developer, I want recorded vendor payloads replayed in CI, so that a vendor silently changing their response format fails the build rather than corrupting a backtest.

### Context
Third-party APIs change shape without notice. Without this, the failure surfaces as subtly wrong backtest numbers weeks later — the worst possible failure mode for this product.

### Requirements
- Capture 3–5 real vendor responses into `/fixtures/vendors/{vendor_id}/` as JSON. Redact any API keys. Include at least one edge case: a holiday gap, a half-day session, or a malformed/partial response.
- A test harness that feeds each fixture through the adapter and asserts the exact resulting `[]CanonicalBar` against a committed expected output.
- Expected outputs are committed files, not inline literals — diffs must be reviewable.
- A documented `make record-fixture` path for adding new ones.

### Acceptance criteria
- **Given** the committed fixtures, **when** `go test ./internal/ingest/...` runs, **then** every fixture round-trips to its exact expected `CanonicalBar` set.
- **Given** an intentional one-field change to an adapter's parsing, **then** the fixture test fails with a readable diff.
- **Given** the malformed-response fixture, **then** the adapter returns a typed error rather than panicking or producing partial garbage.

---

## Sprint boundaries

| | Tickets | Points | Sprint goal |
|---|---|---|---|
| **Sprint 0** | BASIS-1 … BASIS-7 | 33 | Foundations exist and are enforced by CI |
| **Sprint 1 (first half)** | BASIS-8 … BASIS-10 | 16 | Real bars from a real vendor, queryable |

Sprint 0 is above the 20-point nominal velocity because much of it is scaffolding that moves faster than feature work. **Treat these numbers as a hypothesis** — record actuals and re-baseline after Sprint 1, per the PRD's velocity plan.

### What comes next (Sprint 1, second half — not yet specified)
Corporate actions adjustment · CSV import with column mapping · the engine event loop (`BarView` consumer) · portfolio accounting · the determinism CI job.

Those get written at Sprint 1 planning, once these ten have taught you what your actual velocity is.
