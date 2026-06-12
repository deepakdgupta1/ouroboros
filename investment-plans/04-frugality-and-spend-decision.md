# Investment #4 — Frugality & the Spend Decision

> **Merged workstream.** This plan unifies what were two sibling drafts — *visible
> frugality* (the stance + attribution + learning) and *complexity→tier routing*
> (the mechanism that actually decides spend). They are merged because neither
> works without the other: the stance is unfunded without a real spend decision,
> the mechanism is unsteerable without visibility, and visibility is hollow
> without honest signals. One workstream, shipped in concert.

| | |
|---|---|
| **Status** | **Greenlit (2026-06-11)** — split into *estimator* + *actuator* RFCs; actuator reshaped to an **effort dial** (tier machinery to be **removed**, not wired). See [Owner direction](#owner-direction--greenlit--reshaped-2026-06-11). |
| **Date** | 2026-06-11 |
| **Author** | deepak |
| **Owner** | TBD |
| **Effort / Risk** | Med–High / Medium (load-bearing on every AC; touches the hot path) |
| **Persona** | All users — funds the frugality *and* reliability promises at once |
| **Addresses** | [discussion #1377](https://github.com/Q00/ouroboros/discussions/1377) (frugality stance + themes) · [discussion #1384](https://github.com/Q00/ouroboros/discussions/1384) (complexity→tier mechanism) |
| **Related** | [#01 Reliability / completion pre-flight](./01-runtime-provider-reliability.md) · [#06 Atomicity sibling](./06-atomicity-decomposition-reliability.md) · [Architecture Value Map §11.4](../ouroboros-architecture-value-map.md) · [improvement-themes §1 (Stage E)](../improvement-themes.md) · memory: *frugality-reflective-guardrails* |

## TL;DR

Frugality in Ouroboros has three parts that only deliver value together:

1. **The stance (#1377):** frugality is *waste-only and goal-subordinate* — it
   removes motion that didn't advance a *verified* acceptance criterion, never the
   comprehensiveness the user pays for.
2. **The mechanism (#1384):** a principled decision about *how much model
   capability to commit per unit* — which today is **not even wired into the live
   path**, and where computed, mistakes length for difficulty.
3. **Visibility & learning:** surface waste per run and codify recurring waste as
   reusable guardrails — using signals that are **already event-sourced** but never
   aggregated.

This plan delivers all three: make the spend decision real and safe, make the
resulting waste visible and honest, then learn from it so it compounds.

## Owner direction — greenlit & reshaped (2026-06-11)

The owner ([Q00](https://github.com/Q00)) responded to both discussions this plan
serves. Verdict: **greenlit, with two structural reshapes.** We move forward on
that basis; the inline sections below are annotated where the owner changed them.

**From [#1377](https://github.com/Q00/ouroboros/discussions/1377#discussioncomment) (the stance):**

- **Waste-only / goal-subordinate confirmed exactly.** The destination (ACs met,
  verified) is fixed; only path cost is negotiable. A budget guillotine that halts a
  run mid-way is "the one tool we'd never reach for." The two invariants — never
  reduce the achieved outcome, never increase rework risk — are accepted as *the
  acceptance test* for any frugality mechanism.
- **The four themes are one control loop, not four features** (the owner's TCP
  analogy — a controller for a network whose capacity it cannot observe):
  - **Spend attribution = the *sensor*** — everything else reads from it, so it goes first.
  - **Feasibility pre-flight = the controller's *initial estimate*,** not a one-shot
    gate (see the GLM concurrency-vs-quota correction now in [#01](./01-runtime-provider-reliability.md)).
  - **429 / limit handling = the controller's *normal operation*,** not exception
    handling — shrink the window, pause, resume after reset; completed work is never lost.
  - **Assurance dial = the *policy input*,** held by the user — never automatic.
  - **Reflective guardrail loop = the *learning layer*** that improves the estimate across sessions.
- **Fan-out discipline is core's job, not the runtime's** (the one sharpening to
  layered ownership) — and core already shipped the first response:
  `orchestrator/backend_limits.py` serializes delivery to **1 AC at a time** for any
  backend whose limits Ouroboros can't know (every CLI runtime, hermes included),
  raised only via explicit `OUROBOROS_MAX_CONCURRENCY`;
  [#1372](https://github.com/Q00/ouroboros/pull/1372) added configurable rate-budget
  pacing for non-Claude delivery. **The 14-AC stampede cannot recur in that form.**
- **First slice greenlit: spend attribution + the advisory guardrail loop** (P1
  below), exactly as proposed. The **adaptive-concurrency evolution of
  `backend_limits`** (static cap → signal-driven) is the **natural second slice**.

**From [#1384](https://github.com/Q00/ouroboros/discussions/1384) (the mechanism) —
the strongest of the four threads; every code-level claim verified:**

- **The actuator is reasoning *effort*, not model switching.** Route on the
  thinking/effort dial modern backends expose (Anthropic thinking budget, OpenAI
  reasoning effort, Codex `model_reasoning_effort`). It is monotonic and cliff-free
  *by construction* (property 8), escalation is cheap so fail-safe-on-uncertainty is
  actually practical (property 3), and a mis-estimate no longer changes the
  executor's *capability class* — which shrinks the cold-start risk. **Model-tier
  switching survives only as the coarse outer rung of an escalation ladder** (effort
  low → effort high → bigger model) and as the fallback where a backend exposes no
  effort dial (GLM's thinking toggle is nearly binary). **Consequence: the unwired
  `PALRouter` / tier machinery is removed, not wired** — it encodes the wrong actuator.
- **The ten properties are restructured** (they are *not* all v1 necessary conditions):
  - **v1 invariants — the safety properties: 2** (stakes axis), **3** (escalate under
    uncertainty), **8** (monotonic/stable), **10** (faithful rationale).
  - **v2 maturity goals: 6** (calibration) **and 7** (cross-run learning) — they
    collide with cold-start (a single user's runs don't yield enough labeled
    outcomes). v1 must *record* what calibration needs (real inputs + real outcomes —
    property 10 enables this); v2 closes the loop.
  - **New property (owner-added) — runtime-scoped applicability.** "Load-bearing in
    the live path" (1) has a backend-dependent ceiling: where Ouroboros calls the LLM
    directly it can *route*; where it delegates to a CLI runtime (hermes, codex) it
    may have no per-call control and can only *advise*. The mechanism needs a
    **capability matrix + an explicit advise-fallback**; a design assuming universal
    routability is vacuous on exactly the backends the incident came from.
- **First slice (the author's pick, reshaped):** (a) make the decision load-bearing
  *where the backend permits* — i.e. **wire an effort-first investment decision, not
  the tier router**; (b) add the stakes axis; (c) ship the escalate-on-uncertainty default.
- **RFC split:** one RFC for the **estimator** (difficulty + stakes, measured inputs
  only), one for the **actuator / wiring** (effort dial + capability matrix +
  escalation ladder). The calibration loop waits for v2, with its event-stream
  groundwork laid in v1.

## Why one workstream

- "Spend the cheap model on cheap work" (the stance) is **impossible** until
  something reliably identifies cheap work (the mechanism).
- The mechanism is **unsteerable and untrustworthy** until its spend and its waste
  are visible to the user (visibility).
- Visibility is **hollow** until the underlying numbers are real, not fabricated
  (honest signals — the shared failure pattern with [#06](./06-atomicity-decomposition-reliability.md)).

Shipping these as one effort avoids building a visibility layer over a decision
that doesn't exist, or a routing mechanism no one can see or steer.

## Discussion alignment

**#1377 (the stance).** Two pieces are binding design constraints here:

- **Two hard invariants (non-negotiable):** the destination (working product, all
  ACs met) is fixed; only the *path* cost varies. Every mechanism must (1) **never
  reduce the achieved outcome** and (2) **never increase rework risk**, and must
  **never halt a run mid-way**. This is why a prospective budget cap is the wrong
  tool and an explicit non-goal — waste is often indistinguishable from genuine
  exploration *in advance*, so guardrails are emitted **retrospectively**, only
  once spend is clearly non-advancing.
- **Layered ownership:** Ouroboros (methodology) owns *how much* work, at *what
  fidelity*, plus **spec crispness** (a sharper seed prevents the priciest waste,
  rework). The agent **runtime + LLM backend** own *how cheaply* a commissioned
  unit executes (re-reads, retries, regeneration); for that layer core can only
  **advise** (a guardrail in the spec) or **route** (pick a cheaper backend/model).

The four #1377 themes map onto the phases below:

| #1377 theme | Where it lands |
|---|---|
| **Spend attribution** (per-stage: interview / execute / consensus) | P1 (measure + surface) |
| **Reflective guardrail loop** (advisory v1) | P1 (detect) + P2 (codify, tagged methodology→act vs. execution→advise/route) |
| **User-held assurance dial** (the *one* legitimate cost/assurance lever) | P2 (single shared dial) |
| **Completion-feasibility pre-flight** | Owned by [#01](./01-runtime-provider-reliability.md) — cross-referenced, not duplicated |

**#1384 (the mechanism).** Its ten acceptance properties are this plan's acceptance
criteria for the spend decision — see [Target state](#target-state).

## Problem (grounded in current `main`, verified)

### Part A — the spend mechanism isn't load-bearing

| Verified finding | Where | Consequence |
|---|---|---|
| The complexity **estimate** is computed, but the **tier router is never called in execution.** `estimate_complexity` is invoked at `execution/atomicity.py:271`, yet `PALRouter.route()` / `ModelRouter.route()` (`routing/router.py:94`, `plugin/orchestration/router.py:156`) have **no non-test call site**. | `routing/`, `plugin/orchestration/router.py` | No complexity-based **tier selection** in the live path. The "route to a cheaper model" lever does not exist at runtime. |
| Live phase execution runs **one fixed model** for all four Double-Diamond phases. | `execution/double_diamond.py:466` (`self._default_model = get_double_diamond_model()`) | Uniform spend regardless of triviality or difficulty. |
| The per-AC meta-decisions (atomicity, decomposition) **default to Opus.** | `config/models.py:202-204`; `_model_defaults.py:31` (`DEFAULT_OPUS_MODEL = "claude-opus-4-8"`) | The gating calls that run on *every* AC use the priciest tier by default — the opposite of frugal. |
| Complexity = weighted sum of three **length-ish** scalars: tokens ×0.30 (÷4000), tools ×0.30 (÷5), depth ×0.40 (÷5). | `routing/complexity.py:40-52` | **Length treated as difficulty.** "Fix the race in the lock-free queue" scores ≈0 → would route Frugal. |
| The tool-dependency factor is a **fabricated count re-encoded as placeholder strings.** | `plugin/orchestration/router.py` (`tool_dependencies=[f"tool_{i}" …]`) | A 30%-weighted factor carries no real signal. |
| Weights, thresholds (0.4/0.7), normalizers (4000/5/5), tier multipliers (1×/10×/30×) are **uncalibrated constants**; the only adaptive signal is escalate-after-2-failures *within one run*. | `routing/router.py`, `routing/tiers.py:59-60` | No empirical basis; multipliers may be stale vs. real pricing; the estimate never learns across runs. |

**One line:** today there is effectively **no complexity-based investment
decision** — work runs on a fixed tier, the meta-decisions run on the priciest
tier, and the routing machinery sits unused and, where exercised, equates
verbosity with difficulty.

> **Owner refinements (2026-06-11) to this analysis:**
> 1. **The fix is not "wire the tier router" — it's an *effort-first* decision.** The
>    owner accepts every finding above but reshapes the actuator to the reasoning-effort
>    dial; `PALRouter` / `ModelRouter` and the tier-multiplier machinery encode the
>    *wrong actuator* and are **removed, not wired** (see [Owner direction](#owner-direction--greenlit--reshaped-2026-06-11)).
> 2. **`execution/double_diamond.py` is itself off the live path.** Per the deeper
>    caller-trace in [#1385](https://github.com/Q00/ouroboros/discussions/1385), the
>    `DoubleDiamondExecutor` has **no non-test caller in `src/`**; the live executor is
>    `orchestrator/parallel_executor.py` (115 `execution.ac.completed` events, **zero**
>    atomicity-check events across recent runs). The row above describing it as "the live
>    phase executor" inherits that correction — but the bottom line (no complexity-based
>    investment decision at runtime) is **unchanged, strengthened**. The effort-first wiring
>    therefore targets `parallel_executor.py`, not `double_diamond.py`. See [#06](./06-atomicity-decomposition-reliability.md).

### Part B — waste is invisible and uncompounded

- Cost/token signals **are already event-sourced** (`orchestrator/events.py`
  `estimated_cost_usd`; `orchestrator/workflow_state.py:750-753` `estimated_tokens`;
  `session.py` persists them) — but **nothing aggregates them into a *waste*
  view** (tokens on ACs that later failed/were re-done, dead escalations,
  stagnation cycles).
- Cost/token is **display-only in the TUI** (`tui/cost_tracker.py`,
  `tui/token_tracker.py`) — not a persisted, analyzable metric.
- `observability/retrospective.py` already produces per-run retrospectives, and
  `resilience/stagnation.py` already detects wasted-motion patterns — but **nothing
  carries a lesson forward** between runs.

### The user-visible pain (#1377)

Spend is opaque (no per-stage attribution, no pre-run expectation, no signal a run
can even *finish* on the backend's quota), and frugality is easy to misframe as
"spend less / cut scope." Grounding: a real multi-day build on a single backend
exhausted its rolling quota mid-run — a 14-AC fan-out hit HTTP 429 and hard-failed
*all* of them with zero useful work, and the agent sometimes thrashed (overlapping
output dirs, scratch files) — pure motion advancing no acceptance criterion.

## Why now

This decision is load-bearing for **two** headline promises at once:

- **Frugality (#1377):** "spend cheap on cheap work" needs something that reliably
  identifies cheap work.
- **Reliability ([#01](./01-runtime-provider-reliability.md)):** its mirror is
  "don't under-power hard/high-stakes work" — a mis-route to a weak tier produces
  silent failure and rework, the priciest waste #1377 names.

Both rest on one estimate. If it is unprincipled, both are unfunded.

## Target state

### The mechanism properties (#1384)

> **Restructured per the owner (2026-06-11).** They are no longer all v1 necessary
> conditions: the **safety properties (2, 3, 8, 10) are v1 invariants**;
> **calibration (6) and cross-run learning (7) demote to v2 maturity goals** (they
> collide with cold-start — v1 *records* what they need, v2 closes the loop); and a
> **new property — runtime-scoped applicability (11)** — is added. See
> [Owner direction](#owner-direction--greenlit--reshaped-2026-06-11).

1. **Load-bearing in the live path — *where the backend permits*** — the decision
   actually gates execution. Reshaped: the lever is the **reasoning-effort dial**,
   not tier/model switching; and per property 11 it *routes* where Ouroboros calls
   the LLM directly, *advises* where it delegates to a CLI runtime.
2. **Two-axis minimum: difficulty *and* stakes** — investable on cost-of-being-wrong
   (reversibility, blast radius, sensitivity) alone, e.g. auth / schema migration /
   payment paths.
3. **Fail-safe, safety-asymmetric under uncertainty** — when unsure, **escalate
   rather than cheapen.**
4. **Inputs are measured signals, not fabricated counts.**
5. **Difficulty decoupled from length** — a short hard unit rates hard; a long
   mechanical one rates easy.
6. **Calibrated to observed outcomes, inspectably** — constants justified by
   recorded success/failure/rework/cost.
7. **Closed-loop across runs** — a unit that failed at tier T shifts how similar
   future units route.
8. **Monotonic and stable** — more difficult/higher-stakes never maps cheaper; no
   threshold cliffs.
9. **Current, configurable cost model + a single user-held risk dial** — real,
   updatable pricing; the economize-vs-assure appetite is one explained lever.
10. **Faithful, auditable rationale** — every decision emits which axis drove it,
    the confidence, and the *real* inputs used. *(v1 invariant — also the enabler for
    the v2 calibration loop.)*
11. **Runtime-scoped applicability** *(owner-added)* — the mechanism carries a
    **capability matrix** and an explicit **advise-fallback**: it *routes* effort
    where Ouroboros calls the LLM directly, and *advises* (hands the guardrail down
    in the spec) where it delegates to a CLI runtime that exposes no per-call effort
    control. A design assuming universal routability is vacuous on the very backends
    the incident came from.

> **v1 vs v2.** v1 invariants: **2, 3, 8, 10, 11**. v2 maturity goals: **6, 7**
> (calibration + cross-run learning — gated on cold-start; v1 only records their
> inputs). Properties 1, 4, 5, 9 are v1 *targets* shaped by the effort-first actuator.

### Frugality visibility targets (#1377)

- Every run ends with a **waste retrospective**: total spend and an itemized
  *avoidable* portion (rework, dead escalations, stagnation), in tokens and
  estimated USD, plus **per-stage attribution**.
- **Floor-preserving:** never flags the *thorough* work the user asked for as
  waste; only motion that failed to advance a *verified* AC.
- Recurring waste is **codified as reusable guardrails** future runs consult.

## Strategy

Sequence so each phase rests on the last; honor #1384's "cheapest highest-leverage
first slice" (make it load-bearing + stakes-aware + fail-safe before making it
smart):

1. **Make the spend decision real and safe first.** A mediocre estimate that
   *escalates under uncertainty* beats a perfect estimate nothing calls. Wire the
   **effort-first** decision into the live path (`parallel_executor.py`) with a
   fail-safe default, behind the capability matrix (P0).
2. **Then make it honest and visible together.** Honest, length-decoupled inputs
   (the mechanism) are exactly what make the waste retrospective trustworthy — so
   measured inputs and waste visibility ship in the same phase (P1).
3. **Then learn and compound.** Cross-run calibration (the mechanism) and learned
   waste guardrails (frugality) are the same closed-loop idea from two angles —
   unify them, behind the single user-held risk/assurance dial (P2).

**Shared design rules** (with [#06](./06-atomicity-decomposition-reliability.md)):
honest outputs, calibrated thresholds, faithful rationale — never fabricated
numbers; and layered ownership — core decides/routes, advises the runtime.

## Plan (phased, baby-steps)

### P0 — Make the spend decision load-bearing (effort-first), stakes-aware, fail-safe
> **Reshaped per owner (2026-06-11):** the actuator is the **reasoning-effort dial**,
> not the tier router; this is the **actuator/wiring RFC**. The `PALRouter` / tier
> machinery is **removed**, not wired.
- **Tasks:**
  - (a) **Wire an effort-first investment decision** into the **live** executor
    (`orchestrator/parallel_executor.py` — *not* `double_diamond.py`, which is off the
    live path per [#1385](https://github.com/Q00/ouroboros/discussions/1385)). The
    decision sets the backend's reasoning-effort/thinking dial per unit; the escalation
    ladder is *effort low → effort high → bigger model*. Behind the **capability
    matrix** (property 11): route where the backend exposes an effort dial, **advise**
    (guardrail handed down in the spec) where it does not. Delete the unwired
    `PALRouter` / `ModelRouter` / tier-multiplier code.
  - (b) **Add the stakes axis:** reversibility / blast-radius / sensitivity, so a
    short-but-high-stakes unit is investable on stakes alone.
  - (c) **Fail-safe default:** when estimate confidence is low, **escalate, never
    cheapen** — cheap to do on the effort dial.
- **Touch-points:** `orchestrator/parallel_executor.py`, the effort/thinking params on
  the provider adapters + `backends/capabilities.py` (capability matrix),
  `config/models.py`; **remove** `routing/router.py`, `routing/tiers.py`,
  `plugin/orchestration/router.py`.
- **Acceptance:** mechanism properties **2, 3, 11** hold — a high-stakes short unit
  gets higher effort; an uncertain estimate escalates effort; a live (non-test) call
  site applies the effort decision on an effort-capable backend, and falls back to
  *advise* on a CLI runtime that exposes no effort control (both asserted by tests).
- **Effort:** ~1–2 weeks. **Ships alone** — converts "no decision / fixed tier"
  into a principled, safe, effort-first one.

### P1 — Honest, length-decoupled inputs + waste visibility
- **Tasks:**
  - **Honest inputs (mechanism, properties 4, 5, 8, 10):** replace fabricated
    `tool_dependencies` placeholders with measured signals; reweight so token
    length doesn't dominate difficulty; guarantee monotonic, cliff-free mapping;
    emit the real inputs and driving axis.
  - **Waste retrospective (frugality):** a frugality aggregator joins the
    event-sourced cost/token signals with AC outcomes, tier history, and stagnation
    events to compute `total_cost`, `avoidable_cost`, a breakdown (`rework`,
    `dead_escalation`, `stagnation`), and **per-stage attribution** (interview /
    execute / consensus). Emit via `observability/retrospective.py`.
  - **Surface it:** a non-judgmental waste line in the run summary (CLI + the
    journey progress block from [#02](./02-journey-transparency.md)) and a TUI panel
    — "~$0.40 of ~$1.10 went to re-work; biggest contributor: AC-7 escalated twice
    without progress."
- **Touch-points:** the **estimator** module (`routing/complexity.py`, rebuilt as the
  difficulty+stakes estimator — *not* the deleted tier router), new
  `observability/frugality.py`, `observability/retrospective.py`, `cli/status.py`,
  `tui/cost_tracker.py`.
- **Acceptance:** "fix the race in the lock-free queue" rates **hard**; a long
  boilerplate unit rates **easy**; a run with a known re-done AC reports non-zero
  `rework` waste; a clean run reports ~0 avoidable; a long first-try-successful AC
  is **not** flagged (floor-preserving).

### P2 — Calibrate, learn, compound (behind one dial)
- **Tasks:**
  - **Cross-run calibration (mechanism, properties 6, 7):** feed the recorded
    success/cost/rework stream back to recalibrate weights/thresholds; replace
    hardcoded tier multipliers with a current, updatable cost model.
  - **Learned waste guardrails (frugality):** persist recurring waste patterns
    keyed by domain/AC-class (`auto/domain_profile.py`, `auto/task_classes.py`) and
    feed them into the **effort decision's starting point** and the
    decomposition defaults. Each guardrail is auditable and reversible.
  - **One user-held dial (property 9 + #1377 assurance dial):** a single explained
    lever spanning economize-vs-assure — tier appetite *and* the consensus/
    generation assurance tradeoff (consensus on every AC vs. only risky ones; 1
    generation vs. 3). Never an automatic, invisible cut.
- **Touch-points:** `routing/`, `auto/domain_profile.py`, `config/models.py`,
  `evaluation/`/`consensus`.
- **Acceptance:** a unit that failed at tier T shifts similar future routing; tier
  boundaries are inspectable and tied to observed outcomes; the avoidable/total
  ratio drops across repeated runs in a domain; moving the one dial visibly changes
  routing *and* assurance behavior and is never applied silently.

## Metrics / success signals

- A live, non-test call site applies the **effort decision** on an effort-capable
  backend, with a recorded *advise* fallback on CLI runtimes (properties 1, 11 — binary).
- `avoidable_cost / total_cost` per run, trending **down** across repeated runs in
  a domain (the compounding signal).
- Under-powering incidents (hard unit → weak tier → silent failure/rework) → down;
  correlates with [#01](./01-runtime-provider-reliability.md)'s zero-token / rework
  signatures.
- Share of trivial units routed below Frontier **without** an evaluation pass-rate
  regression (the floor-preserving guarantee).
- Estimate accuracy improving across runs (property 7).

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Wiring routing in regresses today's "always capable" behavior | P0 is safety-asymmetric: cheapen *only* on confident-trivial; default and uncertainty both escalate. Ship behind a flag; compare eval pass-rate before/after. |
| "Waste" mislabels legitimate exploration as waste | Conservative definition (only spend on steps that failed to advance a *verified* AC); bias to *not* flagging; show methodology so users can judge. |
| `estimated_cost_usd` is an estimate, not billed truth | Label "estimated"; prefer real backend usage where returned; never present false precision. |
| Cold-start: no calibration data yet | Ship principled defaults in P0/P1; calibration (P2) refines, is not a prerequisite. |
| Learned guardrails over-fit and under-power tasks | Guardrails bias defaults only, never cap; auto-escalation remains the safety net; reversible and logged. |

## Non-goals

- Hard budget caps or spend limits (explicitly rejected — floor-preserving only).
- A perfect difficulty oracle ("better than length, and safe when unsure" is the
  P0 bar) or a billing-accurate cross-provider cost model (estimates suffice for
  *relative* waste).
- Fixing execution-level waste directly (re-reads, retries) — that is the runtime/
  backend's; core only advises or routes.

## Open questions

- **Default error bias:** escalate-on-uncertainty (the #1384 author's position) —
  adopted as the P0 default.
- **Calibration vs. cold-start; single risk dial vs. per-domain stakes** — resolve
  in P2; recommendation: one global dial in v1, per-domain later.
- **"Verified AC advancement"** sourced from the evaluation verdict
  ([#06](./06-atomicity-decomposition-reliability.md)/`harness` Deliver gate) or
  AC-tree completion events? Likely both — cross-check.
- **Guardrail scope:** project-scoped `.ouroboros/` (recommended, avoids
  cross-project contamination) vs. user dir.
- **One RFC or split?** **Resolved (owner, 2026-06-11): split.** Per #1384, one RFC
  for the **estimator** (difficulty + stakes, measured inputs only) and one for the
  **actuator / wiring** (effort dial + capability matrix + escalation ladder); per
  #1377, the frugality slices are separate issues against the shared control-loop
  frame — first the **spend-attribution + advisory-guardrail** slice (P1), then the
  **adaptive-concurrency** evolution of `backend_limits`. Calibration (P2) is v2.
