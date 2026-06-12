# Investment #1 — Runtime / Provider Reliability Contract

| | |
|---|---|
| **Status** | In flight (partial) — **owner reframed P2 (2026-06-11): feasibility is concurrency-aware, not quota-only**; the stampede is already mitigated by `backend_limits` + [#1372](https://github.com/Q00/ouroboros/pull/1372). See [Owner direction](#owner-direction--the-pre-flight-is-concurrency-aware-not-quota-only-2026-06-11). |
| **Date** | 2026-06-11 |
| **Author** | deepak |
| **Owner** | TBD |
| **Effort / Risk** | Medium / Low (mostly additive contracts + a lookup table) |
| **Persona** | All users; the failure mode is catastrophic for everyone |
| **Addresses** | [discussion #1377](https://github.com/Q00/ouroboros/discussions/1377) (completion-feasibility pre-flight theme) |
| **Related** | [#04 Frugality & the spend decision — don't under-power hard work](./04-frugality-and-spend-decision.md) · [Architecture Value Map §11.1](../ouroboros-architecture-value-map.md) · [RCA R1/R3/R4](../usage-driven-feedback/rca-and-remediation-plan.md) · [Usage issues §1](../usage-driven-feedback/usage-issues.md) |

## TL;DR

Make "the run died instantly for an opaque reason" structurally impossible.
Three linked gaps remain at the runtime/provider boundary: (1) parameters are
forwarded to providers that reject them, (2) provider quota is unknown until a
run stampedes into it, and (3) a single backend has no fallback. The unifying
fix is **typed contracts at the boundary** instead of heuristic reconstruction.

## Problem (user-visible pain)

The single most destructive logged failure: `/ouroboros:run` reached Deliver,
fanned out **all 14 acceptance criteria**, and **every one failed instantly with
zero tokens consumed** — the working tree stayed empty. Root cause: a typed HTTP
429 (carrying a quota-reset timestamp) was flattened to free-form text at the
hermes subprocess boundary, so the existing usage-limit-pause safety never fired
(RCA R4). Sibling failures in the same cluster:

- `litellm.UnsupportedParamsError` — GLM rejects `response_format`, hard-failing
  QA (usage issue §1.2, RCA R1).
- Anthropic-shaped rate ceilings applied to a Z.AI run (RCA R3).
- No fallback provider, so one 429 takes down the whole fan-out (usage §1.1).

**Why it matters:** this is the class of failure that converts "magic" into "I
wasted an afternoon and wrote no code." It erodes trust in everything downstream.

## Owner direction — the pre-flight is concurrency-aware, not quota-only (2026-06-11)

The owner ([Q00](https://github.com/Q00)) on [#1377](https://github.com/Q00/ouroboros/discussions/1377)
reframes this plan's completion-feasibility work (P2) and confirms what has landed:

- **Fan-out discipline is core's job, and core already acted.**
  `orchestrator/backend_limits.py` serializes delivery to **1 AC at a time** for any
  backend whose limits Ouroboros can't know (every CLI runtime, hermes included),
  raised only via explicit `OUROBOROS_MAX_CONCURRENCY`;
  [#1372](https://github.com/Q00/ouroboros/pull/1372) added configurable rate-budget
  pacing for non-Claude delivery. **The 14-AC stampede cannot recur in that form** —
  the catastrophic incident is already substantially mitigated; this plan hardens the
  contract around it.
- **The incident was a concurrency-cap rejection, not quota exhaustion.** Two facts:
  most OpenAI-compatible backends **don't expose remaining quota**, and **GLM enforces
  a hard concurrency limit tied to plan tier** — Z.AI's docs define rate limits *as*
  concurrency limits, and their error table defines 429 as *"interface request
  concurrency exceeded."* So **a quota-only pre-flight could not have prevented the
  incident**: 14 parallel ACs against a Lite/Pro-tier concurrency cap trip 429 even
  with quota remaining. Feasibility must reason about **concurrency headroom, not just
  token quota.**
- **The pre-flight is the controller's *initial estimate*, not a one-shot gate**
  (#1377's control-loop frame). 429/limit handling is the controller's **normal
  operation** — shrink the window, pause, resume after reset; completed work is never
  lost. The **adaptive-concurrency evolution of `backend_limits`** (static cap →
  signal-driven) is the named **second slice** of the frugality workstream
  ([#04](./04-frugality-and-spend-decision.md)).

## Current state (what has landed vs. what remains)

**Landed (do not redo):**

- `src/ouroboros/orchestrator/runtime_error.py` — `classify_subprocess_failure()`
  now *re-derives* typed fields (`error_type`, HTTP status, rate-limit phrasing)
  from free-form subprocess text, so the usage-limit classifier can run
  (commit `d842be3b`). **Caveat:** this is a regex bridge over text, applied at
  the hermes path — not a source-typed contract emitted by every adapter.
- `src/ouroboros/orchestrator/backend_limits.py` + delivery fan-out now respects
  backend concurrency limits (commit `916b92e8`) — serializing to **1 AC at a time**
  for any backend whose limits are unknown (every CLI runtime), raised only via
  `OUROBOROS_MAX_CONCURRENCY`; [#1372](https://github.com/Q00/ouroboros/pull/1372)
  added configurable rate-budget pacing for non-Claude delivery. Per the owner, **the
  14-AC stampede cannot recur in that form.**
- `backends/capabilities.py` has a **coarse** `BackendCapability`
  (`supports_runtime` / `supports_llm` / `supports_interview_driver` /
  `supports_tool_envelope`) — but nothing at **parameter** granularity.

**Remaining (this plan):**

1. `providers/base.py` `LLMAdapter` protocol still exposes only `complete()`;
   there is **no capability query**.
2. `providers/litellm_adapter.py:284` still forwards `response_format`
   **unconditionally**; `litellm.drop_params` is never set; the only
   provider-specific handling is an inline `"anthropic" in model_lower` string
   check (`:278`, `:83`).
3. The rate-limit bucket (`orchestrator/rate_limit.py`) is a 60-second window,
   Anthropic-hardcoded, process-local — it cannot model a 5-hour rolling quota
   and starts every run believing it has full budget.
4. There is **no fallback provider** path and **no long-horizon quota probe**;
   the only component that inspects live quota is the user's own
   `scripts/ooo_preflight.sh`.

## Target state (definition of done)

- A provider that cannot accept a parameter has it **dropped or translated**,
  never hard-failing the call.
- Every runtime adapter emits **source-typed** error fields on failure; the regex
  bridge is demoted to a last-resort fallback.
- A usage-limit/quota condition produces a **graceful pause with a correct
  `resume_after`**, on every backend — not just hermes, not just via text match.
- A run that points at an already-exhausted quota **pauses before fan-out**
  rather than firing N instant failures.
- A single backend's 429 can **shed to a configured fallback** instead of
  killing the run.

## Strategy

The RCA's unifying thesis is the design north star: *the mechanisms mostly exist;
they fail because meaning is reconstructed heuristically at a boundary where a
typed contract was required.* So:

1. **Declare capabilities; stop guessing.** Add a per-provider capability
   descriptor (parameter-level) and consult it before every call.
2. **Type the error at the source.** Have each adapter populate structured error
   fields directly; keep the `runtime_error.py` regex as fallback only.
3. **Make quota a first-class, provider-aware signal.** Source limits from the
   connected backend's profile; add a pre-fan-out probe that pauses, not
   stampedes.
4. **Degrade gracefully.** A declared fallback list lets the orchestrator shed
   load instead of hard-failing.

## Plan (phased, baby-steps)

### P0 — Parameter-level capability descriptor (closes usage §1.2/1.3)
- **Tasks:** Extend `BackendCapability` (or add a sibling `ProviderCapability`)
  with parameter flags (`supports_response_format`, owned model namespaces, …).
  In `litellm_adapter._build_completion_kwargs`, drop/translate unsupported
  params using the descriptor (or set `litellm.drop_params=True` per provider).
  Replace the inline `"anthropic" in model_lower` checks (`:278`, `:83`) with
  descriptor lookups.
- **Touch-points:** `backends/capabilities.py`, `providers/base.py`,
  `providers/litellm_adapter.py`.
- **Acceptance:** a GLM/Z.AI call with `response_format` set **succeeds with the
  param stripped** (regression test); Anthropic top-p quirk handled via the
  descriptor, not a string check.
- **Effort:** ~1–2 days. **Ships alone.**

### P1a — Source-typed runtime-error fields across all CLI adapters
- **Tasks:** Each CLI runtime's non-zero-exit path
  (`hermes_runtime`, `gemini_cli_runtime`, `codex_cli_runtime`,
  `opencode_runtime`, `kiro_adapter`, `pi_runtime`, `goose_runtime`) parses the
  upstream failure and emits the typed fields directly into `AgentMessage.data`,
  rather than relying on the text re-derivation. Demote
  `classify_subprocess_failure` to a documented fallback.
- **Acceptance:** every adapter has a regression test: a 429 with a reset time
  produces a `usage_limit` pause with the correct `resume_after`.

### P1b — Provider-aware quota tracking
- **Tasks:** Generalize `_build_rate_limit_bucket` (`adapter.py:956`) to source
  RPM/TPM from the connected backend's profile instead of the Anthropic
  constants. Persist consumed-quota state so a fresh run does not assume full
  budget.
- **Acceptance:** on a non-Anthropic backend, the bucket reflects that provider's
  declared limits (unit test); consumed quota survives a process restart.

### P2 — Completion-feasibility pre-flight + fallback providers
- **Framing (#1377, owner-reframed):** this is **completion-feasibility**, not a
  budget cap — the goal is that a run **completes all its ACs** rather than dying
  mid-fan-out. It is the controller's **initial estimate, not a one-shot gate**;
  429/limit handling is the controller's *normal operation* (shrink the window, pause,
  resume — never lose completed work). Crucially, feasibility must reason about
  **concurrency headroom, not just token quota**: the grounding incident was a
  *concurrency-cap* rejection (GLM's tier-bound limit), which a quota-only check would
  miss entirely.
- **Tasks:** Add a pre-Deliver-fan-out probe that checks **both** live rolling-window
  quota **and concurrency headroom against the backend's declared cap**, and **pauses
  or sheds parallelism with advice** rather than stampeding (absorbing
  `scripts/ooo_preflight.sh` so it is no longer a manual convention). Where remaining
  quota is unobservable (most OpenAI-compatible backends), fall back to the
  concurrency-serialized delivery `backend_limits` already enforces. Add a declared
  `fallback_providers` list the orchestrator sheds to on 429.
- **Acceptance:** a run that would exceed the backend's concurrency cap **sheds to
  serialized delivery** instead of firing N instant 429s; pointing a run at an
  exhausted quota yields a single graceful pause **with a recommended next action**;
  a forced 429 on the primary routes to the fallback and the run continues to completion.

## Metrics / success signals

- **Zero-token mass failures → 0** (the §1.1 signature: N ACs fail with no tokens
  and no tool calls).
- Share of provider failures that resolve to a **graceful pause or fallback**
  rather than a hard run-kill.
- `UnsupportedParamsError` count → 0 across supported providers.

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Capability descriptors drift from real provider behavior | Treat the descriptor as code with tests; fail *open* (forward the param) only when a provider is unknown, and log it. |
| Dropping `response_format` weakens structured output | Translate to prompt-level JSON instructions where the provider lacks native support (the Pi adapter already models this). |
| Persisted quota state goes stale | Store with a TTL keyed to the provider's documented window; probe verifies before trusting. |

## Non-goals

- Building a universal cross-provider cost model (that is Plan #4).
- Re-architecting LiteLLM usage; this works *within* the existing adapter.

## Open questions

- Where should the capability descriptor live — extend `BackendCapability`, or a
  new `providers/capabilities.py` keyed by provider rather than backend?
- Should `fallback_providers` be global, per-tier (`economics`), or per-role?
- Is consumed-quota persistence in the event store, or a separate small cache?
