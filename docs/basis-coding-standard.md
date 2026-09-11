# Basis — Coding Standard

**Status:** Ratified · **Applies to:** `/engine` (C++20), `/server` (Go), `/web` (TypeScript)
**Enforcement:** blocking in CI (see §7)

This document is normative. Deviations require a `// NOLINT` with a written justification (§7.4) or an ADR.

---

## 1. Scope note on the "German safety standard" basis

One thing to be upfront about: Deutsche Börse's internal coding standard is proprietary and not something I have access to. What follows is synthesised from three public sources that genuinely do shape German safety-critical and exchange C++:

- **AUTOSAR C++14 Guidelines** (German automotive consortium; merged into **MISRA C++:2023**) — the substantive rule set behind most "German standard" C++.
- **MISRA C++:2023** — the current unified standard.
- **Conventional low-latency exchange practice** — no allocation after init, no exceptions in the matching path, deterministic teardown.

We adopt a **curated subset** (§4), not full MISRA compliance. Full compliance requires a certified static analyser, a formal deviation process, and audit records — appropriate for a system under regulatory certification, disproportionate for this codebase today. The subset below captures the rules that actually prevent defects in *this* system. If regulatory certification ever becomes a requirement, §4 is the migration base, not a rewrite.

---

## 2. C++ Naming Convention

### 2.1 Identifiers

| Kind | Convention | Example |
|---|---|---|
| Class / struct / enum / type alias | `PascalCase` | `InstrumentContext`, `FillReason` |
| Method / free function | `PascalCase` | `GetPrice`, `ComputeSlippage` |
| Private/protected member | `m_camelBack` | `m_instrumentContext`, `m_tickSize` |
| Public member (POD/aggregate only) | `camelBack` | `ticks`, `minor` |
| Static member / file-static | `s_camelBack` | `s_modelRegistry` |
| Global (avoid; must be justified) | `g_camelBack` | `g_buildVersion` |
| Local variable | `camelBack` | `fillPrice`, `barIndex` |
| Function parameter | `camelBack` | `orderQty` |
| Constant / `constexpr` | `k_camelBack` | `k_maxDepthLevels` |
| Enum constant | `k_PascalCase` | `k_FilledAtTouch` |
| Namespace | `lower_case` | `basis::engine` |
| Template parameter | `PascalCase` | `TExecutionModel` |
| Macro (avoid) | `UPPER_SNAKE` | `BASIS_ASSERT` |
| File | `PascalCase.hpp` / `.cpp` | `InstrumentContext.hpp` |

### 2.2 Method name suffixes — normative

Return category is encoded in the method name rather than relying on `const` overload resolution. This is deliberate: it makes call sites greppable, removes ambiguity when both overloads exist, and makes accidental non-const access visible in review.

| Suffix | Returns | Example |
|---|---|---|
| *(none)* | by value, or non-const reference for mutable access | `GetPrice()`, `GetInstrumentContext()` |
| `Const` | `const&` | `GetPriceConst()`, `GetInstrumentContextConst()` |
| `Ptr` | raw non-owning pointer, may be null | `GetInstrumentContextPtr()` |
| `PtrConst` | `const*`, may be null | `GetInstrumentContextPtrConst()` |
| `Ref` | non-const reference (use when `&` vs value is non-obvious) | `GetPortfolioRef()` |

Rules:
- A method returning `Ptr` **must** document its null semantics in a one-line comment. If it can never be null, return a reference instead.
- Raw pointers are **non-owning, always**. Ownership transfer uses `std::unique_ptr`; shared ownership requires an ADR.
- Do not create both `GetX()` and `GetXConst()` unless both are actually called. One is dead code.

### 2.3 Booleans and predicates

Prefix with `Is`, `Has`, `Can`, or `Should`: `IsExpired()`, `HasSufficientLiquidity()`, `m_isActive`. Never a bare noun for a boolean.

---

## 3. C++ Error Handling

Three zones, three mechanisms. The zone a function lives in is determined by *when it runs*, not what it does.

### 3.1 Zone A — Hot path (per-bar, per-fill, per-node evaluation)

**No exceptions. No allocation. `noexcept` on every function.**

```cpp
enum class ErrorCode : int32_t {
  k_Ok = 0,
  k_InsufficientLiquidity,
  k_InvalidPrice,
  k_PositionLimitExceeded,
  // ...
};

template <typename T>
class Result {                       // hand-rolled: std::expected is C++23, we target C++20
public:
  static Result Ok(T value) noexcept;
  static Result Err(ErrorCode code) noexcept;
  bool IsOk() const noexcept;
  ErrorCode GetErrorCode() const noexcept;
  const T& GetValueConst() const noexcept;   // precondition: IsOk()
  T&& TakeValue() noexcept;
private:
  ErrorCode m_code;
  alignas(T) std::byte m_storage[sizeof(T)];
};
```

- Every hot-path function is declared `noexcept`. This is enforced, not aspirational — an exception escaping `noexcept` calls `std::terminate`, which is the correct behaviour: a hot-path exception means the invariants are already broken.
- `[[nodiscard]]` on every function returning `Result<T>`. Ignoring an error must not compile.
- No `Result<void>` — use bare `ErrorCode` as the return type.

### 3.2 Zone B — Setup and configuration (runs once per backtest, before the loop)

**Exceptions permitted**, because this is where allocation, parsing, file I/O, and graph compilation happen, and failure here aborts the run cleanly.

- Throw only types derived from `basis::EngineException`.
- Exceptions must not escape a destructor, a move operation, or a `swap` (AUTOSAR A15-5-1).
- Every `catch` either handles or rethrows with context. Never `catch(...)` with an empty body.

### 3.3 Zone C — Process boundary (gRPC handlers, `main`)

**No exception may cross the process boundary.** Every gRPC handler is wrapped:

```cpp
grpc::Status RunBacktest(...) noexcept {
  try { /* ... */ }
  catch (const EngineException& e) { return ToGrpcStatus(e); }
  catch (const std::exception& e)  { return grpc::Status(grpc::INTERNAL, e.what()); }
  catch (...)                      { return grpc::Status(grpc::INTERNAL, "unknown"); }
}
```

### 3.4 Contracts

```cpp
BASIS_PRECONDITION(cond, msg)   // caller's responsibility — active in debug, documented in release
BASIS_INVARIANT(cond, msg)      // class/loop invariant
BASIS_ASSERT_ALWAYS(cond, msg)  // active in release too — use for memory-corruption-class checks
```

Preconditions are documented in the header. A violated precondition is a **bug**, not an error condition — it never returns `ErrorCode`.

---

## 4. Adopted Safety Subset (from AUTOSAR C++14 / MISRA C++:2023)

Twelve rules, chosen because each maps to a real failure mode in this system.

| # | Rule | Why it's here |
|---|---|---|
| S-1 | No dynamic allocation after engine initialisation | Determinism + hot-loop performance (AUTOSAR A18-5-5) |
| S-2 | No `malloc`/`free`; `new`/`delete` only during setup; prefer arena/pool | A18-5-1 |
| S-3 | No RTTI, no `dynamic_cast` | See §8 conflict note — replaced by explicit interface query |
| S-4 | No C-style casts; `static_cast`/`reinterpret_cast` only, each justified | A5-2-2 |
| S-5 | `const` by default on locals, params, and methods | A7-1-1 |
| S-6 | Rule of Zero, or all five special members declared | A12-0-1 |
| S-7 | No implicit conversions on user-defined types; single-arg ctors `explicit` | A12-1-4 |
| S-8 | No recursion in the engine | Bounded stack; deterministic |
| S-9 | Every `switch` on an enum handles all cases; no `default` on closed enums | Compiler catches new enum values |
| S-10 | No `goto`, no `volatile` without an ADR | — |
| S-11 | Fixed-width integer types only (`int64_t`, not `long`) | Portability + the fixed-point contract |
| S-12 | No floating point in cash, price, position, or fill-price paths | Determinism (see technical plan §1.1) |

---

## 5. Go Standard

Deliberately thin — `gofmt` settles formatting, and Go's idioms are well-established. Divergence from standard Go is a cost with no benefit.

- `gofmt -s` and `goimports`. No custom formatting.
- Standard Go naming: `MixedCaps`, exported = capitalised. **Do not port the C++ convention** — no `m_` prefixes, no type suffixes. Consistency *within* a language beats consistency across languages.
- Errors: return `error`, wrap with `%w`, define typed sentinels (`ErrUnsupportedTimeframe`) at package boundaries. Never `panic` for control flow; a `panic` reaching an HTTP handler is a bug.
- `context.Context` is the first parameter of any function doing I/O.
- Interfaces defined by the **consumer** package, not the implementer.
- No ORM — `sqlc`-generated code from hand-written SQL.
- Linting: `golangci-lint` with `errcheck`, `govet`, `staticcheck`, `ineffassign`, `gosec` enabled.

---

## 6. TypeScript Standard

- Prettier (2-space, single quotes, trailing commas, 100 cols) — no debate, no per-file overrides.
- ESLint with `@typescript-eslint/recommended-type-checked`.
- `strict: true`. **`any` is banned** — use `unknown` and narrow.
- Naming: `PascalCase` for components and types, `camelCase` for everything else, `UPPER_SNAKE` for module constants.
- All API types generated from the OpenAPI/proto schema. Hand-written mirrors of server types are forbidden — they drift within weeks.
- Components are function components with hooks. No class components.

---

## 7. Enforcement

### 7.1 `.clang-format` (key settings)

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 110
PointerAlignment: Left          # int64_t* ptr
AccessModifierOffset: -4
AllowShortFunctionsOnASingleLine: InlineOnly
BreakBeforeBraces: Custom
BraceWrapping:
  AfterClass: true
  AfterFunction: true
  AfterControlStatement: false
SortIncludes: CaseSensitive
```

### 7.2 `.clang-tidy` naming enforcement

```yaml
Checks: >
  -*,
  bugprone-*, cert-*, cppcoreguidelines-*, modernize-*,
  performance-*, portability-*, readability-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers,
  -cppcoreguidelines-avoid-magic-numbers
WarningsAsErrors: '*'
CheckOptions:
  - { key: readability-identifier-naming.ClassCase,            value: CamelCase }
  - { key: readability-identifier-naming.StructCase,           value: CamelCase }
  - { key: readability-identifier-naming.EnumCase,             value: CamelCase }
  - { key: readability-identifier-naming.EnumConstantCase,     value: CamelCase }
  - { key: readability-identifier-naming.EnumConstantPrefix,   value: k_ }
  - { key: readability-identifier-naming.FunctionCase,         value: CamelCase }
  - { key: readability-identifier-naming.MethodCase,           value: CamelCase }
  - { key: readability-identifier-naming.PrivateMemberCase,    value: camelBack }
  - { key: readability-identifier-naming.PrivateMemberPrefix,  value: m_ }
  - { key: readability-identifier-naming.ProtectedMemberPrefix,value: m_ }
  - { key: readability-identifier-naming.PublicMemberCase,     value: camelBack }
  - { key: readability-identifier-naming.StaticVariablePrefix, value: s_ }
  - { key: readability-identifier-naming.GlobalVariablePrefix, value: g_ }
  - { key: readability-identifier-naming.LocalVariableCase,    value: camelBack }
  - { key: readability-identifier-naming.ParameterCase,        value: camelBack }
  - { key: readability-identifier-naming.ConstantPrefix,       value: k_ }
  - { key: readability-identifier-naming.NamespaceCase,        value: lower_case }
  - { key: readability-identifier-naming.TypeAliasCase,        value: CamelCase }
  - { key: readability-identifier-naming.MacroDefinitionCase,  value: UPPER_CASE }
```

### 7.3 CI gates (blocking)

| Gate | Scope |
|---|---|
| `clang-format --dry-run --Werror` | `/engine` |
| `clang-tidy` with `WarningsAsErrors: '*'` | `/engine` |
| `gofmt -l` empty + `golangci-lint run` | `/server` |
| `prettier --check` + `eslint --max-warnings 0` + `tsc --noEmit` | `/web` |
| `buf lint` + `buf breaking` | `/proto` |

Compiler flags: `-Wall -Wextra -Wpedantic -Wconversion -Wshadow -Wnon-virtual-dtor -Werror`.

### 7.4 Suppression policy

```cpp
// NOLINTNEXTLINE(cppcoreguidelines-pro-type-reinterpret-cast): arena placement,
// alignment guaranteed by ArenaAllocator::Allocate. See ADR-012.
```

A bare `// NOLINT` fails review. Every suppression names the specific check and gives a reason. A grep for `NOLINT` is a monthly review item — if a check is suppressed more than five times, either the check is wrong for this codebase (disable it globally, with an ADR) or the design is fighting it.

---

## 8. Conflicts With Earlier Decisions — resolved here

Three things in the technical dev plan don't survive contact with this standard. Flagging them explicitly so they don't surface as surprises mid-sprint.

**8.1 `dynamic_cast` for interface versioning → replaced.**
The technical plan §5.3 proposed `dynamic_cast` to detect whether a plugin implements `IExecutionModelV2`. Rule S-3 bans RTTI. Replace with an explicit query, which is also faster and ABI-stable:

```cpp
class IExecutionModel {
public:
  virtual void* QueryInterface(InterfaceId id) noexcept = 0;   // nullptr if unsupported
  virtual uint32_t GetInterfaceVersion() const noexcept = 0;
};
```

This is the COM/Vulkan pattern and is strictly better here than RTTI.

**8.2 `std::expected` is C++23; we target C++20.**
Hand-roll `Result<T>` (§3.1, ~80 lines, fully tested) rather than bumping the language standard — bumping constrains the toolchain for one convenience type. Revisit at C++23 migration.

**8.3 Method-name suffixes can't be machine-enforced.**
`clang-tidy` enforces case and prefixes; it cannot verify that `GetPriceConst()` returns `const&`. This goes on the review checklist (§9) and is the one part of the naming standard that depends on human discipline.

---

## 9. Review Checklist

Paste into the PR template:

```
Correctness
- [ ] No floating point in cash / price / position / fill paths
- [ ] Hot-path functions are noexcept; no allocation added to the bar loop
- [ ] Result<T> returns are [[nodiscard]] and checked at every call site
- [ ] No unordered-container iteration affecting output (determinism)

Naming
- [ ] Members m_, statics s_, constants k_
- [ ] Const-returning methods carry the Const suffix; pointer-returning carry Ptr
- [ ] Every Ptr-returning method documents null semantics

Extensibility
- [ ] No switch on model/adapter/node type — registry used instead
- [ ] New config fields use the {type, params} plugin pattern, not a new struct field

Hygiene
- [ ] Every NOLINT names its check and gives a reason
- [ ] New public headers documented; ADR added if a decision was made
```

---

## Appendix — ADRs generated by this document

- **ADR-008** — C++ naming convention (`m_` members, PascalCase methods, type-encoding suffixes)
- **ADR-009** — Three-zone error handling model; `Result<T>` over exceptions in the hot path
- **ADR-010** — Adopted AUTOSAR/MISRA safety subset (12 rules) and the rationale for not pursuing full compliance
- **ADR-011** — Strict, merge-blocking CI enforcement and the NOLINT justification policy
