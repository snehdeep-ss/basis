# Curriculum: the five gaps

This project exists to close five specific gaps. Every epic should touch at least two. Track which axes are being exercised and which are being avoided. Avoidance is information.

The engineer writes C++ for trading systems. They are not weak at code. They are weak at the things a team usually absorbs for them, and at the things an AI-heavy workflow quietly does on their behalf.

---

## 1. Testing and CI/CD

**The real gap:** knowing what to test, not how to write a test.

Force it by: making tests part of AC rather than an afterthought; requiring that a test be shown to fail before the fix lands; asking "what would have caught this in CI" after every bug.

Escalate over time. See the ladder below.

**The pipeline ladder.** One or two tickets per rung, in order:

0. A local script that builds and tests. CI calls it. No logic in CI that can't be run on the laptop. Pipelines that only work in CI are undebuggable.
1. Build + test on push and PR; branch protection requires it green.
2. Matrix: two compilers, two build types, warnings-as-errors.
3. Caching (ccache/sccache) against a stated time budget. A CI they won't wait for is a CI they'll bypass.
4. Quality gates as separate jobs: ASan, UBSan, TSan, clang-tidy, coverage floor. Separate so a failure names itself.
5. Artifacts and releases on tags: versioning, changelog, reproducibility.
6. CD: container, deploy to one box, health check, rollback path.
7. Deliberately break each gate and confirm it fires. A gate never seen to fail is not known to work.
8. Migrate off managed CI: self-hosted runner or a self-hosted engine. The value is in discovering every implicit dependency the managed environment was providing.

Rungs 0–2 come before feature work resumes. The rest fits between tickets.

Pipeline config is code. It falls under the iron law. Cut the ticket, review the diff, never write the YAML.

Things to be harsh about: tests that never fail, mocks that assert the mock, `sleep()` in tests, a green CI that doesn't actually run the interesting suite, "works on my machine" as a justification.

**Signal they've got it:** they start writing the test first without being told, and they get irritated by slow CI rather than tolerating it.

---

## 2. Observability and on-call operations

**The real gap:** building things that can be debugged by someone who isn't the author, at 3am, without a debugger attached.

Force it by: requiring one log line or metric per ticket as part of DoD; asking what an operator sees when this fails, every single review; eventually making them write a runbook and then run a game day against their own system with the code hidden.

Progression: structured logging with levels that mean something → metrics with correct types (counter vs gauge vs histogram) → percentiles not averages → health and readiness distinction → tracing across a boundary → alerting on symptoms rather than causes → a runbook → an incident writeup for a failure they caused.

Things to be harsh about: log lines with no identifiers, DEBUG used as INFO, averages for latency, metrics nobody would ever look at, alerting on CPU instead of on user-visible symptoms, no way to tell healthy from hung.

**Signal they've got it:** they add the metric before you ask, and they can answer "is it healthy" without attaching a debugger.

---

## 3. API and product design

**The real gap:** designing for a user who isn't them. They said they want people to actually use this. That only happens if someone other than the author can pick it up.

Force it by: making them write the README, the error messages, and the first-run experience as deliberate work, not leftovers; asking who the user is and what they were doing thirty seconds before they touched this; requiring a compatibility story before any public interface is declared stable.

Progression: name things from the caller's side → error messages that say what to do next → sane defaults, minimal required config → versioning and deprecation policy → getting one real outside user and watching them fail.

Things to be harsh about: interfaces designed from the implementation outward, options that exist because a decision was avoided, errors that report what failed but not what to do, a README that documents the architecture instead of how to start it.

**Signal they've got it:** they change an interface because it was confusing, not because it was slow.

---

## 4. Distributed systems and failure modes

**The real gap:** partial failure. Single-process reasoning does not survive the network.

Force it by: never accepting a happy-path design; asking "and if that call hangs rather than fails" on every external boundary; requiring an explicit answer on backpressure whenever there's a producer and a consumer.

Progression: timeouts everywhere → retries with jitter and a budget → idempotency → backpressure and bounded queues → at-least-once vs at-most-once, chosen deliberately → clock skew and ordering → graceful degradation → deliberately injected faults.

Things to be harsh about: unbounded queues, retries without jitter, no timeout, assuming a message arrives once, assuming clocks agree, treating slow as a special case of down when it's the harder case, reconnect logic that produces a thundering herd.

**Signal they've got it:** they describe a failure mode you hadn't thought of, unprompted.

---

## 5. Clean code and maintainable design

**The real gap:** taste, and the discipline to refactor before it hurts. This is also the axis where the harsh setting applies most directly.

Force it by: nitpicking every review; periodically making them return to code they wrote weeks ago and either defend or change it; requiring that a refactor ticket be cut whenever a feature ticket reveals the design is fighting them.

Patterns worth exercising, in service of a real problem and never for their own sake: dependency inversion at the I/O boundary so the core is testable, strategy where a switch was growing, RAII and scope guards, value semantics over shared mutable state, composition over inheritance, making illegal states unrepresentable in the type system.

Things to be harsh about: a pattern applied because it was learned rather than needed, a 200-line function, a class that's a namespace with extra steps, comments that restate the code, inconsistent naming within one file, abstraction with exactly one implementation, dead code, TODOs with no owner.

**Signal they've got it:** they delete an abstraction they built, because it wasn't earning its keep.

---

## Tracking

At each retro, score the five axes 1–5 based on the review log and say which one has been quietly avoided. Point the next epic at it.

Do not turn this into a gamified dashboard. It's a note in the retro, nothing more. The moment the score becomes the goal, the reviews start optimising for the score.
