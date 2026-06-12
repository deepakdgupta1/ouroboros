# Ouroboros — Investment Plans

This directory holds dedicated **strategy + plan** documents for the highest-
leverage investment areas in Ouroboros. It covers two sources, kept in one place
because they overlap heavily:

1. The **top-5 areas** from the architecture analysis in
   [`../ouroboros-architecture-value-map.md`](../ouroboros-architecture-value-map.md) (§11).
2. Two **mechanism-level investment cases** raised as GitHub discussions —
   [#1384](https://github.com/Q00/ouroboros/discussions/1384) (complexity→tier)
   and [#1385](https://github.com/Q00/ouroboros/discussions/1385)
   (atomicity/decomposition) — which name decisions the top-5 depend on but did
   not themselves cover.

The complexity→investment mechanism (#1384) is the *spend decision* behind frugality
(reshaped by the owner to a **reasoning-effort dial**, not tier switching), so it is
**merged into Plan #4** as a single workstream rather than a standalone doc
(see [#04](./04-frugality-and-spend-decision.md) for the rationale). That leaves
**six** plans. Each is self-contained: problem, current code state, target,
strategy, and a phased baby-steps plan with file touch-points and acceptance
criteria.

## Owner direction — all four greenlit (2026-06-11)

The owner ([Q00](https://github.com/Q00)) responded to all four discussions and
**greenlit the direction**, with reshapes now encoded in each plan's *Owner
direction* section. We are moving forward with the RFCs on this basis.

| Discussion | Verdict | Key reshape | First slice |
|---|---|---|---|
| [#1376](https://github.com/Q00/ouroboros/discussions/1376) usability | Greenlit; **first RFC merged** ([#1392](https://github.com/Q00/ouroboros/pull/1392)) | Breadcrumb = **state → action**, not "Step N of M"; TUI is a decoupled, **runtime-agnostic** observer; cockpit → `ourocode` | Breadcrumb + `ouroboros tui open` (**merged**); `ooo config` is the next slice (**#3**) |
| [#1377](https://github.com/Q00/ouroboros/discussions/1377) frugality | Greenlit | Four themes = **one control loop** (sensor → controller → policy → learning); fan-out discipline is core's (`backend_limits` + [#1372](https://github.com/Q00/ouroboros/pull/1372) already serialize delivery) | **Spend attribution + advisory guardrail loop**; adaptive-concurrency is the 2nd slice |
| [#1384](https://github.com/Q00/ouroboros/discussions/1384) complexity→invest | Greenlit (strongest thread) | Actuator = **reasoning-effort dial, not model switching**; **`PALRouter` removed, not wired**; properties restructured + new *runtime-scoped applicability* | Effort-first decision + stakes axis + escalate-on-uncertainty; **split: estimator RFC + actuator RFC** |
| [#1385](https://github.com/Q00/ouroboros/discussions/1385) atomicity/decomp | Greenlit; **re-grounded** | The dissected modules are **off the live path** (live executor = `orchestrator/parallel_executor.py`); **delete `execution/atomicity.py` + `decomposition.py`**, don't repair; attempt-then-bounce confirmed + new *live-path-verification-first* property | Delete dead modules + decomposition verifier re-aimed at `parallel_executor.py` |

**Two cross-cutting reshapes** the owner applied to both mechanism threads
(#1384/#1385): (1) the ten "necessary conditions" are **restructured** into v1 safety
invariants vs. v2 maturity goals (calibration / closed-loop collide with cold-start);
(2) **delete-don't-repair** unwired machinery (`PALRouter`; the DoubleDiamond
atomicity/decomposition modules) — fixing code nothing calls is motion, not progress.

| # | Plan | Primary value | Effort | Risk | Status |
|---|------|---------------|--------|------|------------|
| 1 | [Runtime / Provider Reliability Contract](./01-runtime-provider-reliability.md) | Stops catastrophic run-killers (the 0-token mass failure) | Med | Low | **Partially landed**; P2 reframed concurrency-aware |
| 2 | [Journey Transparency](./02-journey-transparency.md) | Adoption/retention for non-technical first-timers | Low–Med | Low | **First RFC merged (#1392)** |
| 3 | [Configuration Coherence](./03-configuration-coherence.md) | Ends the "fixed-but-still-broken" loop | Low (warn) | Low | **Greenlit — next #1376 slice** |
| 4 | [Frugality & the Spend Decision](./04-frugality-and-spend-decision.md) | Funds frugality *and* reliability — stance + spend mechanism + visibility | Med–High | Med | **Greenlit; effort-dial reshape, split RFCs** |
| 5 | [Trustworthy Autonomous Runs](./05-trustworthy-autonomous-runs.md) | The marquee `ooo auto` promise | Med | Med | **Partially landed** |
| 6 | [Atomicity & Decomposition Reliability](./06-atomicity-decomposition-reliability.md) | The gate in front of fan-out; stops compounding mis-splits | Med–High | Med | **Greenlit; re-grounded on `parallel_executor`** |

## Discussion coverage matrix

These plans are written to **solve for** four GitHub discussions. Traceability:

| Discussion | Theme | Solved by |
|---|---|---|
| [#1376](https://github.com/Q00/ouroboros/discussions/1376) | Usability & transparency | **#2** (journey/use/during-run) · **#3** (configure/setup) · **#5** (automation/resume) |
| [#1377](https://github.com/Q00/ouroboros/discussions/1377) | Token frugality (stance + themes) | **#4** (stance, attribution, guardrail loop, assurance dial) · **#1** (completion-feasibility pre-flight) |
| [#1384](https://github.com/Q00/ouroboros/discussions/1384) | Complexity → **effort** mechanism | **#4** (the *spend-decision* half; properties restructured v1/v2; **effort dial, not tier router**) |
| [#1385](https://github.com/Q00/ouroboros/discussions/1385) | Atomicity & decomposition reliability | **#6** (dedicated; **re-grounded on `parallel_executor.py`**; properties restructured v1/v2) |

Each plan's header table carries an **Addresses** row, and the originals gained a
**Discussion alignment** section mapping the discussion's specific asks to phases.
Code claims in #1384/#1385 were re-verified against current `main` before being
encoded (e.g. `decomposition.py` `MAX_DEPTH = 2` vs. its docstring's "5 levels";
the tier router having no non-test call site). The owner's own verification went
one level deeper and found that the **execution modules these claims dissect
(`double_diamond.py`, `atomicity.py`, `decomposition.py`) are themselves off the
live path** — the live executor is `orchestrator/parallel_executor.py` — which is
why the #1384/#1385 fixes are *delete + re-aim*, not repair (see each plan's *Owner
direction*).

## Shared principles (apply to every plan)

Inherited from the project's working principles and the four discussions; not
re-argued in each document:

- **Baby-steps over big-bang.** Every plan's **P0** is shippable on its own and
  delivers value without P1/P2.
- **Floor-preserving frugality.** Optimize *wasted* motion (tokens that did not
  advance a *verified* acceptance criterion), never the comprehensiveness the user
  is paying for. Two hard invariants (#1377): **never reduce the achieved
  outcome**; **never increase rework risk.** Frugality is a learned guardrail,
  never a budget cap, and never halts a run mid-way.
- **Typed contracts over heuristic reconstruction.** Where a producer and consumer
  disagree on a schema at a boundary, fix the contract — don't add another regex.
  (The unifying thesis of the RCA.)
- **Honest outputs, calibrated thresholds, faithful rationale.** The shared
  failure pattern behind #1384 and #1385: *fabricated quantitative outputs and
  uncalibrated constants.* Every recorded quantity must be the one actually used
  in the decision; every load-bearing constant must be justified by observed
  outcomes; every decision must emit its real inputs and confidence.
- **Layered Agent-OS ownership.** Core (`ouroboros`) *decides and routes* — how
  much work, at what fidelity, whether/how to split, and how much **reasoning effort**
  to commit *where the backend permits* (per #1384's runtime-scoped applicability, it
  can only **advise** a CLI runtime that exposes no effort control). It can only
  **advise or route** the agent runtime + LLM backend on execution-level efficiency
  (re-reads, retries, regeneration). The rich always-on dashboard is shell
  (`ourocode`) territory. No strong plugins angle in any of these six.
- **Non-technical first-timer is the primary persona.** When a trade-off is close,
  optimize for the user who cannot read the code or the ~50 env vars.

## Status legend

| Status | Meaning |
|--------|---------|
| `Proposed` | Drafted, not yet scheduled |
| `In flight` | Partially landed on a branch / recent commits |
| `Blocked` | Waiting on a decision in "Open questions" |

## Evidence base

- [`../usage-driven-feedback/usage-issues.md`](../usage-driven-feedback/usage-issues.md)
  — field log of every failure/friction point over a 2-day build.
- [`../usage-driven-feedback/rca-and-remediation-plan.md`](../usage-driven-feedback/rca-and-remediation-plan.md)
  — code-verified root-cause analysis with `file:line` citations.
- [`../improvement-themes.md`](../improvement-themes.md) — journey / methodology /
  configuration opacity themes.
- [`../ouroboros-architecture-value-map.md`](../ouroboros-architecture-value-map.md)
  — the module map these plans act on.
- GitHub discussions [#1376](https://github.com/Q00/ouroboros/discussions/1376),
  [#1377](https://github.com/Q00/ouroboros/discussions/1377),
  [#1384](https://github.com/Q00/ouroboros/discussions/1384),
  [#1385](https://github.com/Q00/ouroboros/discussions/1385) — the direction-checks
  and investment cases these plans answer.

> **A note on "in flight."** Recent commits on `main` already moved parts of #1, #4,
> and #5 — notably `orchestrator/backend_limits.py` and [#1372](https://github.com/Q00/ouroboros/pull/1372)
> (rate-budget pacing), which together neutralize the 14-AC stampede, and RFC
> [#1392](https://github.com/Q00/ouroboros/pull/1392) (the first #1376 slice). Each
> plan opens with an honest **Current state** section that separates *what has landed*
> from *what remains*, so none of this work is duplicated.

## Suggested sequencing (baby-steps)

1. **Quick trust wins first:** the P0s of **#2** (progress on `job_status`) and
   **#3** (detect-and-warn on stale config) — low-risk, read-only/detect-only.
2. **Complete the in-flight hardening:** finish **#1** (capability negotiation +
   completion-feasibility pre-flight) and **#5** (authoritative reconciliation
   sweep) in parallel.
3. **Fund the spend decision:** **#4** P0 (wire the **effort-first** decision +
   stakes axis + fail-safe; remove `PALRouter`) and **#6** P0 (**delete** the
   off-path `atomicity.py`/`decomposition.py` + verify splits in `parallel_executor`)
   — together they make honest frugality possible.
4. **Make frugality visible and compounding:** **#4** P1–P2, which read the
   now-honest signals from #4's mechanism and #6.
