# Improvement Themes — Boosting Ouroboros Usability & Transparency

| | |
|---|---|
| **Status** | **Greenlit by owner (2026-06-11)** — reshaped plans in [`investment-plans/`](investment-plans/README.md); first slice merged as RFC [#1392](https://github.com/Q00/ouroboros/pull/1392) |
| **Date** | 2026-06-08 (owner response folded in 2026-06-11) |
| **Author** | deepak |
| **Type** | Planning / conceptualization (no code in scope) |
| **Deliverable** | This document: proposed enhancement themes, the value each brings to users, and a suggested priority order |
| **Guiding principle** | **Baby-steps over big-bang.** Prefer changes that are faster, cheaper, and lower-risk. Lead with high-leverage, low-risk wins; explicitly defer larger bets. |
| **Primary persona** | **Non-technical first-timer** (the documented acute pain). Most proposals incidentally help power users too. |
| **Priority basis** | **User-journey order** — install → configure → first run → during-run → frugality → automation/evolve. |

---

> **Update (2026-06-11): the owner greenlit all four discussion threads** these themes
> seeded — usability [#1376](https://github.com/Q00/ouroboros/discussions/1376),
> frugality [#1377](https://github.com/Q00/ouroboros/discussions/1377), and the
> mechanism cases [#1384](https://github.com/Q00/ouroboros/discussions/1384) (complexity→invest)
> / [#1385](https://github.com/Q00/ouroboros/discussions/1385) (atomicity). The
> authoritative, reshaped plans now live in [`investment-plans/`](investment-plans/README.md),
> and the first usability slice (a *state → action* breadcrumb + `ouroboros tui open`)
> is **merged** as RFC [#1392](https://github.com/Q00/ouroboros/pull/1392). The most
> material owner reshapes are annotated inline below.

## 1. Why this document

When using Ouroboros today, it **feels like a black box**. That feeling shows up
at three concrete levels that are *in scope* here:

1. **Journey opacity** — at any moment it is unclear *what is happening now, what
   comes next, and how far along the path to success we are* — from install,
   through configuration, to actually building a project.
2. **Methodology opacity** — it is unclear how *optimal* the workflow is on two
   axes the user cares about:
   - **Frugality** — minimizing *wasted* tokens (motion that doesn't advance a
     verified acceptance criterion), never the comprehensiveness or completeness
     the user is paying for — a *floor-preserving* optimization learned over time
     as reusable guardrails (see §5 Stage E), not a budget cap.
   - **Automation** — minimizing the user's manual intervention across the
     whole project lifecycle.
3. **Configuration opacity** — Ouroboros does not proactively surface the
   relevant set of configurations (with their default values) for the user to
   review when setup/config changes occur — in Ouroboros itself, in the AI agent
   runtime, or in both.

This document catalogs those gaps and proposes thematic improvements, each
tagged with the value it delivers and an effort/risk read, ordered along the
user's journey.

---

## 2. Scope & non-goals

**In scope**

- Usability and transparency of the **end-to-end user journey** (install →
  configure → use → evolve).
- **Frugality** and **automation** as *user-visible, user-controllable*
  properties. Frugality is treated as **goal-subordinate and waste-only**, and as
  a **layered responsibility**: Ouroboros owns how-much-work, fidelity, and spec
  crispness; the runtime + backend own execution efficiency (see §5 Stage E).
- **Proactive configuration surfacing** with defaults, including when the AI
  agent runtime config changes.
- **Elevating the existing terminal dashboard** as the near-term visibility move.

**Explicit non-goals (deferred by decision, not oversight)**

- **Architecture-to-value mapping** ("what each internal component is and what
  value it gives the user"). Deliberately kept *out of scope* for this exercise
  and tracked as a separate effort.
- **A net-new web dashboard.** Noted as an aspiration (see [D3](#d3-deferred-web-dashboard-hermes-style)) but
  explicitly deferred under the baby-steps principle.
- **Reliability / correctness internals** (provider 429 handling, typed error
  contracts, zombie-job reconciliation, config-reload mechanics). These are
  already analyzed in [`usage-driven-feedback/rca-and-remediation-plan.md`](usage-driven-feedback/rca-and-remediation-plan.md)
  and [`usage-driven-feedback/usage-issues.md`](usage-driven-feedback/usage-issues.md). Where a usability
  proposal here depends on one of those fixes, it is **cross-referenced, not
  duplicated**.

---

## 3. What already exists (baseline)

Ouroboros is **not** starting from zero on transparency — which means most of the
work below is *surfacing and connecting* existing assets, not building new ones.
This is what makes the baby-steps path viable.

| Capability | Exists today? | Where | The gap (detail in §4) |
|---|---|---|---|
| First-touch onboarding (persona, MCP check, quick reference) | Yes | [`skills/welcome/SKILL.md`](skills/welcome/SKILL.md) | One-time; no persistent "you are here" during work |
| Guided setup wizard (6 steps, celebration checkpoints) | Yes | [`skills/setup/SKILL.md`](skills/setup/SKILL.md) | Doesn't surface effective config or restart/reconnect *why* |
| Pull-based status + drift + a single next-step hint | Yes | [`skills/status/SKILL.md`](skills/status/SKILL.md) | Reactive (user must ask), needs a session_id, not a persistent breadcrumb |
| Rich TUI dashboard (phase bar, AC tree, cost, drift, agents, lineage) | Yes | [`src/ouroboros/tui/`](src/ouroboros/tui/), [`docs/guides/tui-usage.md`](docs/guides/tui-usage.md) | Terminal-only, **separate terminal**, separate `[tui]` extra, manual launch + manual session pick |
| One-command automation pipeline | Yes | [`skills/auto/SKILL.md`](skills/auto/SKILL.md) | Background-job based; progress only surfaces if the agent keeps polling |
| Cost/token trackers, PAL model routing | Yes | [`src/ouroboros/tui/widgets/cost_tracker.py`](src/ouroboros/tui/widgets/cost_tracker.py), help §"PAL Routing" | No pre-run estimate, no budget ceiling, knobs buried in YAML |
| Config file with defaults | Yes | [`docs/config-reference.md`](docs/config-reference.md), `~/.ouroboros/config.yaml` | No in-session "effective config" review; change-needs-reconnect is silent |

> **Takeaway:** the dashboard, status, cost tracking, and automation pipeline
> already exist. The black-box feeling comes from **discoverability and
> surfacing**, not absence. That is exactly the kind of problem baby-steps
> solve well.

---

## 4. The gaps, by journey stage

Each gap is grounded in the documented experience (the internal UX-friction note
about first-time/non-technical users getting lost) and in the code/docs cited.

### Stage A — Install & Setup
- **A-gap-1 — No upfront map of the whole path to success.** A user runs
  `ooo setup` or `ooo interview` without a picture of the full
  install → configure → interview → seed → run → evaluate → evolve arc, how long
  it takes, or where results appear. The loop diagram is shown *once* in welcome
  ([`skills/welcome/SKILL.md`](skills/welcome/SKILL.md) Step 1) and then disappears.
- **A-gap-2 — "Done" is ambiguous.** Setup celebrates success
  ([`skills/setup/SKILL.md`](skills/setup/SKILL.md) Step 5) but the MCP server frequently needs a
  restart/reconnect to actually work, and nothing verifies the *full chain*
  (can it reach the runtime/LLM at all?). Cross-ref reliability: install
  fragility (`usage-issues.md` §2.1, §2.2).

### Stage B — Configuration
- **B-gap-1 — No single view of the *effective* configuration.** Settings are
  split across `~/.ouroboros/config.yaml` and the runtime's own file (e.g.
  `~/.hermes/config.yaml`), plus env overrides. The user never sees one
  consolidated "here is what is actually in effect, and here are the defaults"
  surface.
- **B-gap-2 — Config changes are not proactively surfaced for review.** When the
  user (or setup) changes Ouroboros or runtime config, nothing shows *what
  changed*, the new effective values, the defaults, or **whether a reconnect is
  required**. Some keys apply live, some only after `/mcp` reconnect, with no
  signal which — producing the "I already fixed it twice and it's still broken"
  loop. Cross-ref: [`rca-and-remediation-plan.md`](usage-driven-feedback/rca-and-remediation-plan.md) §R2/R5 and
  `usage-issues.md` §2.5. **This gap is the user's explicit ask.**
- **B-gap-3 — Cost/behavior-critical defaults are buried.** The knobs that most
  affect spend and behavior (model tier, parallel workers, retries, consensus)
  live in YAML, never offered for review at the moment they matter.

### Stage C — Journey awareness while working
- **C-gap-1 — No persistent "you are here."** There is no consistent breadcrumb
  telling the user which step they are on, what they just completed, and the
  single next command. `ooo status` has the *seed* of this (a `📍` next-step hint,
  [`skills/status/SKILL.md`](skills/status/SKILL.md) lines 73–76) but it is reactive and per-session.
- **C-gap-2 — No "why this / what it unlocks" framing.** Interview questions and
  quality gates don't explain *why* they're being asked or what passing unlocks,
  so a non-technical user can't tell signal from ceremony.

### Stage D — Visibility during a run (state & state changes)
- **D-gap-1 — The best visibility tool is hidden.** The rich TUI dashboard
  requires a *separate* `[tui]` extra, a *separate* terminal, a manual
  `ouroboros tui monitor` launch, and a manual session pick
  ([`docs/getting-started.md`](docs/getting-started.md) Step 3 lines 291–304; [`skills/welcome/SKILL.md`](skills/welcome/SKILL.md)
  lines 330–333; [`docs/guides/tui-usage.md`](docs/guides/tui-usage.md) lines 7–16). A non-technical user
  will almost never find it, so the run *feels* like a black box even though a
  glass box exists.
- **D-gap-2 — In-session progress is thin for `auto`/`run`.** `ooo auto` is
  background-job based and returns IDs; progress only appears if the agent keeps
  polling ([`skills/auto/SKILL.md`](skills/auto/SKILL.md) "keep ownership of the conversational
  UX"). When it doesn't, the user is left guessing.

### Stage E — Frugality (tokens)
- **E-gap-1 — No pre-run feasibility or cost expectation.** Nothing tells the
  user "this seed will fan out into ~N acceptance criteria (≈ X tokens), and your
  backend may not have the quota to finish" *before* committing — the exact setup
  that let a run burn its quota and deliver nothing (`usage-issues.md` §1.1).
  Setup advertises "85% savings" ([`skills/setup/SKILL.md`](skills/setup/SKILL.md) Step 0) but the user
  can't see the projected cost or whether the run can complete.
- **E-gap-2 — Spend isn't attributed.** Token/cost trackers exist in the TUI but
  there's no in-session, per-stage attribution (interview vs execute vs
  consensus), so the user can't tell where tokens go.
- **E-gap-3 — Frugality controls are unsurfaced.** PAL routing,
  `max_parallel_workers`, `api_max_retries`, and consensus toggles all affect
  cost but are config-only. (These same knobs amplified a real token stampede —
  `usage-issues.md` §1.1.)

### Stage F — Automation & recovery (minimize intervention)
- **F-gap-1 — `auto` isn't the obvious front door.** The recommended first build
  is often the manual `interview → run` path; the more automated `ooo auto`
  pipeline is presented as an alternative rather than the default for newcomers.
- **F-gap-2 — Resume/recovery state is opaque.** When a run is blocked, paused,
  or orphaned, the user has no clear, one-touch way to see in-flight sessions and
  resume. Cross-ref: `usage-issues.md` §3.5; `ooo resume-session` exists but is
  CLI-only.
- **F-gap-3 — Restart/reconnect friction interrupts flow.** Config fixes that
  require a reconnect leave the user stuck with no prompt or offer to do it. (Same
  root as B-gap-2.)

---

## 5. Proposed enhancement themes

Each theme lists **what**, the **value to the user**, and an **Effort / Risk**
read. Tags: **[Baby step]** = do-now, low-risk, mostly surfacing existing
behavior; **[Bigger bet]** = larger or higher-risk, deferred.

### Stage A — Install & Setup

**A1. "Path-to-success" map, shown up front and re-shown on demand** — **[Baby step]** · Effort: S · Risk: Low
- A one-screen overview of the entire journey (install → setup → interview →
  seed → run → evaluate → evolve), with a one-line "what you get" and rough
  time/effort per step, surfaced at the start of setup and available via a
  command (e.g. `ooo help --journey` or `ooo`).
- **Value:** the user knows the whole arc before starting; directly answers
  "what should I expect on the path to success?" Addresses A-gap-1, the #1 item
  in the UX-friction note.

**A2. Honest "setup complete" with a real end-to-end smoke check** — **[Baby step]** · Effort: S–M · Risk: Low
- Before declaring success, verify the *chain* works (MCP reachable, runtime/LLM
  answers a trivial probe) and state plainly whether a restart/reconnect is
  needed and *why*.
- **Value:** fewer "it said done but nothing works" dead-ends; sets correct
  expectations. Addresses A-gap-2. (Pairs with reliability fixes in the RCA.)

### Stage B — Configuration transparency (the explicit ask)

**B1. `ooo config` — one effective-configuration review surface** — **[Baby step]** · Effort: S–M · Risk: Low
- An in-session, read-only view that merges Ouroboros config + runtime config +
  env overrides into one table: **active value · default · what it controls ·
  source**. No behavior change — pure surfacing.
- **Value:** demystifies the black box at the config level; the user can finally
  see what's actually in effect. Addresses B-gap-1.

**B2. Proactive change surfacing with reconnect awareness** — **[Baby step → Bigger bet]** · Effort: M · Risk: Med
- Whenever setup or config changes (Ouroboros *or* runtime), show a diff: what
  changed, new effective values, relevant defaults, and **a clear flag per key:
  "applies live" vs "needs reconnect/restart."** Offer to perform the reconnect.
- **Value:** kills the "fixed it twice, still broken" loop — the single most
  corrosive friction for newcomers. Directly satisfies the user's stated ask.
  Depends on a key-classification that the RCA shows is structurally knowable
  ([`rca-and-remediation-plan.md`](usage-driven-feedback/rca-and-remediation-plan.md) §R2/R5, §P2). Addresses B-gap-2, F-gap-3.

**B3. Surface decision-critical defaults at the moment they matter** — **[Baby step]** · Effort: S · Risk: Low
- At setup and before a run, present the handful of high-impact defaults (model
  tier, parallel workers, retries, consensus) for review with one-line
  trade-offs, instead of leaving them in YAML.
- **Value:** informed consent on the knobs that cost money and time. Addresses
  B-gap-3; feeds E3.

### Stage C — Journey awareness

**C1. Persistent workflow breadcrumb across all `ooo` commands** — **[Baby step]** · Effort: S · Risk: Low
- A consistent footer every command prints, derived from **live session state** —
  `◆ current state → next: recommended action` (e.g. `◆ Seed ready → next: ooo run`).
  Generalize the existing `📍` next-step pattern from `ooo status`
  ([`skills/status/SKILL.md`](skills/status/SKILL.md)) into a shared convention used everywhere.
- **Owner reshape (2026-06-11):** *not* a "Step N of M" counter. Ouroboros is an
  evolutionary loop, not a pipeline, so a linear counter would lie the moment you
  re-enter the loop (a second generation, a re-interview after drift). The state→action
  form gives the same "you are here" without the lie. Specified in RFC [#1392](https://github.com/Q00/ouroboros/pull/1392).
- **Value:** solves the UX-friction note's core complaints (progress, next-step,
  invisible loop) in one stroke. **Highest leverage / lowest risk in this doc.**
  Addresses C-gap-1.

**C2. "Why we're asking / what this unlocks" framing** — **[Baby step]** · Effort: S · Risk: Low
- Prefix interview questions and gates with a one-line rationale.
- **Value:** non-technical users distinguish meaningful questions from ceremony;
  fewer abandoned interviews. Addresses C-gap-2.

### Stage D — During-run visibility

**D1. In-session live progress digest (no polling required)** — **[Baby step → Bigger bet]** · Effort: M · Risk: Med
- For `ooo run`/`ooo auto`, have the agent surface a compact, periodic digest in
  chat — phase, ACs done/total, current activity, cost-so-far, drift — driven by
  the events the TUI already consumes (`ACUpdated`, `CostUpdated`, `PhaseChanged`,
  etc., per [`docs/guides/tui-usage.md`](docs/guides/tui-usage.md) "Architecture Notes").
- **Value:** the run becomes a glass box *inside the session*, even for users who
  never open the TUI. Addresses D-gap-2.

**D2. Elevate the existing TUI: bundle, auto-offer, `ouroboros tui open`** — **[Baby step]** · Effort: S–M · Risk: Low
- Reduce the friction on an asset that already exists: include `[tui]` by
  default and auto-offer (or auto-launch) the dashboard at run start via
  **`ouroboros tui open`** — a terminal-aware launcher (`$TERM_PROGRAM` detection:
  Ghostty / iTerm2 / Apple Terminal) that spawns the monitor in a new window, with a
  copyable-command fallback for headless/SSH.
- **Owner reshape (2026-06-11):** the TUI is a **fully decoupled, runtime-agnostic
  observer** — it *polls the shared event store* rather than "attaching" to a run, so
  any runtime's run (Claude Code, Hermes, …) is equally visible. Surfacing it is a
  cross-runtime transparency win. Specified in RFC [#1392](https://github.com/Q00/ouroboros/pull/1392).
- **Value:** the rich dashboard actually gets discovered and used. Addresses
  D-gap-1. (This is the chosen near-term "visibility" move — not a new surface.)

**D3. (Deferred) Web dashboard, Hermes-style** — **[Bigger bet]** · Effort: L · Risk: Med-High
- Browser-based, always-on, shareable view of state and state changes. Two
  features make Hermes's dashboard valuable and should anchor the eventual spec:
  (1) **easy visibility into all configurations and settings** in one place (the
  GUI form of B1/B2), and (2) **few-click parity with everything the TUI can
  do** — pause/resume, switch sessions, inspect ACs — without keyboard shortcuts
  in a separate terminal.
- **Value:** best discoverability for non-technical users and remote/mobile
  visibility. Crucially, the baby-steps precursors (**B1/B2** config surfacing,
  **D2** TUI elevation) deliver most of this value *now*; the web dashboard later
  *unifies* them into a point-and-click surface. **Explicitly deferred** until
  B1/B2 and D1/D2 prove the model.

### Stage E — Frugality (waste-only, goal-subordinate)

> **Frugality is subordinate to the goal.** It minimizes only the tokens that
> **do not advance a verified acceptance criterion** — never the comprehensiveness,
> assurance, or completeness the user is paying for. Two hard invariants: (1)
> never reduce the achieved outcome; (2) never increase rework risk. The
> *destination* (working product, ACs met) is fixed; frugality lives entirely in
> the **path** — eliminating waste and protecting completion — never in the
> destination.

**Waste, not "do less."** The same verified outcome can cost an order of
magnitude more or less depending on avoidable motion: thrashing / regenerated
output dirs (`usage-issues.md` §3.3), redundant retries against a failing endpoint
(§1.1), re-reading context already in hand, or an over-powered model on a trivial
sub-AC. Removing these costs **zero** comprehensiveness. (And because "it's clear
in my head" is usually only partly true, a *sharper* interview/seed — not a
shorter one — is the highest-leverage saving: it prevents the priciest waste,
rework.)

**Layered ownership** (the methodology / execution split):

| Layer | Frugality responsibility |
|---|---|
| **Ouroboros** (methodology) | *How much* work and at *what fidelity* it commissions, plus **spec crispness**. Owns the user-held assurance dial and completion-feasibility. |
| **Agent runtime + LLM backend** | *How cheaply* a commissioned unit executes. Ouroboros can only **advise** (guardrail handed down in the spec) or **route** (pick a cheaper backend/model). |

Because waste is often indistinguishable from genuine exploration *in advance*,
the mechanism is **retrospective and conservative** — codify a guardrail only
once the waste is clearly non-advancing. (That is also why a prospective budget
cap is the wrong tool.)

> **Owner reframe (2026-06-11): these four themes are one control loop, not four
> features** — the way TCP regulates a network whose capacity it cannot observe.
> **E2 spend attribution = the sensor** (everything reads from it → ships first);
> **E4 feasibility = the controller's *initial estimate*** (not a one-shot gate);
> **429/limit handling = the controller's *normal operation*** (shrink, pause,
> resume — never lose completed work); **E3 assurance dial = the *policy input*** the
> user holds; **E1 reflective loop = the *learning layer*** that improves the estimate
> across sessions. The owner also confirmed **fan-out discipline is core's job** and
> has already shipped it (`orchestrator/backend_limits.py` + [#1372](https://github.com/Q00/ouroboros/pull/1372)
> serialize delivery), so the §1.1 stampede can't recur. Authoritative plan:
> [`investment-plans/04`](investment-plans/04-frugality-and-spend-decision.md).

**E1. Reflective frugality guardrail loop** — **[Baby step → Bigger bet]** · Effort: M · Risk: Med
- After each unit (AC / phase / generation / session), run a short, conservative
  efficiency retrospective. Where spend was clearly non-advancing, emit a
  *generalizable* guardrail into a frugality policy set, each tagged by owner:
  - **Methodology-level** (Ouroboros acts): prune assurance on low-risk ACs, cap
    decomposition depth, fewer generations, tighten the spec it hands down, and
    commit **lower reasoning effort** to trivial work (the owner's reshaped actuator —
    the reasoning-effort dial, *not* model switching; see [`investment-plans/04`](investment-plans/04-frugality-and-spend-decision.md)).
  - **Execution-level** (agent/runtime owns): Ouroboros can only **advise** (pass
    the guardrail down in the spec/prompt) or **route** to a cheaper backend.
- **Invariant:** a guardrail may only remove motion that didn't advance a verified
  AC — never reduce the outcome or raise rework risk.
- Leverages the existing
  [`src/ouroboros/observability/retrospective.py`](src/ouroboros/observability/retrospective.py) and the evolve loop, so a
  first version (reflect + propose one *advisory* guardrail per session) is a baby
  step; auto-enforcing high-confidence guardrails is the bigger bet.
- **Value:** frugality *compounds* — each project makes the next cheaper at equal
  quality, with no per-project budgeting from the user. Primary frugality
  mechanism; E2–E4 support it.

**E2. Per-stage spend attribution in the progress digest** — **[Baby step]** · Effort: S–M · Risk: Low
- Fold cost-so-far and a per-stage breakdown (interview / execute / consensus)
  into D1 and the TUI (which already tracks cost). This is the evidence base the
  E1 reflection consumes to tell waste from progress.
- **Value:** the user sees *where* tokens go and can act on it. Addresses E-gap-2.

**E3. User-held assurance dial ("Economy ↔ Thorough")** — **[Baby step]** · Effort: S · Risk: Low
- The *one* place frugality legitimately trades against cost is discretionary
  assurance depth (consensus on every AC vs only risky ones; 1 generation vs 3).
  Surface it as a single explained dial the **user** sets — never an automatic cut
  — collapsing the knobs (reasoning effort, parallelism, retries, consensus) that
  today live in YAML. Builds on B3; codified E1 guardrails may *propose* default
  positions but never override the user's choice.
- **Value:** the assurance/cost tradeoff becomes explicit and user-owned, not a
  hidden default. Addresses E-gap-3. *(Note: the legacy `PALRouter`/tier machinery
  this once surfaced is being **removed** in favor of the effort dial — owner, #1384.)*

**E4. Completion-feasibility pre-flight** — **[Baby step → Bigger bet]** · Effort: M · Risk: Med
- Before fan-out, estimate the run ("≈ N acceptance criteria, ≈ X tokens") and
  check it against the backend's available limits. If the run likely *can't finish*,
  pause and advise (resume after the window resets, shed parallelism, or route to a
  backend with headroom) so the run **completes all ACs** instead of dying mid-way.
  The opposite of a budget guillotine — its job is to protect completion, not truncate
  spend. Cross-ref the quota-probe the RCA wants the system to own ([`rca-and-remediation-plan.md`](usage-driven-feedback/rca-and-remediation-plan.md) §P1, §1.1).
- **Owner correction (2026-06-11):** check **concurrency headroom, not just token
  quota**. The §1.1 incident was a *concurrency-cap* rejection (GLM's tier-bound
  limit — Z.AI defines 429 as "interface request concurrency exceeded"), which a
  quota-only pre-flight would miss entirely; and most OpenAI-compatible backends don't
  expose remaining quota at all. The pre-flight is the controller's *initial estimate*,
  not a one-shot gate. See [`investment-plans/01`](investment-plans/01-runtime-provider-reliability.md).
- **Value:** prevents the catastrophic "burned the quota, delivered nothing"
  failure (the real §1.1 stampede). Addresses E-gap-1.

### Stage F — Automation & recovery

**F1. Make `ooo auto` the default front door, self-reporting** — **[Baby step → Bigger bet]** · Effort: M · Risk: Med
- Default the newcomer path (getting-started, welcome's "Start a project") to
  `ooo auto`, with D1 progress reporting and auto-resume on transient blocks
  (e.g. quota waits) using existing pause/resume + the quota-probe the RCA
  recommends ([`rca-and-remediation-plan.md`](usage-driven-feedback/rca-and-remediation-plan.md) §P1).
- **Value:** fewer manual touchpoints end-to-end; the pipeline babysits itself.
  Addresses F-gap-1.

**F2. One-touch "resume / what's in flight"** — **[Baby step]** · Effort: S–M · Risk: Low
- Surface in-flight/paused/blocked sessions and a single resume affordance
  in-session (promote `ooo resume-session` out of CLI-only).
- **Value:** dead-ends become recoverable without spelunking. Addresses F-gap-2;
  cross-ref `usage-issues.md` §3.5.

---

## 6. Suggested priority order

Ordered by **user journey** (the chosen basis), with the **baby-steps-first**
lens applied *within* each stage and all bigger bets deferred to the end.

| # | Theme | Stage | Tag | Effort | Why here |
|---|---|---|---|---|---|
| 1 | **A1** Path-to-success map | Install | Baby step | S | First thing a user needs; trivial to add |
| 2 | **A2** Honest setup + smoke check | Install | Baby step | S–M | Stops day-one dead-ends |
| 3 | **B1** `ooo config` effective view | Configure | Baby step | S–M | Foundation for all config transparency |
| 4 | **B2** Proactive change surfacing + reconnect flag | Configure | Baby→bigger | M | The user's explicit ask; kills the worst loop |
| 5 | **B3** Surface decision-critical defaults | Configure | Baby step | S | Informed consent on costly knobs |
| 6 | **C1** Persistent breadcrumb | Use | Baby step | S | **Highest leverage / lowest risk overall** |
| 7 | **C2** "Why this / what it unlocks" | Use | Baby step | S | Keeps non-technical users oriented |
| 8 | **D2** Elevate the existing TUI | During-run | Baby step | S–M | Makes an existing asset discoverable |
| 9 | **D1** In-session progress digest | During-run | Baby→bigger | M | Glass box without leaving the session |
| 10 | **E1** Reflective frugality guardrail loop | Frugality | Baby→bigger | M | Frugality north star — turns waste into reusable guardrails |
| 11 | **E2** Per-stage spend attribution | Frugality | Baby step | S–M | Cheap, rides on D1/TUI; feeds E1's reflection |
| 12 | **E3** User-held assurance dial | Frugality | Baby step | S | The one legitimate cost/assurance tradeoff — user-owned |
| 13 | **E4** Completion-feasibility pre-flight | Frugality | Baby→bigger | M | Protects completion; prevents "burned quota, delivered nothing" |
| 14 | **F2** One-touch resume | Automation | Baby step | S–M | Recoverable dead-ends |
| 15 | **F1** `auto` as default, self-reporting | Automation | Baby→bigger | M | End-to-end intervention drop |
| — | **D3** Web dashboard (Hermes-style) | During-run | **Deferred bigger bet** | L | Unifies B1/B2 + TUI parity; revisit after they prove out |

### If you only do three things (the baby-steps starting line)
1. **C1 — persistent breadcrumb.** Smallest change, biggest dent in the
   "I'm lost" problem.
2. **B1 + B2 — effective-config view and proactive change surfacing.** Directly
   answers the stated ask and ends the "fixed-but-still-broken" loop.
3. **D2 — elevate the TUI.** Turns an already-built dashboard from hidden to
   default.

These three are all low-risk, mostly surface existing capability, and together
move the needle most on the black-box feeling.

---

## 7. Open questions & assumptions

**Resolved (incorporated above)**
- *Hermes dashboard* → D3 anchored on its two valued features: all-config
  visibility and few-click TUI parity.
- *Token budget* → no fixed ceiling and no truncating cap; frugality is
  **waste-only and goal-subordinate**, driven by the reflective guardrail loop
  (E1), with E4 protecting *completion* (not capping spend).
- *Guardrail enforcement* → **advisory-first, per-session, conservative**;
  high-confidence guardrails may graduate to auto-enforcement later, and may only
  ever remove non-advancing spend (the E1 invariant).
- *`[tui]` by default* → **yes** (D2). *`ooo config`* → **yes**, top-level (B1).

**Still open**
- **Auto-enforcement bar.** What confidence/evidence threshold lets a guardrail
  graduate from advisory to a hard gate?
- **Persona confirmation.** Proposals center the non-technical first-timer; if
  power users are co-primary, D3 and a scripting/headless surface rise in priority.
- **Document home.** Drafted at repo root as requested; may fit better beside the
  related planning docs in [`usage-driven-feedback/`](usage-driven-feedback/) — easy to relocate.

---

## 8. Evidence & references

- **UX-friction (primary driver):** internal memory note — non-technical
  first-timers get lost; no journey visibility, no progress orientation, unclear
  next command, invisible loop.
- **Reliability companion docs (cross-referenced, not duplicated):**
  [`usage-driven-feedback/usage-issues.md`](usage-driven-feedback/usage-issues.md),
  [`usage-driven-feedback/rca-and-remediation-plan.md`](usage-driven-feedback/rca-and-remediation-plan.md).
- **Journey & onboarding:** [`docs/getting-started.md`](docs/getting-started.md),
  [`skills/welcome/SKILL.md`](skills/welcome/SKILL.md), [`skills/setup/SKILL.md`](skills/setup/SKILL.md),
  [`skills/help/SKILL.md`](skills/help/SKILL.md).
- **Status & dashboard:** [`skills/status/SKILL.md`](skills/status/SKILL.md),
  [`docs/guides/tui-usage.md`](docs/guides/tui-usage.md), [`src/ouroboros/tui/`](src/ouroboros/tui/).
- **Automation:** [`skills/auto/SKILL.md`](skills/auto/SKILL.md).
- **Configuration:** [`docs/config-reference.md`](docs/config-reference.md), `~/.ouroboros/config.yaml`,
  runtime config (e.g. `~/.hermes/config.yaml`).
