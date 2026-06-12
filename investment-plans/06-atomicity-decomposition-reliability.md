# Investment #6 — Atomicity & AC-Decomposition Reliability

| | |
|---|---|
| **Status** | **Greenlit & re-grounded (2026-06-11)** — the dissected modules are **off the live path**; first slice is now *delete* them + re-aim verification at `orchestrator/parallel_executor.py`. See [Owner direction](#owner-direction--greenlit--re-grounded-2026-06-11). |
| **Date** | 2026-06-11 |
| **Author** | deepak |
| **Owner** | TBD |
| **Effort / Risk** | Medium–High / Medium (the gate in front of fan-out; recursive) |
| **Persona** | All users — a wrong call here compounds through the whole tree |
| **Addresses** | [discussion #1385](https://github.com/Q00/ouroboros/discussions/1385) |
| **Related** | [#04 Frugality & the spend decision (sibling — same failure pattern)](./04-frugality-and-spend-decision.md) · [#01 Reliability](./01-runtime-provider-reliability.md) · [discussion #1377](https://github.com/Q00/ouroboros/discussions/1377) |

## TL;DR

Before every acceptance criterion, Ouroboros decides *is this unit atomic
(execute it) or not (decompose it first)?* Today the **only honest bit of that
decision is the boolean** — every numeric output is fabricated, the judgment is
one shaky low-temperature sample with a keyword fallback that is *anti-correlated*
with reality, and when it splits, nothing checks the split is sound or even
simpler. Because decomposition is recursive, an early misjudgment poisons every
branch beneath it. This is the gate in front of fan-out; it deserves a real
mechanism, not a patch.

## Owner direction — greenlit & re-grounded (2026-06-11)

The owner ([Q00](https://github.com/Q00)) gave this thread "the deepest verification
of the four" — and it surfaced a finding that **re-grounds the entire plan: the
module this analysis dissects is not in the live path.** We move forward on the
owner's revised basis.

- **The dissected module is dead on the live path.** `DoubleDiamondExecutor` — and
  with it `execution/atomicity.py` and `execution/decomposition.py` — has **no
  non-test caller in `src/`**. The owner's event store agrees: across recent runs,
  **115 `execution.ac.completed` events, zero atomicity-check events.** The live
  executor is **`orchestrator/parallel_executor.py`**, which carries its *own*
  profile-based decomposition (`axis: testable_unit`, an explicit `min_unit`, an
  `atomic_verifier_verdict`, and depth-cap warnings that *are* recorded).
- **Every file-level claim in the table below still holds** (the hardcoded
  `complexity_score = 0.3 if is_atomic else 0.8`, `MAX_DEPTH = 2` vs. the docstring's
  5, the Opus defaults, the keyword fallback, the accidental default-to-atomic) — they
  were simply aimed at the wrong file. **The same disease exists at larger scale in
  the live executor:** a ~6,300-line module whose load-bearing judgments ride on
  text-surface heuristics — e.g. AC classification via
  `re.finditer(r"\b(?:and|then|while|plus)\b|[,;:]", …)`, the exact anti-correlated
  shape called out below — and much of whose bulk reads as accreted per-failure regex
  patches.
- **First slice, revised:** (a) **delete** `execution/atomicity.py` +
  `execution/decomposition.py` rather than repair them — fixing fabricated fields in a
  module nothing calls is motion, not progress (the same call as removing `PALRouter`
  in [#04](./04-frugality-and-spend-decision.md)); (b) the decomposition-verification
  slice, **re-aimed at `parallel_executor.py`'s actual split path.**
- **Error bias — attempt-then-bounce empirically confirmed.** The live system already
  works this way de facto: ACs execute at seed granularity and evaluation bounces
  failures (115 ACs ran with *zero* pre-execution atomicity judgments). The work is
  not *choosing* the bias; it is adding discipline to the loop that already exists:
  **bounded attempts, bounce-cause classification** (too-big vs. bad-spec vs.
  environment), and **using the bounce *trace* as the decomposition input** — splitting
  from what was actually attempted and what remains beats splitting from text.
- **Properties — accepted with the same restructure as [#1384].** The safety-critical
  ones are v1 invariants (executor-relative, graded confidence, real measurements,
  *reasoned* asymmetric bias, verified decomposition, guaranteed termination with
  recorded compromises, safe degradation, faithful rationale). **Closed-loop (8)
  demotes to a v2 maturity goal** (cold-start). **Boundary-reliability (4)** carries a
  frugality tension — trigger extra samples *only* when confidence lands in the mid
  band. And one **owner-added property — live-path verification first** (property 11):
  any failure analysis or fix must establish what *actually executes* before investing
  (the lesson of this very thread).
- **Lineage note (for the RFC).** DoubleDiamond was Ouroboros's methodology-first
  executor — understand (Discover/Define) *before* judging atomicity, decompose
  recursively with discipline. Its philosophy wasn't wrong; its implementation
  abandoned it (the hardcoded scores are the moment it did). These properties largely
  **re-derive that philosophy as testable criteria** — so deleting the module is not
  discarding the idea, it is **transplanting it onto the executor that actually runs.**
- **Split, re-grounded:** the failure analysis this plan performs should be **redone
  against `parallel_executor.py`** — that is where the properties apply.

## Problem (originally grounded in `execution/atomicity.py` — now known off-path; see Owner direction)

| Verified finding | Where | Consequence |
|---|---|---|
| **The LLM path discards its own analysis and hardcodes the result fields** (`complexity_score`, `tool_count`, `estimated_duration` keyed only off the boolean). | `execution/atomicity.py` (`AtomicityResult` numeric fields; LLM parse path) | Only the boolean is real. Every numeric field in `ac_atomicity_checked` events is **fabricated** — the event stream records fiction. |
| Documented criteria ("atomic if complexity < 0.7, tools < 3, duration < 300s") are **bypassed** because the numbers are hardcoded. | `execution/atomicity.py:7-9, 48-50` (`DEFAULT_MAX_COMPLEXITY=0.7`, `DEFAULT_MAX_DURATION_SECONDS=300`) | The stated decision rule is dead on the primary path. |
| The judgment is a **single low-temperature sample**, no self-consistency. | `execution/atomicity.py` (`CompletionConfig(temperature=0.3, …)`) | Borderline units — the only ones that matter — are decided by one near-deterministic roll. No confidence, no voting. |
| The heuristic fallback is **keyword counting** ("and", "then", "database"…). | `execution/atomicity.py:211-248` (`_heuristic_atomicity_check`, `tool_keywords`) | "Add a button and a label" → "and" → non-atomic; "re-architect the persistence layer" → no keywords → atomic. On adversarial input it is *anti-correlated* with reality. |
| On any failure the loop **silently defaults to "atomic."** | `execution/double_diamond.py:1171` (`is_atomic = True  # Default to atomic if check fails`), also `:1539` | The unguarded default is the *costlier* error (execute an oversized unit), chosen by accident, not reasoned. |
| **Decomposition is asserted MECE, never verified.** Prompt requests MECE; validation only checks count (2–5), non-empty, parent-cycle. | `execution/decomposition.py:114` (MECE prompt) vs. `:194-225` (`_validate_children`) | Children can overlap or drop scope and the loop won't notice until evaluation — if then. No check that a child is actually *simpler* than its parent. |
| **Depth caps are internally inconsistent.** Docstring says "Max depth is 5 levels (NFR10)"; the constant is `MAX_DEPTH = 2`; the caller enforces its own `max_depth=5`. | `execution/decomposition.py:11` vs. `:59` (`MAX_DEPTH = 2`) vs. `execution/double_diamond.py:1041` (`max_depth=5`) | Trees are shallower/inconsistent vs. documented; at the cap a non-atomic unit is forced to execute as atomic **with no record** the compromise happened. |
| The decomposition meta-call **defaults to Opus.** | `config/models.py:203` (`decomposition_model = DEFAULT_OPUS_MODEL`) | Priciest tier on a call whose numeric outputs are then discarded (see Plan #4). |

**One line:** the only honest bit is the boolean; everything quantitative is
invented; the judgment is one shaky sample with an anti-correlated fallback; and
when it decides to split, nothing checks the split is sound or reduces complexity.

## Why now

The atomicity call is the **gate in front of fan-out**, with asymmetric costs:
- **Under-decompose** (oversized unit called "atomic") → executor takes on too
  much → phase fails/half-completes → **rework** (the priciest waste, #1377).
  This is the shape behind real fan-out blow-ups.
- **Over-decompose** (split something already doable) → wasted tokens on needless
  subtasks — recoverable, but pure motion.

Recursion compounds either error down every branch beneath a bad split.

## Target state — the acceptance properties (#1385)

These are this plan's acceptance criteria, **applied to `parallel_executor.py`'s
split path** (the live target).

> **Restructured per the owner (2026-06-11), mirroring [#1384].** Most are v1
> invariants; **closed-loop (8) demotes to a v2 maturity goal** (cold-start);
> **boundary-reliability (4)** fires extra samples *only* in the mid-confidence band
> (a frugality tension); and a **new property — live-path verification first (11)** —
> is added. See [Owner direction](#owner-direction--greenlit--re-grounded-2026-06-11).

1. **Executor-relative** — "atomic" = "completable in one focused unit by *the
   executor that will actually run it*" (its tier, tools, context). Judging in
   ignorance of the executor is disqualifying.
2. **Graded with explicit confidence**, not a bare boolean — so the next decision
   is risk-weighted, not coin-flipped.
3. **Outputs are real measurements** — no hardcoded stand-ins; the event stream is
   auditable and learnable.
4. **Reliable at the decision boundary** — robustness concentrated where calls are
   close (self-consistency / multiple samples / ensembling, or a cheap bounded
   execution attempt). Identical input → repeatable decision.
5. **Reasoned, asymmetric error bias** — the default under uncertainty
   deliberately favors the cheaper-to-recover error, and the direction is argued,
   not accidental.
6. **Decomposition is verified, not asserted** — children are collectively
   exhaustive (no dropped scope), mutually exclusive (no overlap), and **each
   measurably closer to atomic** than the parent. A failed check triggers
   repair/retry, not silent acceptance.
7. **Provable convergence and termination** — every level strictly reduces
   remaining difficulty; depth caps are internally consistent; no path forces a
   known-non-atomic unit to execute as atomic **without recording the compromise.**
8. **Closed-loop with execution and evaluation** — units judged atomic that then
   fail for being too big (and splits that were needless) recalibrate future
   judgments.
9. **Safe degradation without an LLM** — any fallback beats a trivial baseline, or
   reports "uncertain → decompose conservatively / defer" — never a confident
   keyword guess. An anti-correlated fallback is worse than none.
10. **Faithful, auditable rationale** — each decision emits its reasoning,
    confidence, and the real inputs used. *(v1 invariant — also the v2-calibration enabler.)*
11. **Live-path verification first** *(owner-added)* — before any failure analysis or
    fix is funded, establish *what actually executes*. This thread's central lesson:
    the original analysis was correct file-by-file but aimed at a module nothing calls.
    Any RFC here must open by confirming the live caller graph (here: `parallel_executor.py`,
    not `double_diamond.py`).

> **v1 vs v2.** v1 invariants: **1, 2, 3, 5, 6, 7, 9, 10, 11**. v2 maturity goal: **8**
> (closed-loop recalibration — gated on cold-start; v1 records its inputs honestly).
> Property **4** (boundary robustness) is v1 but *budget-aware*: extra samples only in
> the mid-confidence band.

Honor the author's "cheapest highest-leverage first slice," which removes the most
dangerous **silent** failures with the smallest diff:

1. **Make the data honest before making the judgment smart.** Stop fabricating
   numeric outputs (property 3) and fix the depth-cap inconsistency + record
   forced-atomic compromises (property 7) — these are small diffs that immediately
   make the event stream trustworthy and auditable.
2. **Verify splits.** Add decomposition verification with one repair retry
   (property 6) — the second-most-dangerous silent failure.
3. **Then** strengthen the judgment itself (confidence, boundary robustness,
   executor-relativity, reasoned bias) and close the loop.

Shared design rule with Plan #4: **real measurements, calibrated thresholds,
faithful rationale** — never fabricated outputs.

## Plan (phased, baby-steps)

### P0 — Delete the off-path modules + verify splits in the live executor (revised)
> **Reshaped per owner (2026-06-11):** the original "stop fabricating outputs" tasks
> targeted `execution/atomicity.py` / `execution/decomposition.py`, which nothing in
> `src/` calls. Repairing a dead module is motion, not progress — so P0 is now
> *delete + re-aim at the live split path*.
- **Tasks:**
  - (a) **Delete** `execution/atomicity.py` + `execution/decomposition.py` (plus the
    dead `DoubleDiamondExecutor` atomicity/decomposition wiring and their always-Opus
    `config/models.py` defaults). The methodology *lineage* is preserved in the RFC,
    not the code.
  - (b) **Re-aim the decomposition verifier at the live split path** in
    `orchestrator/parallel_executor.py`: after a split, check collective-exhaustiveness,
    mutual-exclusivity, and "each child measurably closer to atomic"; on failure,
    **one repair retry**, then escalate — not silent acceptance (property 6).
  - (c) **Honest outputs where the live executor records them:** confirm the existing
    `atomic_verifier_verdict` / depth-cap-warning fields carry the *real* inputs used,
    and record any forced-atomic compromise (properties 7, 10).
- **Touch-points:** **delete** `execution/atomicity.py`, `execution/decomposition.py`
  (+ their tests); **edit** `orchestrator/parallel_executor.py`, `config/models.py`.
- **Acceptance:** the two off-path modules are gone with **no `src/` regression** (a
  test asserts they have no non-test importer before removal); an overlapping/
  scope-dropping split *in the live path* is caught and repaired; every forced-atomic
  compromise appears in the event stream.
- **Effort:** ~1 week. **Ships alone** — removes the dead code *and* the most
  dangerous silent failure in the live path.

### P1 — Graded, boundary-robust, executor-relative judgment
- **Tasks:** Return a **confidence** alongside the boolean (property 2); add
  boundary robustness — self-consistency / multiple samples, or a cheap bounded
  execution attempt for near-boundary cases (property 4); make the judgment
  **executor-relative** to the effort/tools/context that will run it (property 1, ties
  to Plan #4's *effort* decision); replace the **silent default-to-atomic** with a
  **reasoned, argued** bias (property 5); make the no-LLM fallback **safe** —
  "uncertain → decompose conservatively / defer," never a keyword guess (property
  9).
- **Acceptance:** identical input yields a repeatable decision; borderline units
  get extra scrutiny, not a coin flip; the fallback never emits a confident
  keyword-based verdict.

### P2 — Closed-loop recalibration
- **Tasks:** Feed evaluation/execution outcomes back: units called atomic that
  failed for size, and splits that were needless, recalibrate future judgments
  (property 8). Natural feedback source is the evaluation verdict stream
  (`harness` Deliver gate / `evaluation/`).
- **Acceptance:** a class of unit that repeatedly mis-judged shifts its future
  default; the shift is auditable and reversible.

## Metrics / success signals

- % of the live executor's split/verdict fields (`atomic_verifier_verdict`,
  depth-cap warnings) that carry **real** inputs (target: 100% after P0). *(The
  off-path `ac_atomicity_checked` event is removed with `atomicity.py`, not fixed.)*
- Fan-out "blow-up" rate (a batch of "atomic" units that were not actually atomic)
  → down.
- Decomposition repair-retry catch rate (splits caught as non-MECE / not-simpler).
- Mis-judgment recurrence across runs → down (property 8).

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Boundary robustness (multiple samples) raises cost on every AC | Concentrate extra sampling *only* at the boundary (property 4); a cheap bounded execution attempt can replace extra LLM votes; route the meta-call off always-Opus (Plan #4). |
| "Each child simpler than parent" is hard to measure objectively | Use the same difficulty signal Plan #4 builds; start with a coarse monotonicity check, refine with calibration. |
| Changing the default error bias regresses behavior | Make the bias explicit and configurable; A/B against eval pass-rate; author leans **attempt-then-bounce** for near-boundary cases (evaluation already exists to catch an oversized attempt). |

## Non-goals

- A formal convergence proof in v1 (property 7 is satisfied by consistent caps +
  recorded compromises + monotone-progress checks, not a theorem).
- The *how-much-to-invest* decision — that is Plan #4 (#1384). This plan is the
  *whether-and-how-to-split* decision.

## Open questions (from #1385)

- **Default error bias:** attempt-then-bounce for near-boundary (author's lean,
  adopted as P1 default) vs. decompose-when-unsure — confirm per Ouroboros's real
  rework vs. waste costs.
- Is full convergence-proof overkill for v1, or exactly the non-negotiable?
  (Recommendation: consistent caps + recorded compromises in P0; monotone-progress
  check in P1; defer any formal proof.)
- **One RFC or split?** **Resolved (owner, 2026-06-11): split, and re-grounded.** The
  failure analysis is **redone against `orchestrator/parallel_executor.py`** (per
  property 11), then split into the deletion + live-path decomposition verifier (P0),
  the graded/boundary-robust judgment (P1), and the v2 feedback loop (P2).
