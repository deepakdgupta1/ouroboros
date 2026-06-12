# Investment #2 — Journey Transparency ("de-blackbox the run")

| | |
|---|---|
| **Status** | **Greenlit; first-slice RFC merged ([#1392](https://github.com/Q00/ouroboros/pull/1392))** — breadcrumb reshaped to *state → action*; TUI confirmed runtime-agnostic. Substrate already landed. See [Owner direction](#owner-direction--greenlit-first-slice-merged-2026-06-11). |
| **Date** | 2026-06-11 |
| **Author** | deepak |
| **Owner** | TBD |
| **Effort / Risk** | Low–Medium / Low (the read-model exists; this exposes it) |
| **Persona** | Primary: non-technical first-timer (the documented acute pain) |
| **Addresses** | [discussion #1376](https://github.com/Q00/ouroboros/discussions/1376) (journey opacity, Use / During-a-run rows) |
| **Related** | [#03 Configure](./03-configuration-coherence.md) · [#05 Automation front-door](./05-trustworthy-autonomous-runs.md) · [Architecture Value Map §11.2](../ouroboros-architecture-value-map.md) · [improvement-themes §1](../improvement-themes.md) · memory: *Ouroboros UX friction* |

## TL;DR

Give every user one always-available answer to *"what is happening now, what
comes next, and how far along am I?"* The expensive part — a schema-versioned
Run/Stage/Step projection over the event store — **already exists**. The gap is
that the thing hosts actually poll (`job_status`) returns only a status string
and an **opaque integer cursor**. This plan distills the projection into a
structured progress payload and surfaces it everywhere.

## Owner direction — greenlit; first slice merged (2026-06-11)

The owner ([Q00](https://github.com/Q00)) confirmed the framing ("the grounding is
exactly the kind of evidence that makes a usability thread actionable") and **the
first slice is already drafted and merged** as RFC [#1392](https://github.com/Q00/ouroboros/pull/1392)
(`docs/rfc/transparency-breadcrumb-tui.md`). We move forward on that basis.

- **Appetite: yes. Split slice-by-slice, not one RFC.** #1392 is the first slice
  (breadcrumb + TUI surfacing); **`ooo config` / change-surfacing is the next slice
  with its own RFC** — see [#03](./03-configuration-coherence.md). (The owner: "we've
  been bitten by the silent-MCP-reconnect problem ourselves, so that report rings true.")
- **Breadcrumb reshaped: state → action, *not* a step counter.** Ouroboros is an
  evolutionary loop, not a pipeline — "Step 2 of 5" would lie the moment you re-enter
  the loop (a second generation, a re-interview after drift). Every `ooo` skill instead
  ends with a one-line **`◆ current state → next: recommended action`** footer derived
  from live session state. This is the binding change to P1 below.
- **The TUI is better-positioned than this plan assumed: a fully decoupled,
  runtime-agnostic observer.** It *polls the shared event store* rather than attaching
  to a run — so a run driven from Claude Code, Hermes, or any other runtime is equally
  visible. Surfacing it is therefore a **cross-runtime transparency win**, not just a
  convenience. #1392 ships **`ouroboros tui open`** — terminal-aware (`$TERM_PROGRAM`
  detection: Ghostty / iTerm2 / Apple Terminal verified on macOS) spawning the monitor
  in a new window, with an honest copyable-command fallback for headless/SSH — plus a
  **TUI auto-offer in `ooo run`** (`tui_autolaunch`), with the in-session progress relay
  as the baseline.
- **Core vs shell confirmed.** The always-on cockpit (all settings in one place,
  few-click parity with the TUI) belongs to **`ourocode`** (the shell). **Core owns the
  primitives it builds on:** the event store, the status projections, the TUI itself,
  and the surfacing hooks. The deferred browser dashboard stays deferred and lands in
  `ourocode`.
- **Open offer:** the owner invited claiming any slice and opening a PR; the private
  friction log would most help the breadcrumb wording and the `ooo config` follow-up.

## Discussion alignment (#1376)

[Discussion #1376](https://github.com/Q00/ouroboros/discussions/1376) is the
usability/transparency direction-check; its key insight is that most of this is
**surfacing and connecting things Ouroboros already has** — low-risk, incremental
work, not net-new infra. It names three opacity levels; this plan owns **journey
opacity** (the other two are split out):

| #1376 journey stage | Lands in this plan | Or cross-referenced to |
|---|---|---|
| **Use** — a persistent "you are here" breadcrumb (reshaped per owner to *state → recommended action*, e.g. `◆ Seed ready → next: ooo run` — **not** a "Step N of M" counter, which lies in an evolutionary loop), and a one-line "why we're asking / what this unlocks" on interview questions and gates | **P1** (breadcrumb) + **P2** (gate rationale) | — |
| **During a run** — make the existing TUI discoverable (bundle, auto-offer/launch, auto-attach) + a compact in-session progress digest | **P0** (digest) + **P2** (TUI discoverability) | — |
| **Install / setup** — an upfront "path-to-success" map | **P2** (flow map) | smoke-check of the chain → [#03](./03-configuration-coherence.md) |
| **Configure** — effective-config view + change surfacing | — | [#03](./03-configuration-coherence.md) |
| **Automation** — `ooo auto` as default front door + resume affordance | — | [#05](./05-trustworthy-autonomous-runs.md) |

Net-new tasks pulled in from #1376 are folded into the phases below (breadcrumb,
gate rationale, TUI auto-offer, flow map). The **deferred bigger bet** — a browser
dashboard — stays deferred and is naturally shell (`ourocode`) territory, not core.

## Problem (user-visible pain)

Ouroboros "feels like a black box." The memory note records it directly:
*"Layperson users get lost; no workflow visibility or progress tracking."*
[`improvement-themes.md`](../improvement-themes.md) names **journey opacity** as
the first and most acute pain: at any moment it is unclear what is happening,
what comes next, and how far along the path to success the run is.

The field logs show the symptom concretely: a user watching a stuck run could
only see the **event cursor move 293 → 296** with "zero worker activity" — an
integer that means nothing to a human and could not distinguish *progress* from
*a zombie* (usage issues §3.1). The only rich visibility today is the TUI, which
a non-technical first-timer in an AI-CLI session never opens.

## Current state (what has landed vs. what remains)

**Landed (the substrate — do not rebuild):**

- `harness/projection.py` — immutable, schema-versioned **Run / Stage / Step /
  Artifact / Verdict** records (the public read-model).
- `harness/projection_builder.py` — `ProjectionBuilder` / `build_projection()`
  that walks `EventStore` events into those records (LLM/tool call pairing, AC
  scoping, step kinds).
- `orchestrator/workflow_state.py` — already tracks `phase`, `estimated_tokens`,
  and `estimated_cost_usd` per workflow.
- `ouroboros_query_projection` MCP tool exists, and the TUI renders AC trees,
  drift, cost, and lineage.

**Remaining (this plan):**

- `mcp/job_manager.py` `JobSnapshot` exposes only `status` + `cursor` (an opaque
  event count). There is **no** `{phase, ac_done/ac_total, current_step,
  next_action, est_cost}` summary on the path hosts poll.
- `ouroboros_job_status` therefore cannot answer the human question; the rich
  projection is built but **not distilled** for the polling consumer.
- No compact, host-agnostic "progress line" exists outside the TUI.

## Target state (definition of done)

- `ouroboros_job_status` / `ouroboros_job_wait` return a **structured progress
  block** every host can render with no extra work:
  `{ phase, step_label, ac_done, ac_total, percent, next_action, est_cost_usd,
  is_progressing }`.
- A one-line human-readable progress string is available from the CLI
  (`ouroboros status execution <id>`) and inside MCP sessions.
- "What comes next" is explicit, not inferred — the user always sees the named
  upcoming phase/step.

## Strategy

This is a **surfacing** problem, not a measurement problem. The principle:
*compute the human-facing summary once, in the projection layer, and let every
consumer read the same value* (the harness exists precisely so consumers don't
reinvent status formats). Three moves:

1. **Distill, don't re-derive.** Add a `progress` projection that reduces the
   Run/Stage/Step records + `workflow_state` into the structured block above.
2. **Put it on the polling path.** Carry that block in `JobSnapshot` so
   `job_status`/`job_wait` return it — the single highest-traffic surface.
3. **Render it everywhere cheaply.** A shared formatter turns the block into a
   one-line string for CLI and MCP text output; the TUI already has the rich view.

## Plan (phased, baby-steps)

### P0 — Structured progress on the polling path
- **Tasks:** Add a `ProgressSummary` derived from `build_projection()` +
  `workflow_state` (`phase`, `ac_done/ac_total`, `current_step`, `percent`,
  `est_cost_usd`, `is_progressing`). Embed it in `JobSnapshot` and return it from
  `ouroboros_job_status` / `ouroboros_job_wait`.
- **Touch-points:** `harness/projection_builder.py` (or a new
  `harness/progress.py`), `mcp/job_manager.py`, `mcp/job_handlers.py`.
- **Acceptance:** polling a running job returns a populated progress block;
  `ac_done/ac_total` matches the AC tree; `is_progressing=false` coincides with
  the watchdog's no-progress signal (ties directly to Plan #5).
- **Effort:** ~2–3 days. **Ships alone — every host benefits immediately.**

### P1 — Human progress line + persistent "you are here" breadcrumb
- **Tasks:** A shared formatter renders the block as
  `"[Execute · Deliver]  AC 9/14 (64%) · building auth_middleware · ~$0.42 · advancing"`.
  Wire it into `ouroboros status execution <id>` and the MCP tools' text output.
  **Add the #1376 breadcrumb (reshaped per owner — state → action, per RFC [#1392](https://github.com/Q00/ouroboros/pull/1392)):**
  a persistent `◆ current state → next: recommended action` footer (e.g.
  `◆ Seed ready → next: ooo run`) derived from **live session state**, emitted across
  all `ooo` commands. Deliberately **not** a "Step N of M" counter — Ouroboros is an
  evolutionary loop, so a linear counter would lie on re-entry (second generation,
  re-interview after drift).
- **Touch-points:** `cli/status.py`, `cli/progress.py`, `mcp/*handlers`, the skill
  dispatch path (`router/`, `skills/*/SKILL.md`).
- **Acceptance:** a non-technical user gets a readable status line without opening
  the TUI; every `ooo` command shows the current step and the suggested next one.

### P2 — "What's next", gate rationale, TUI discoverability, flow map
- **Tasks:**
  - Surface the named upcoming phase/step from the workflow plan (`workflow_ir`),
    and a coarse ETA from historical phase durations (`baseline_metrics`).
  - **Gate rationale (#1376):** a one-line "why we're asking / what this unlocks"
    on interview questions and gates, so prompts aren't opaque.
  - **TUI discoverability (#1376; specified by RFC [#1392](https://github.com/Q00/ouroboros/pull/1392)):**
    ship **`ouroboros tui open`** — a terminal-aware launcher (`$TERM_PROGRAM`
    detection: Ghostty / iTerm2 / Apple Terminal) that spawns the monitor in a new
    window, with a copyable-command fallback for headless/SSH — and **auto-offer/launch
    it at run start** (`tui_autolaunch`). Note (per owner): the TUI is a *decoupled,
    runtime-agnostic observer* that **polls the shared event store**, so it does not
    "attach" to a run — any runtime's run is equally visible.
  - **Path-to-success map (#1376):** an upfront flow map at install/first-run
    (what the whole journey is, roughly how long).
- **Touch-points:** `orchestrator/workflow_ir.py`, `orchestrator/baseline_metrics*`,
  `bigbang/interview.py` (gate rationale), `tui/app.py` + `cli/tui.py` (auto-offer),
  `skills/welcome`/`skills/tutorial` (flow map).
- **Acceptance:** the user sees the next named action before it happens; gates
  explain themselves; the TUI is offered without the user knowing it exists; a
  first-timer sees the whole path up front.

## Metrics / success signals

- Reduction in "is it stuck or working?" support questions (proxy: the §3.1
  confusion pattern).
- % of runs where a user can answer "what phase / how far" from a single tool
  call (target: 100% post-P0).
- Time-to-first-meaningful-status after launch.

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Progress % is misleading when ACs decompose mid-run (the denominator grows) | Report `ac_done/ac_total` as *current known* and show phase as the primary signal; treat % as secondary and label it "estimated." |
| Extra projection work slows the hot polling path | Build progress incrementally / cache by event cursor; the projection is already O(events) and read-only. |
| "Next action" is non-deterministic in agentic phases | Show the next *phase/gate* (deterministic), not a predicted tool call. |

## Non-goals

- A new GUI. This reuses the existing TUI and MCP/CLI surfaces.
- Precise time estimates — directional only.

## Open questions

- Should `ProgressSummary` be its own `schema_version`-bumped record in
  `harness/projection.py`, or a computed view that is not persisted?
- Do we gate the progress block behind a tool arg (opt-in) or make it default on
  `job_status`? (Recommendation: default on — it is small and universally useful.)
