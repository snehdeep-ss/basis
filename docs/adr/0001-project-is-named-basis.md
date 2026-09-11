# 0001: The project is named Basis

**Status:** accepted
**Date:** 2026-09-11

## Context
The design-phase documents (PRD, technical dev plan, coding standard, Sprint 0 backlog) were written under the working name "Slippage". That word is also the core domain concept the product models (slippage models, per-fill slippage, `ISlippageModel`), so using it as the product name makes every sentence ambiguous and every grep useless.

## Decision
The product, repository, and all code identifiers derived from the product name use **Basis**:
- C++ namespace `basis::`, public headers under `/engine/include/basis/`, macro prefix `BASIS_`
- Go binary `basis-server`, under `/server/cmd/basis-server/`
- Proto package path `/proto/basis/...`
- Ticket prefix `BASIS-n`
- Document filenames `docs/basis-*.md`

"Slippage" remains the domain term and is unchanged wherever it means the concept.

## Consequences
- Design-phase source files cited in PRD §14 keep their historical `slippage-*` names; they are external references, not part of this repo.
- Any future doc or code introducing `slippage` as a product identifier is a review defect.
