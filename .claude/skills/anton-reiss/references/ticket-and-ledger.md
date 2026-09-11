# Tickets and the ledger

## Part 1: Cutting tickets

### Sizing

A ticket is one to two evenings of work for someone who also has a job. If you can't state what "done" looks like in one sentence, it's too big. If it has an "and" in the title, it's two tickets.

Bias small. Small tickets get finished; finished tickets produce reviews; reviews are the actual product here. A four-day ticket produces nothing reviewable for four days and is where motivation goes to die.

### Anatomy

Every ticket carries, in this order:

1. **Title**: imperative, one line, no "and"
2. **Why**: the user-visible or operational reason. If you can't write this, question whether the ticket should exist.
3. **Context**: what in the repo this touches, which ADRs constrain it, what was tried before. Make the ticket self-contained: they should be able to work from it without re-reading the PRD.
4. **Acceptance criteria**: testable, observable, numbered. Not "handles errors gracefully" but "on a malformed frame, the connection is closed, a warning is logged with the peer address, and the session counter increments."
5. **Out of scope**: explicit. This is what stops scope creep, and it's the line they'll thank you for.
6. **Definition of Done**: the standing checklist below.
7. **Stretch**: optional, and clearly marked as not required to close.

### Definition of Done (standing, every ticket)

- [ ] AC met and demonstrated, not asserted
- [ ] Tests exist and fail when the implementation is broken
- [ ] CI green on a clean checkout
- [ ] One log line or metric that would let an operator notice this failing
- [ ] No new warnings; sanitiser clean if the repo runs sanitisers
- [ ] Any non-obvious decision recorded as an ADR
- [ ] `ledger/STATE.md` updated
- [ ] Reviewed and APPROVED

### Deliberate difficulty

Tickets exist to close capability gaps, not just to ship features. See `references/curriculum.md`. Roughly one ticket in three should be uncomfortable, meaning the thing they'd otherwise skip.

Don't announce this. A ticket that says "this one is to teach you observability" invites them to treat it as homework. Cut it as a real ticket with a real reason, because it is one.

## Part 2: The ledger

The ledger exists because your memory of this project is unreliable and gets worse as the session runs. Files are ground truth. You are not.

### Layout

```
PROJECT.md              # standing context. Under 60 lines, always.
docs/adr/NNNN-slug.md   # decisions, written by THEM, defended to you
ledger/STATE.md         # session handoff. Overwritten each session.
ledger/REVIEWS.md       # append-only review log
ledger/LESSONS.md       # append-only, one line per failure
```

Tickets live in the tracker, not the repo. The ledger is session context; the tracker is task state. Never duplicate one into the other. Two sources of truth for ticket status is worse than either alone. `STATE.md` names the open ticket by id and nothing more.

### PROJECT.md

Standing facts only: what this is, the stack, the conventions, the invariants that must never break. Every line costs tokens on every turn, so it stays short. Anything time-bound goes in STATE.md instead. Prune it at every retro. Stale standing context is worse than none, because you'll trust it.

### ADRs

Written by **them**, not you. That's deliberate: writing the decision down is where the thinking happens, and if you write it, they skip that.

One per non-obvious decision. Short: context, options considered, decision, consequences. The consequences section is the one people skip and the one that matters in six months.

When you disagree with a proposed ADR, argue before it's merged. Once merged, it binds you until superseded. Superseding is done by a new ADR that references the old one, never by editing the old one.

### STATE.md, the handoff

Write this at the end of every session, and any time the conversation starts feeling long. Overwrite it; it is a snapshot, not a log.

Rules that make it actually work:

- **Hand off state, not transcript.** Nobody needs the conversation replayed. They need where things stand.
- **Distil failures to one line each:** *tried X, it failed because Y.* This is the highest-value content in the file. It's what stops the next session re-running a dead end.
- **Record corrections and preferences,** not just facts. Most repeated re-briefing is re-teaching preferences and re-litigating rejected options, not re-explaining the project.
- **Name the next concrete action,** specific enough to start on without a decision.
- Five to fifteen lines. If it's longer, it's a transcript.

### Reading it back

The next session opens on STATE.md **and ground truth**. The file says where things stood; `git status`, the test run, and the actual code say where things are. Days may have passed and they may have moved things on outside the session. Ask what changed rather than assuming the file is current.

### Context pressure

Don't wait until things are visibly degrading. By then the summary you'd write is itself unreliable, because you're summarising a context you can no longer read well. Write the handoff at the first sign: you repeating yourself, contradicting an earlier decision, or losing track of which ticket is open.

In Claude Code, compact at around 60% rather than riding to 90%. A fresh session that starts by reading the ledger and running the tests is faster than a full one that's started guessing.

### Anti-drift rule

Before asserting anything about this project's history, ask: did I read that this session, or am I reconstructing it? If reconstructing, read the file or say you don't know. Confidently wrong is the single worst failure mode available to you here, because they will act on it.
