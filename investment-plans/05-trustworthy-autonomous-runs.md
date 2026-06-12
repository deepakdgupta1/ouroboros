# Investment #5 — Trustworthy Autonomous Runs (`ooo auto` / `ralph`)

| | |
|---|---|
| **Status** | In flight (partial) |
| **Date** | 2026-06-11 |
| **Author** | deepak |
| **Owner** | TBD |
| **Effort / Risk** | Medium / Medium (touches liveness + persisted state) |
| **Persona** | Power users running unattended work; the product's marquee promise |
| **Addresses** | [discussion #1376](https://github.com/Q00/ouroboros/discussions/1376) (Automation row) |
| **Related** | [#02 Self-reporting progress](./02-journey-transparency.md) · [Architecture Value Map §11.5](../ouroboros-architecture-value-map.md) · [RCA S](../usage-driven-feedback/rca-and-remediation-plan.md) · [Usage issues §3.1, §3.5](../usage-driven-feedback/usage-issues.md) |

## TL;DR

Make unattended, multi-hour, cross-restart runs dependable enough to trust
overnight. Two remaining gaps after recent zombie-reconciliation work:
(1) reconciliation is **lazy (on snapshot read)**, not an **authoritative
startup/reconnect sweep**; and (2) **resume state is opaque** — "no in-flight
sessions to resume" never explains *why*. This is what converts `ooo auto` from
"impressive demo" into "I trust it."

## Discussion alignment (#1376)

[Discussion #1376](https://github.com/Q00/ouroboros/discussions/1376)'s
**Automation** row asks for two things this plan owns the trust-foundation for:

- **`ooo auto` as the default front door, with self-reporting progress.** The
  *progress* mechanism is [#02](./02-journey-transparency.md) (it makes `ooo auto`
  report itself); this plan makes the underlying autonomous run *dependable enough
  to be the default* — no zombies, honest resume. Front-door + self-report only
  earns trust if the run beneath it is trustworthy.
- **A one-touch "what's in flight / resume" affordance** when a run is blocked or
  paused. Folded into **P1** below — the resume-state surface answers "what's in
  flight" and offers the exact re-attach command in one step.

So #1376's automation ask is split cleanly: **#02 reports progress, #05 makes the
run dependable and its resume state legible.**

## Problem (user-visible pain)

`ooo auto` and `ooo ralph` are the flagship "set it and forget it" workflows —
but the field logs show the two ways that trust breaks:

- **Zombie jobs (§3.1).** The MCP server restarted mid-build; the hermes workers
  died, but the DB row stayed `running` forever at **0/14**, requiring manual
  cancellation. The control plane (persisted status) and data plane (real worker
  liveness) were reconciled opportunistically, not authoritatively (RCA S).
- **Opaque resume (§3.5).** Attempts to resume found *"There are no in-flight
  Ouroboros sessions to resume"* — with no indication whether prior runs
  completed, failed, were cancelled, or were silently orphaned. A dead end with
  no explanation.

For a workflow whose entire value proposition is *walking away*, "it might be a
zombie and I can't tell" is fatal to trust.

## Current state (what has landed vs. what remains)

**Landed (do not redo):**

- `mcp/job_manager.py` now records the **owning process identity (pid + start
  time)** on job creation, and on **snapshot read** reconciles a non-terminal job
  into a synthetic `INTERRUPTED` event when the owner is provably dead and no live
  runner remains (commit `f1e36f30`). Liveness is recycling-safe via
  `heartbeat.is_process_identity_alive`. Conservative: legacy jobs without
  recorded identity, and live/other-process owners, are never reconciled.
- Reconciliation is **deferred while a linked session is still alive**
  (commit `ea8aaa44`).
- `auto/state.py` already models a **resume capability matrix**
  (`ResumeCapability`: `NO_RESUME` / `PARTIAL_RESUME` / `RESUME`),
  `TERMINAL_PHASES`, valid phase transitions, and per-phase timeouts with env
  overrides. `auto/recovery_plan.py` exists for blocked-run recovery.
- `runtime/watchdog.py` + `runtime_controls.*` bound idle / no-progress / safety
  timeouts.

**Remaining (this plan):**

1. Reconciliation only runs when **a specific job's snapshot is read**
   (`_reconcile_orphaned_job_snapshot`). There is **no startup/reconnect sweep**
   that enumerates *all* `running` jobs and reconciles orphans proactively — so a
   zombie that no one queries stays `running` (RCA P2 is only half-done).
2. The resume capability is **computed but not surfaced**: the user is told
   "nothing to resume" without the *reason* drawn from the capability matrix.
3. Reconciliation outcomes are not reported back to the user ("job X was orphaned
   by a server restart and marked interrupted; resume with …").

## Target state (definition of done)

- On MCP server startup/reconnect, **every** `running` job is checked against its
  owning-process lock and orphans are transitioned to a terminal/recoverable
  state — without waiting for someone to read that job.
- `ooo resume-session` (and `ouroboros resume`) **always explains** the resume
  state: in-flight / completed / failed / cancelled / orphaned-and-reconciled,
  per session, with the exact re-attach command where applicable.
- When a job is reconciled, the user is **told what happened and what to do next**.

## Strategy

The RCA frames this precisely: *liveness exists but isn't authoritative.* The fix
is to make the persisted control plane and the real data plane reconcile
**proactively and visibly**, reusing the machinery that already landed. Principles:

1. **Authoritative, not opportunistic.** Generalize the proven on-read reconcile
   into a sweep that runs at the natural trust boundary (server start/reconnect).
2. **Never a silent dead end.** Every resume/recovery path explains itself using
   the capability matrix that already exists.
3. **Stay conservative.** Keep the landed safety properties — identity-gated,
   recycling-safe, idempotent, no reconciliation of live or other-process owners.

## Plan (phased, baby-steps)

### P0 — Authoritative startup/reconnect reconciliation sweep
- **Tasks:** On MCP server start/reconnect, enumerate all non-terminal jobs and
  run the existing `_reconcile_orphaned_job_snapshot` logic across each, emitting
  the synthetic `INTERRUPTED` event for provably-dead owners. Reuse the landed
  identity + heartbeat checks; keep idempotency (deterministic event id) and the
  read-only-store no-op.
- **Touch-points:** `mcp/job_manager.py` (extract the per-job reconcile into a
  sweep), MCP server connect path (`mcp/server`/`manager.py`).
- **Acceptance:** kill a runner mid-job, restart the server, and **without reading
  that job** it is reconciled to `interrupted` within the sweep; live and
  other-process jobs are untouched (regression tests mirror the existing
  conservative cases).
- **Effort:** ~2–3 days. **Ships alone — directly closes §3.1.**

### P1 — Resume-state transparency + one-touch "what's in flight"
- **Tasks:** Make `ooo resume-session` / `ouroboros resume` always report, per
  session, the resume classification from `auto/state.py` and *why*: terminal
  (completed/failed/cancelled), orphaned-and-reconciled, or resumable (+ exact
  command). When there is nothing to resume, say which of these applies.
  **Add the #1376 one-touch affordance:** when a run is blocked or paused, a
  single "what's in flight / resume" view lists in-flight sessions and offers the
  re-attach command directly — the user does not assemble it by hand.
- **Touch-points:** `cli/resume.py`, `skills/resume-session/SKILL.md`,
  `mcp/job_handlers.py`, reading `auto/state.py` `resume_capability`.
- **Acceptance:** "nothing to resume" is replaced by an explained list; a
  partially-completed auto session shows `PARTIAL_RESUME` with the resume command;
  a blocked/paused run is one action away from resuming.

### P2 — Reconciliation reporting + handoff clarity
- **Tasks:** When the P0 sweep reconciles a job, surface a user-facing note (via
  the Plan #2 progress channel) explaining the orphaning and the recovery path.
  Tighten the `auto/handoff_contract.py` unknown-status guidance so `detached` /
  `blocked` / `failed` outcomes are never confused with dispatch failure.
- **Acceptance:** after an orphaning event, the user sees a plain-English
  explanation and a next step, not a silent status flip.

## Metrics / success signals

- Zombie jobs requiring **manual** cancellation → 0 (the §3.1 signature).
- % of `resume` invocations that return an **explained** state (target: 100%).
- Mean time from "owning process died" to "job in a terminal/recoverable state"
  (bounded by sweep cadence, not by whether someone reads the job).

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| A sweep races a legitimately slow-but-alive runner and falsely interrupts it | Reuse the landed liveness gate (identity + heartbeat + linked-session-alive defer); only reconcile *provably dead* owners; never mid-task. |
| Sweep cost on startup with many historical jobs | Scope to non-terminal jobs only; the set is small by construction; index on status. |
| Resume classification misreads a complex partial state | Drive entirely from the existing `auto/state.py` matrix (single source of truth) rather than ad-hoc inference. |

## Non-goals

- Distributed/multi-host job ownership. Single-machine local-first assumptions
  hold (matches the heartbeat PID-lock model).
- Changing the auto state machine's phases — this surfaces and reconciles them,
  it does not redesign them.

## Open questions

- Sweep trigger: only on server start/reconnect, or also on a low-frequency timer
  during long idle periods? (Recommendation: start/reconnect first; add a timer
  only if zombies are observed between reconnects.)
- Should a reconciled-orphan auto session attempt **auto-resume**, or always wait
  for explicit user confirmation? (Recommendation: explain + wait, per the
  "never silently change the execution envelope" control-contract invariant.)
