# Investment #3 — Configuration Coherence (surface it; make reload honest)

| | |
|---|---|
| **Status** | **Greenlit as the next #1376 slice (2026-06-11)** — own RFC; owner corroborated the silent-reconnect pain. See [Owner direction](#owner-direction--greenlit-as-the-next-slice-2026-06-11). |
| **Date** | 2026-06-11 |
| **Author** | deepak |
| **Owner** | TBD |
| **Effort / Risk** | Low for the detect-and-warn half (high value) / Low |
| **Persona** | Primary: non-technical first-timer (cannot know which of ~50 knobs matters) |
| **Addresses** | [discussion #1376](https://github.com/Q00/ouroboros/discussions/1376) (Install/setup + Configure rows) |
| **Related** | [#02 Journey](./02-journey-transparency.md) · [Architecture Value Map §11.3](../ouroboros-architecture-value-map.md) · [RCA R2/R5](../usage-driven-feedback/rca-and-remediation-plan.md) · [Usage issues §2.4–2.6, §5](../usage-driven-feedback/usage-issues.md) · [improvement-themes §1](../improvement-themes.md) |

## TL;DR

Kill the *"I fixed it but it's still broken"* loop, and stop hiding the config
surface from users. Two halves: (a) when a structural config field changes but a
live adapter still holds the old value, **detect it and tell the user a reconnect
is required** — never let a no-op edit pass silently; (b) **proactively surface
the relevant configs and their defaults** at setup and on change, for Ouroboros
*and* the underlying runtime.

## Owner direction — greenlit as the next slice (2026-06-11)

Per the owner ([Q00](https://github.com/Q00)) on [#1376](https://github.com/Q00/ouroboros/discussions/1376),
the usability theme is **split slice-by-slice**: the breadcrumb + TUI surfacing landed
first as RFC [#1392](https://github.com/Q00/ouroboros/pull/1392), and **`ooo config`
/ change-surfacing is the explicitly named *next* slice, with its own RFC** — it "has
its own design questions (config source unification, live-vs-reconnect detection) and
deserves its own RFC rather than riding along."

The owner independently corroborated the core pain: *"we've been bitten by the
silent-MCP-reconnect problem ourselves, so that report rings true."* The
detect-and-warn P0 and the effective-config view (P1) carry forward unchanged; **this
plan is the source for that next RFC.**

## Discussion alignment (#1376)

[Discussion #1376](https://github.com/Q00/ouroboros/discussions/1376) names
**configuration opacity** as one of its three pains and proposes two concrete
surfaces this plan now adopts as acceptance targets:

- **`ooo config` — one in-session view of the *effective* configuration**, each
  key shown as **active value · default · what it controls · source** (Ouroboros
  vs. runtime vs. env), unified across all three. Folded into **P1**.
- **Proactive change surfacing** — when config changes, show a **diff**, the new
  effective values, and a per-key **"applies live vs. needs reconnect"** flag.
  This is exactly the P0 classification (`live | reconnect`) rendered as a
  user-facing column — the two halves of this plan meet here.
- **Honest "setup complete"** — `setup` should **smoke-check the chain** (MCP +
  runtime/LLM reachable) and state plainly when a reconnect is needed and why.
  Folded into **P1** (and complements the install flow map in [#02](./02-journey-transparency.md)).

## Problem (user-visible pain)

The most corrosive friction in the field logs was not a crash — it was a user
editing config, seeing no effect, and concluding the tool was broken:

> *"I have already done that twice and still this error is not going away."*
> — usage issues §5 / §2.5

Root cause (RCA R2/R5): **some config values are read live from disk, others are
bound when the adapter is constructed (at MCP-connect time)** — with no signal to
the user which is which. So `permission_mode` edits silently require an `/mcp`
reconnect, while `get_usage_limit_pause_seconds` reads live. The user's mental
model is destroyed: identical-looking edits behave differently.

Layered on top is **configuration opacity** (improvement-themes §1): Ouroboros
never proactively shows the user the relevant set of knobs and their defaults —
in Ouroboros itself, in the runtime, or both — so first-timers can't even find
the lever they need (e.g. the `permission_mode: acceptEdits → bypassPermissions`
fix for headless runs, usage §2.4).

## Current state (what exists)

- `orchestrator/adapter.py:883` captures `self._permission_mode` at construction;
  `:913` exposes it as a read-only property. **Construction-bound.** Editing
  `~/.ouroboros/config.yaml` does nothing to a live adapter until reconstruction
  (an `/mcp` reconnect).
- `config/loader.py` reads *some* values live from disk
  (`get_usage_limit_pause_seconds`, `:760`) — proving the inconsistency is real.
- `config/models.py` + `loader.py` are the single source of truth for the 13
  sections and ~50 env overrides, fully documented in
  [`docs/config-reference.md`](../docs/config-reference.md).
- `ouroboros config init` generates defaults; `cli/mcp_doctor.py` exists for MCP
  diagnostics — a natural home for a config-coherence check.

No mechanism today **compares on-disk config against the values a live adapter is
actually using**, and nothing **surfaces the relevant subset of config at the
moment it becomes relevant** (setup, runtime switch, permission change).

## Target state (definition of done)

- Changing a structural, construction-bound field and then acting **never passes
  silently**: the user is told "this change needs a reconnect to take effect"
  (ideally naming the field).
- After `setup` / runtime switch, the user sees the **relevant config keys, their
  current values, and their defaults** — Ouroboros *and* runtime — without
  reading a 900-line reference.
- A single command (`ouroboros config doctor` or folded into `mcp-doctor`)
  reports drift between disk and the live session.

## Strategy

Sequence by value-per-effort: the **detect-and-warn** half is cheap and removes
the entire confusion loop, so it goes first. Full live-reload is more invasive
and is deferred. Principles:

1. **Make staleness loud, not silent.** A no-op edit is a bug in the *feedback*,
   not the user. Surface it.
2. **Surface config at the moment of relevance,** not as a wall of reference.
   Show only the subset tied to what just changed (runtime, permissions, models).
3. **One source of truth.** Reuse `config/models.py` metadata; do not hand-maintain
   a second list of "which fields are construction-bound."

## Plan (phased, baby-steps)

### P0 — Detect stale structural edits and warn (the whole confusion loop)
- **Tasks:** Tag each config field with a `reload: live | reconnect` classification
  (a property on the Pydantic models in `config/models.py`). On MCP
  reconnect/tool entry, diff the on-disk config against the values bound in the
  live adapter; if any `reconnect`-class field changed, return a clear warning
  ("`orchestrator.permission_mode` changed on disk (acceptEdits → bypassPermissions)
  but the live session still uses acceptEdits — reconnect with `/mcp` to apply").
- **Touch-points:** `config/models.py`, `config/loader.py`, `orchestrator/adapter.py`,
  `mcp/` connect path, `cli/mcp_doctor.py`.
- **Acceptance:** editing `permission_mode` without reconnecting produces a
  visible, specific warning the next time a tool runs; live-class edits produce
  no warning.
- **Effort:** ~2–3 days. **Ships alone and resolves usage §2.5 directly.**

### P1 — Effective-config view + change surfacing + setup smoke-check
- **Tasks:**
  - **`ooo config` effective view (#1376):** one in-session table where every key
    shows **active value · default · what it controls · source** (Ouroboros /
    runtime / env), unifying the 13 sections + the runtime-side config
    (`~/.codex/*.config.toml`, `~/.hermes/config.yaml`) + env overrides. A
    `--relevant` filter narrows to what the active backend actually uses.
  - **Change surfacing (#1376):** on a detected config change, show a **diff**, the
    new effective values, and a per-key **"applies live vs. needs reconnect"**
    flag (rendering the P0 `live | reconnect` classification).
  - **Honest setup (#1376):** `ouroboros setup` smoke-checks the **chain** (MCP +
    runtime/LLM reachable) and states plainly when a reconnect is needed and why.
- **Touch-points:** `cli/setup.py`, `cli/config.py`, `cli/mcp_doctor.py`, runtime
  profile modules (`codex/`, `copilot/`, …), `config/models.py` (metadata for the
  "what it controls" column).
- **Acceptance:** a first-timer who runs setup sees the handful of knobs that
  matter for their runtime, with defaults and source; `ooo config` answers
  "what is the value, where did it come from, will my edit apply" in one view;
  setup never reports "complete" when the chain is actually unreachable.

### P2 — Selective coherent reload
- **Tasks:** Where safe, rebuild the construction-bound objects (adapter, rate
  bucket, worker pool) on detected change instead of requiring a manual reconnect.
  Start with the highest-friction field (`permission_mode`).
- **Acceptance:** editing a supported field applies without a manual `/mcp`
  reconnect, with an audit event recording the rebind.

## Metrics / success signals

- Elimination of the "edited config, no effect, gave up" pattern (proxy: §2.5 /
  §5 recurrence).
- % of config edits that either apply immediately or produce an explicit
  reconnect prompt (target: 100% — zero silent no-ops).
- Setup completion → first successful run conversion (config opacity is a known
  drop-off point).

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| The `live | reconnect` tag drifts from real binding behavior | Derive it from one place and add a test that fails if a `reconnect`-tagged field is actually read live (and vice-versa). |
| Live-reload (P2) introduces mid-run state inconsistency | Restrict P2 to fields safe to rebind between tasks; never rebind mid-task; record an event. |
| Surfacing runtime config touches files Ouroboros doesn't own | Read-only display by default; never auto-edit `~/.codex` / `~/.hermes` without explicit action. |

## Non-goals

- A config GUI. Terminal output + existing TUI only.
- Full hot-reload of every field in P0/P1 — detect-and-warn first, reload later.

## Open questions

- Should the warning block tool execution or just annotate it? (Recommendation:
  annotate + warn; block only when the stale field would clearly cause failure,
  e.g. a headless run still on `acceptEdits`.)
- Is `config doctor` a new subcommand or an extension of `mcp-doctor`?
