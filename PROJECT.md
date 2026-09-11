# Basis: standing context

**What it is:** A strategy design and backtesting platform where realistic execution (fills, fees, slippage) is modelled by default, and every fill carries a stated reason. Bar-level, single-instrument, single-tenant in R1.
**Who it's for:** Retail and semi-pro traders who don't trust their backtests; honest enough for a quant.

## Stack
- `/engine`: C++20, CMake presets (`debug`, `release`, `asan`, `tsan`, `ubsan`). No exceptions/allocation on the hot path.
- `/server`: Go, single binary, modular monolith (`api/ strategy/ ingest/ runner/ results/`). Postgres only; job queue is `FOR UPDATE SKIP LOCKED`.
- `/proto`: gRPC contracts, `buf`. Single source of truth for the C++/Go seam.
- `/web`: TypeScript/React, React Flow canvas.
- Engine ↔ server is a separate process over streamed gRPC. Not cgo, not FFI.

## Invariants
- Money, price, quantity are `int64` fixed-point in distinct types. `double` is for derived analytics only, never cash/position/fill.
- Time is `int64` ns since epoch, UTC. Bars are stamped interval-close. Timezone is a presentation concern outside the engine.
- Strict event time-ordering; look-ahead prevention is architectural, not a test.
- Bit-for-bit deterministic: no `-ffast-math`, no unordered iteration affecting output, every RNG seeded from the run manifest, fixed `-march` baseline. CI runs a fixture twice and diffs.
- Every fill has a `reason`. `strategy_versions` are immutable and checksummed.
- Hot path (per-bar/fill/node): `noexcept`, no allocation, error codes. Setup: exceptions derived from `EngineException`. Process boundary: nothing escapes a gRPC handler.
- No cross-package struct sharing in `/server`. Never hand-write a struct mirroring a proto message.
- No Kafka, ClickHouse, WASM sandbox, or compiler service in R1 (PRD D3, D5, D6). Design direction is locked (D4).

## Conventions
- C++: `PascalCase` types and methods, `m_` members, `k_` constants, `basis::` namespaces, `PascalCase.hpp/.cpp`. Full table in `docs/basis-coding-standard.md` §2.
- Flags: `-Wall -Wextra -Wpedantic -Wconversion -Wshadow -Wnon-virtual-dtor -Werror`. `clang-tidy` warnings are errors.
- Anything touching fills, orders, or money needs a fixture test with hand-verified expected values. Anything touching a registry passes its conformance suite.
- Every PRD §4 decision and dev-plan §1 rule gets an ADR in `docs/adr/`. ADR beats memory.
- Run tests: *not yet, Sprint 0 deliverable*
- Run lint/sanitisers: *not yet, Sprint 0 deliverable*

## Where things live
- `docs/`: PRD, technical dev plan, coding standard, Sprint 0 backlog. PRD wins on conflict.
- `docs/adr/`: numbered, immutable decision records.
- `ledger/`: STATE.md, LESSONS.md. Session handoff lives here, nowhere else.
- `/fixtures`: recorded vendor payloads, golden outputs, hand-verified P&L cases.
- `/engine /server /proto /web`: see Stack. None exist yet.
