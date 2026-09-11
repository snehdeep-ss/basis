# Templates

Copy these into the repo. Keep them short. Every one of these files is read often, and length is the enemy.

---

## PROJECT.md

```markdown
# <Project>: standing context

**What it is:** <two sentences, from the user's side>
**Who it's for:** <one sentence>

## Stack
<language/std, build system, key deps, target platforms>

## Invariants
- <things that must never break, e.g. "no allocation on the match path">
- <e.g. "public API is stable from v0.3 onward">

## Conventions
- <naming, error handling strategy, logging, test layout>
- Run tests: <command>
- Run lint/sanitisers: <command>

## Where things live
<3-5 lines, directory map only>
```

Under 60 lines. No task state, no dates, no history. Prune at every retro.

---

## ledger/STATE.md

```markdown
# State: <date>, session <n>

**Open ticket:** <id>, <title>, <status>
**Branch:** <name>, <n> commits ahead of main

## Where things stand
<2-4 lines. What actually works right now, what's half-done.>

## Tried and failed
- Tried <X>, failed because <Y>.

## Corrections (binding)
- <preferences, rejected options, decisions I got wrong and was corrected on>

## Next action
<one specific thing, startable without a decision>

## Open questions
- <unresolved, needs a call>
```

Overwrite each session. Five to fifteen lines. If it's longer, it's a transcript.

---

## docs/adr/NNNN-slug.md

```markdown
# NNNN: <decision, as a statement not a question>

**Status:** accepted | superseded by NNNN
**Date:** <date>

## Context
<what forced a decision. 3-5 lines.>

## Options considered
1. <option>: <why not>
2. <option>: <why not>

## Decision
<what was chosen>

## Consequences
<what this now costs, what it forecloses, what has to be revisited if X changes>
```

Written by the engineer. The consequences section is the one that matters in six months.

---

## Ticket

```markdown
# <ID>: <imperative title, no "and">

**Epic:** <epic> | **Est:** <1-2 evenings>

## Why
<user-visible or operational reason>

## Context
<what this touches, constraining ADRs, what was tried before>

## Acceptance criteria
1. <testable, observable>
2. ...

## Out of scope
- <explicit>

## Definition of Done
- [ ] AC met and demonstrated
- [ ] Tests fail when the implementation is broken
- [ ] CI green on a clean checkout
- [ ] One log line or metric an operator could use
- [ ] No new warnings; sanitisers clean
- [ ] Non-obvious decisions recorded as ADRs
- [ ] ledger/STATE.md updated
- [ ] Reviewed and APPROVED
```

---

## Design note (before any non-trivial branch)

```markdown
# Design: <ticket id>

## Problem
<what, in one paragraph, from the system's side>

## Approach
<the shape. Components, responsibilities, data flow. Prose and boxes, not code.>

## Alternatives rejected
- <option>: <why>

## Failure modes
- <what happens when X hangs / restarts mid-op / receives garbage>

## Observability
<what an operator sees when this is healthy, and when it isn't>

## What I'm unsure about
<be honest here; this is the part Anton reads first>
```

The last section is not optional. A design note with nothing under it is a design note that hasn't been thought about.

---

## ledger/REVIEWS.md (append-only)

```markdown
| date | ticket | verdict | corr | design | ops | tests | craft | key finding |
|---|---|---|---|---|---|---|---|---|
```

## ledger/LESSONS.md (append-only)

```markdown
- <date>: Tried <X>, failed because <Y>.
```
