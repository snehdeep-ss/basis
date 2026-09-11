# Basis — Technical Development Plan

**Audience:** the engineer building this. **Stack:** C++20 engine · Go services · TypeScript/React UI.

This is not a feature plan — the PRD has that. This is the set of engineering decisions that are expensive to change later, ordered roughly by how expensive. Everything here optimises for one property: **adding a new execution model, data vendor, broker, node type, or asset class should be additive.** If a change requires editing a `switch` statement in the hot loop or a struct that everything else includes, that's a design bug, and the time to catch it is now.

---

## 1. The Correctness Foundations

These are the decisions that, if made wrong, produce a backtester that is subtly and permanently wrong. They come first because every other layer inherits them.

### 1.1 Never represent money or prices as `double`

A `double` cannot represent `0.1` exactly. Accumulate P&L over 50,000 fills and you will have drift that no test catches and every user eventually notices. Worse, `a + b + c != c + b + a`, which silently breaks reproducibility.

```cpp
// Fixed-point: int64 minor units scaled by a per-instrument factor.
struct Price { int64_t ticks; };     // in instrument tick units
struct Money { int64_t minor; };     // paise / cents
struct Qty   { int64_t units; };     // lots or shares, never fractional unless the venue allows it
```

Use `double` only for *derived analytics* — Sharpe, correlation, indicator values — never for cash, position, or fill prices. Convert at the boundary, once, explicitly. Wrap them in distinct types (above) so the compiler prevents adding a `Price` to a `Qty`; that mistake is otherwise invisible and common.

### 1.2 Time is `int64` nanoseconds since epoch, UTC, always

One representation, no exceptions. Timezone is a *presentation* concern handled in Go or TS, never in the engine. Session boundaries (IST market open, DST-affected US sessions) are resolved during ingestion into UTC nanosecond ranges and stored that way — the engine never asks "what timezone is this."

Bar timestamps are **interval-close by convention** (a 15m bar stamped 09:30 covers 09:15–09:30). Write this in a header comment and enforce in the ingestion adapters, because vendors disagree about this and it is a classic source of an off-by-one-bar look-ahead bug.

### 1.3 Determinism is a hard requirement, not a nice property

Reproducibility ("re-run this backtest from six months ago, get the identical number") is a product promise. It constrains code:

- **No `-ffast-math`, ever.** It permits reassociation and breaks bit-reproducibility across builds.
- **Never iterate an unordered container** when the iteration affects output. Use `std::map`, sorted `vector`, or iterate an explicit index order. Hash iteration order varies across runs and libc++/libstdc++.
- **All randomness takes an explicit seed** stored in the run manifest (§6.3). Monte Carlo, walk-forward shuffles, fill-probability sampling — all of it.
- **Pin the floating-point environment**: no changes to rounding mode, and be aware that `-march=native` can change results via FMA contraction. Build engine release artifacts with a fixed `-march` baseline.
- **CI job that runs a fixture backtest twice and diffs the output byte-for-byte.** This catches determinism regressions the day they're introduced, which is the only time they're cheap to find.

### 1.4 Look-ahead prevention is structural, not tested

Don't give the strategy an API that *can* see the future and then test that it doesn't. Give it an API where the future is unreachable:

```cpp
class BarView {                      // handed to node evaluation
  const Bar* data_;  std::size_t i_; // current index
public:
  const Bar& at(std::size_t back) const {   // back=0 is current, back=1 is previous
    assert(back <= i_);
    return data_[i_ - back];
  }
  // there is deliberately no forward accessor, and no raw pointer escape
};
```

The engine advances `i_`; nodes only ever index backwards. A cheating strategy becomes uncompilable rather than untested. This is the single highest-leverage twenty lines in the codebase.

---

## 2. Repository & Build Topology

```
/engine        C++20. CMake + presets. The simulator, execution models, node kernels.
  /include/basis/    public headers (the extension surface)
  /src/                 implementation
  /tests/               unit, property, golden
/proto         .proto contracts. Single source of truth. buf for lint/breaking-change checks.
/server        Go. Modular monolith (PRD D2). Owns HTTP, persistence, orchestration.
/web           TypeScript/React. React Flow canvas.
/fixtures      Recorded vendor payloads, golden backtest outputs, hand-verified P&L cases.
/docs/adr      Architecture Decision Records. One file per decision, numbered, immutable.
```

**Contract-first.** `/proto` is the interface between C++ and Go. Generate both sides; never hand-write a struct that mirrors a proto message. Run `buf breaking` in CI against the main branch — a breaking proto change should fail the build, not surface as a runtime deserialisation error six weeks later.

**ADRs from day one.** Every decision in this document and the PRD's §4 gets a numbered ADR. Cost: 15 minutes each. Value: in month eight, when you wonder why prices are fixed-point, the answer exists and you don't re-litigate it. Also the highest-signal artifact you can show an interviewer.

**Build:** CMake presets (`debug`, `release`, `asan`, `tsan`, `ubsan`). Sanitiser builds are not optional extras — the engine is the one place where a memory bug corrupts financial output silently. Run ASan+UBSan on the full test suite in CI nightly.

---

## 3. The C++/Go Seam

**Decision: separate process, gRPC, streamed. Not FFI/cgo.**

Rationale, in priority order:
1. **Crash isolation.** A segfault in strategy evaluation kills one backtest worker, not the API server serving every user.
2. **cgo is a genuine tax** — it defeats Go's scheduler on blocking calls, complicates cross-compilation, and makes stack traces unreadable across the boundary.
3. **Independent deployability later** costs nothing extra: the process boundary is already the service boundary when you split (PRD D2's migration path).
4. **Resource control**: cgroup/rlimit a worker process for CPU and memory. You cannot meaningfully bound a cgo call.

The cost is serialisation overhead per job, which is irrelevant — jobs are seconds-to-minutes long and events stream. If profiling ever shows the boundary is hot (it won't), shared memory for the bar buffer is the escape hatch, not cgo.

**Worker lifecycle:** the Go `runner` module owns a pool of engine subprocesses. Health-check via a gRPC `HealthCheck`; kill and respawn on timeout. Each job gets a hard wall-clock deadline and a memory cap — a user's pathological graph must not take the box down.

---

## 4. Engine Internals

### 4.1 Data layout

Bars arrive as a columnar block, not an array of structs. The hot loop touches close prices far more than it touches volume:

```cpp
struct BarSeries {                 // struct-of-arrays; cache-friendly
  std::vector<int64_t> ts;
  std::vector<int64_t> open, high, low, close;   // fixed-point ticks
  std::vector<int64_t> volume;
  // extension: named extra columns, sparse, absent by default
  std::unordered_map<std::string, std::vector<int64_t>> attrs;
};
```

The `attrs` map is the extensibility hatch from the design phase (notional volume, OI, spread) — **it lives outside the hot path**. Nodes that need it resolve their column pointer *once at graph-compile time*, not per bar. Never do a string lookup inside the bar loop.

### 4.2 The hot loop rule

**Zero heap allocation between bar 0 and bar N.** Everything the run needs — node state, output buffers, order objects, fill records — is allocated during graph setup from an arena and reused. Enforce it:

- Node state is a pre-sized `std::vector<NodeState>` indexed by node ID, not a map.
- Orders and fills come from a fixed-capacity pool with intrusive free lists.
- In debug builds, override `operator new` to `assert(!in_hot_loop)`. This turns "we should avoid allocating" into a test that fails.

### 4.3 The `reason` string trap

Every fill carries a human-readable reason — the product's core promise. Naively that's a `std::string` format call per fill, which is an allocation per fill in the hot loop.

Instead, emit a structured code plus typed parameters, and render text at the *edge*:

```cpp
struct FillReason {
  ReasonCode code;          // enum: FILLED_AT_TOUCH, PARTIAL_VOLUME_CAP, SLIPPED_IMPACT, ...
  int64_t p0, p1;           // fixed-point params, meaning per code
};
```

Go (or the UI) renders `PARTIAL_VOLUME_CAP{p0=40}` → "partial fill: 40% of average 1-minute volume". Benefits beyond speed: reasons become **queryable and aggregatable** ("show me every run where >20% of fills hit the volume cap"), and they're translatable for the India market later. This is a small decision that gets much more expensive after a million fills are stored as prose.

### 4.4 Concurrency

**One backtest run = one thread. No exceptions.** Parallelism comes from many independent runs across the worker pool (a sweep is N jobs). This is not a compromise — it's what makes §1.3 determinism achievable at all. Multi-threading inside a single book/portfolio introduces nondeterministic interleaving that destroys reproducibility for a speedup you don't need.

The engine process may host multiple concurrent runs on separate threads, but they share **nothing mutable** — no global registry mutation after startup, no shared caches without immutability.

---

## 5. Extension Points — Doing the Registry Pattern Correctly in C++

This is the part most designs get right on paper and wrong in code. Three specific traps:

### 5.1 Static initialisation order fiasco

A namespace-scope registry object may not be constructed when a translation unit's self-registration static runs. Use a function-local static (constructed on first use, thread-safe since C++11):

```cpp
class ExecutionModelRegistry {
public:
  static ExecutionModelRegistry& instance() {   // NOT a global object
    static ExecutionModelRegistry r;
    return r;
  }
  void add(std::string id, Factory f);
  std::unique_ptr<IExecutionModel> create(std::string_view id) const;
};
```

### 5.2 The linker drops your registrations

If models live in a static library and nothing references them, **the linker discards the object file and the registration never runs.** The symptom is maddening: works in a test binary, silently missing in production. Options, in order of preference:

1. Build extension modules into an object library (`add_library(... OBJECT)`) linked directly into the target — objects are not garbage-collected.
2. `-Wl,--whole-archive` on the models archive (`-force_load` on macOS).
3. An explicit `register_builtin_models()` called from `main()` — least elegant, most obvious, zero surprises. **Start here.** The magic self-registration macro is a convenience you add once the pattern is proven; explicit registration is easier to debug and grep.

### 5.3 Interface versioning across an ABI

The moment external plugins load as shared objects, adding a virtual method to `IExecutionModel` reorders the vtable and breaks every compiled plugin — with no compile error, just undefined behaviour at runtime.

Rules: **never add, remove, or reorder virtuals on a shipped interface.** Extend by inheritance (`IExecutionModelV2 : IExecutionModel`) and query with `dynamic_cast`. Pass a version integer in the factory. Keep POD structs at the boundary — no `std::string`/`std::vector` across a DLL boundary (allocator and layout mismatches).

Until you actually ship third-party plugins, **build everything in-tree** and skip the ABI tax entirely — but write the interfaces as if you will, because retrofitting is the expensive direction.

### 5.4 The four extension surfaces

| Surface | Interface | Registered by | Owner |
|---|---|---|---|
| Execution realism | `IExecutionModel` (statistical / matching engine) | string ID from run config | Engine |
| Fee | `IFeeModel` | string ID | Engine |
| Node kernel | `INodeKernel` (evaluate + param schema) | node type ID | Engine |
| Data vendor | `IMarketDataAdapter` | vendor ID | Go |
| Broker | `IBrokerAdapter` | broker ID | Go |

Node kernels are the surface that will grow fastest (22 → 60 → 90+). Design its registration to be boring and mechanical: one file per kernel, a param schema declared next to the implementation so they cannot drift, and a codegen step that emits the TypeScript palette entry from the same schema. **One source of truth for a node's params, three consumers** (engine validation, UI form generation, optimiser search space).

---

## 6. Testing Strategy

The normal pyramid is wrong here. A backtester's failure mode isn't crashing — it's producing a plausible wrong number. Weight tests accordingly.

### 6.1 Hand-verified fixtures (highest value)

A handful of tiny scenarios — 10 bars, 2 trades — where you computed the expected P&L, fees, and slippage **by hand on paper** and committed the expected output. These are the only tests that can tell you the engine is *right* rather than merely *consistent*. Every fee/slippage/accounting change must keep them green.

### 6.2 Property tests

Invariants that must hold for any input, tested with generated data:
- Cash + position value == equity, at every bar.
- Sum of fill quantities == submitted quantity (nothing created or lost).
- No fill timestamp precedes its order's arrival timestamp.
- Order book (when built) never crosses: `best_bid < best_ask`.
- Slippage sign is never favourable beyond a configured bound.

### 6.3 Golden-master + run manifests

Store full outputs of reference backtests. Any diff must be *explained* in the PR, not silently accepted. This is what catches an "innocuous" refactor changing results in the fourth decimal.

Every run emits a **manifest** — the reproducibility contract:

```json
{ "engine_version":"1.4.2+git.a3f9c1", "graph_checksum":"sha256:…",
  "data_snapshot":"nse.banknifty.15m.2025-01..2025-06#sha256:…",
  "execution_model":"volume_impact@2", "fee_model":"flat_pct@1",
  "seed": 42, "fp_baseline":"x86-64-v2" }
```

If two runs share a manifest and differ in output, that is a P0 bug. This also makes "why did my number change?" answerable in seconds instead of a day of bisecting.

### 6.4 Differential testing

When the compiled graph path lands (PRD D3), run the interpreter and compiler over the same graphs and assert identical output. Same technique later for statistical model vs matching engine — they should agree in *aggregate* even where individual fills differ; systematic divergence means one has a bug.

### 6.5 Fuzzing

Fuzz the graph validator (`libFuzzer`) with malformed/adversarial graph JSON. It is a parser accepting user input; it will be handed cycles, dangling refs, absurd depths, and NaN-ish params. Better to find that in CI.

---

## 7. Go Service Design

**Hexagonal, boring, no framework.** `net/http` + `chi` for routing. **No ORM** — use `sqlc` to generate type-safe Go from hand-written SQL. You will be writing time-series queries with partition pruning and window functions; an ORM actively fights that, and the generated-from-SQL approach keeps the query the source of truth.

**Package layout mirrors PRD §5 modules**, with strict rules: no package imports another's internal types; communication is via interfaces defined by the *consumer*; `domain` types are shared and dependency-free.

**Job queue in Postgres** (PRD D6) — `SELECT … FOR UPDATE SKIP LOCKED` is a correct, well-understood queue up to thousands of jobs/minute. Add a `attempts` column, an exponential backoff, and a dead-letter state from day one; retry semantics retrofitted later are always wrong.

**Errors:** typed sentinel errors at package boundaries, wrapped with `%w`, mapped to the single API error shape (`{code, message, field}`) in one place — the HTTP layer. Never let a `pq: duplicate key` string reach a user.

**Context propagation everywhere.** Every engine job carries a `context.Context` with the run deadline; cancellation must actually kill the subprocess, not orphan it. Test this explicitly — orphaned workers are the classic way a box quietly runs out of RAM at 3am.

---

## 8. Observability

Three things, from the first sprint, because none can be reconstructed retroactively:

1. **Structured logs** (`slog`, JSON) with `run_id` on every line across Go *and* engine stderr. One grep reconstructs a run's full life.
2. **Metrics** (Prometheus): backtest wall-clock p50/p95, bars-per-second throughput, queue depth, worker restarts, data-load latency. These are the numbers that answer PRD decisions D3 and D6 with evidence instead of opinion — that's their real job, not dashboards.
3. **A performance budget, asserted in CI.** Pick a number now (e.g. *6 months of 15m bars, 20-node graph, < 2s*), write a benchmark, and fail the build on >20% regression. Performance defended continuously is cheap; performance recovered after six months of drift is a project.

---

## 9. Sequencing (maps to PRD sprints)

| Sprint | Engineering focus | Foundational thing that must land |
|---|---|---|
| S0 | Repo, CMake presets, buf, sqlc, ADR 001–007 | Fixed-point + time types (§1.1, §1.2) |
| S1 | Ingestion, `IMarketDataAdapter`, golden fixtures | Bar convention documented and enforced |
| S2 | Engine loop, portfolio accounting, gRPC seam | `BarView` (§1.4) + determinism CI job |
| S3 | Graph interpreter, node kernel registry | Explicit registration (§5.2) + param-schema single source |
| S4 | Canvas UI, first 22 kernels, results chart | Schema→TS codegen for the palette |

**Do not defer §1 to "clean up later."** Changing price representation after 40 files depend on `double` is a multi-week refactor with silent-corruption risk. Changing it in S0 is an afternoon.

---

## 10. Traps to Avoid

- **A `switch` on model type inside the engine.** If one appears, the registry isn't being used. Grep for this in review.
- **Config structs that grow a field per feature.** Use the typed-plugin-params pattern (`{type, params{}}`) so a new model adds no field to a shared struct.
- **String IDs without a registry check at validation time.** A typo'd model ID must fail before the job is queued, not after 90 seconds of compute.
- **Letting the UI define node semantics.** The engine's kernel + param schema is authoritative; the palette is generated from it. Two hand-maintained lists will diverge within a month.
- **Premature distribution.** Kafka, ClickHouse, and service splits are all *reversible* additions later and *irreversible* time sinks now (PRD D2/D6).
- **Skipping the run manifest.** It's an hour of work and it's the difference between "I can prove this result" and "I think it changed when we upgraded the compiler."

---

## Appendix — First Seven ADRs

1. Fixed-point money and price representation
2. UTC nanosecond time; interval-close bar convention
3. Determinism requirements and the FP build baseline
4. Process boundary + gRPC between Go and C++ (not cgo)
5. Modular monolith for v1; module boundary rules
6. Registry pattern with explicit registration; interface versioning policy
7. Postgres-only persistence; triggers for introducing ClickHouse
