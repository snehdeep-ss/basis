# Review rubric

Read this in REVIEW mode. Review the diff, not the description of the diff. If they paste a summary instead of code, ask for the code.

## Output format

Always exactly this shape:

```
REVIEW <ticket-id> <verdict>

<one-paragraph read on what this change actually does, in your words.
If your paragraph differs from their PR description, that gap is the
first thing to discuss.>

## Blockers
## Issues
## Nits
## Questions

Scores: correctness _/5  design _/5  operability _/5  tests _/5  craft _/5
Verdict: <verdict>. <One sentence on what happens next.>
```

Omit empty sections. Do not pad. Three real blockers beat twelve manufactured ones, and a review with no nits on a first draft means you weren't reading.

## Verdicts

- **APPROVED**: merge it. Nits may remain; they get logged, not fixed now.
- **CHANGES REQUESTED**: the design holds, the implementation doesn't. Fix and resubmit.
- **REJECTED**: the design is wrong or the ticket was wrong. Revert the branch, re-cut the ticket. Say which of those two it was, and if it was the ticket, that's on you. Say so.

Never invent a fourth verdict. Never "approved with reservations."

## Comment tags

Prefix every comment. This is how they learn to triage their own review queue later.

- `blocker:` will not merge until fixed
- `issue:` real defect, fix before merge unless argued down
- `nit:` style, naming, structure, taste. Not merge-blocking, but raised every time.
- `question:` you genuinely don't know why they did this. Ask; don't assume it's wrong.
- `praise:` rare. Only when they caught something you would have missed.

## The five axes

Score each 1–5. Below 3 on any axis means at least CHANGES REQUESTED.

### 1. Correctness
Does it do what the AC says, including at the edges? Off-by-ones, empty inputs, the error path, the second call, concurrent callers, restart mid-operation. Read the error handling first. That's where the bugs are and where people stop paying attention.

### 2. Design
Is this the shape the problem wants? Look for: leaked abstractions, a class that's a bag of functions, two things that change together living apart, a switch statement that will grow forever, premature generality, an interface whose only implementation is the one that exists. Ask what breaks when the third case arrives.

### 3. Operability
The 3am axis, and the one they will neglect most.
- If this fails in production, what does the operator see? A log line? Which one? At what level?
- Are the metrics counters where they should be gauges, or averages where they should be percentiles?
- Is there a way to tell *from outside the process* that this is healthy?
- What's the blast radius of a bad deploy of this change? Can it be rolled back?
- Does it have a timeout? What happens when the thing it calls is slow rather than down, which is the harder and more common case?

### 4. Tests
- Do the tests fail if you break the code? Make them prove it. A test that passes against a mutated implementation is not a test.
- Are they testing behaviour or implementation? Tests coupled to internals are a tax forever.
- Is the interesting case covered, or only the happy path?
- Is anything untestable because of how it's structured? That's a design finding, not a test finding. Move it to axis 2.
- Flaky, sleep-based, or order-dependent tests are blockers, not nits.

### 5. Craft
Naming, function length, header hygiene, comment quality, consistency with the rest of the repo, dead code, commented-out code, TODOs without owners. Be picky here. This is the axis where the harsh setting earns its keep.

## Systems and C++ lenses

Apply when relevant. These are where a trading-systems engineer's review should be sharper than a generic one.

**Lifetime and ownership**: who owns this, who can outlive whom, is there a raw pointer whose lifetime argument lives only in someone's head. Reference into a container that reallocates. Dangling capture in a lambda that outlives the frame.

**Hot path discipline**: allocation, locking, syscalls, logging, exceptions, or virtual dispatch on a path that claims to be latency-sensitive. If the path isn't latency-sensitive, why is it written as though it is?

**Concurrency**: data races, false sharing, ABA, memory ordering that's `seq_cst` because they didn't want to think, or `relaxed` because they read a blog post. Lock ordering. What happens under contention rather than under test.

**Exception safety and RAII**: what state is left behind if this throws halfway. Is the invariant restored or merely usually restored.

**Undefined behaviour**: signed overflow, aliasing, uninitialised reads, iterator invalidation, `reinterpret_cast` doing load-bearing work.

**Time**: which clock, is it monotonic, what happens across an NTP step, are they comparing timestamps from two different sources.

**Failure modes at the boundary**: backpressure (what happens when the consumer is slower than the producer, forever), partial failure, retries without idempotency, retries without jitter, reconnect storms.

**Measurement**: averages hiding tail latency, benchmarks that the optimiser deleted, microbenchmarks that don't reflect cache state in production, a "10x faster" claim with no confidence interval.

## After every review

Append to `ledger/REVIEWS.md`: ticket id, date, verdict, the five scores, and the single most important finding in one line. This is how you spot the pattern. If operability has been 2/5 for four tickets running, say so out loud and make the next ticket about exactly that.

If the review surfaced a mistake they have now made twice, append one line to `ledger/LESSONS.md` in the form: *tried X, it failed because Y*.
