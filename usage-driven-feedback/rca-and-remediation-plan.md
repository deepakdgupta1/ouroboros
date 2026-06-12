# RCA — June 5–7, 2026 Run Failures

> Root-cause analysis of the failures logged in
> [`usage-issues.md`](./usage-issues.md), verified against source.
> Stack under test: Claude Code plugin → `hermes` runtime → Z.AI/GLM via litellm.
> Every claim below cites `file:line` and was confirmed by reading the code,
> not inferred from the transcripts.

## TL;DR

The single most disruptive failure (1.1 — a 429 that killed all 14 acceptance
criteria with zero tokens consumed) was **not** caused by a missing safety
mechanism. The pause-on-quota mechanism existed and was live. It failed because
**typed error information was destroyed at the hermes subprocess boundary**, and
the orchestrator's quota classifier requires structured fields that the hermes
error path never emits.

This same disease — *reconstructing meaning heuristically at a boundary where a
typed contract was needed* — explains the rest of the LLM-backend cluster. The
relevant mechanisms mostly **exist**; they are keyed to the wrong signal, scoped
to the wrong provider, or invoked at the wrong time.

---

## Evidence table

| # | Root cause | Verdict | Primary evidence |
|---|---|---|---|
| R1 | Leaky LLM abstraction; no parameter-level capability negotiation | Confirmed | `providers/base.py:110`, `providers/litellm_adapter.py:284`, `:278` |
| R2 | No single source of truth; structural config bound at construction | Confirmed (refined) | `orchestrator/adapter.py:912`; live reads via `get_usage_limit_pause_seconds` |
| R3 | Retry/rate governance scoped to the wrong provider and window | Refined (partially refuted) | `orchestrator/rate_limit.py`, `orchestrator/adapter.py:956` |
| R4 | Quota-pause safety keyed to fragile signals; lossy error boundary | Confirmed (strongest) | `orchestrator/hermes_runtime.py:521`, `orchestrator/runner.py:1130`, `:1074` |
| R5 | One execution/permission model for two runtimes | Confirmed | `orchestrator/adapter.py:912` (bound at init) |
| S | State coherence: zombie jobs survive restarts | Refined | `orchestrator/heartbeat.py`, `mcp/job_manager.py:1046`, `:1107` |

---

## R4 — The whole-run killer (issue 1.1), traced end to end

The pause-on-usage-limit logic has existed since 2026-05-02 (`git log -S` on
`_message_has_runtime_error_shape`), so it was live during the incident. It
still did not fire. Here is why, step by step.

```
Z.AI returns HTTP 429 (typed: includes a reset timestamp)
  → hermes subprocess exits non-zero
  → orchestrator/hermes_runtime.py:521 collapses it to:
        data = {"subtype": "error", "exit_code": returncode}
        content = "Hermes execution failed:\n...Usage limit reached...reset at..."
      ← the typed 429 + reset time is now only in free-form `content`
  → orchestrator/runner.py:1212 classifies the failure
  → _is_usage_limit_text (runner.py:1130):
        if not has_runtime_error_shape: return False   ← short-circuits here
  → _metadata_has_runtime_error_shape (runner.py:1074) requires one of:
        error_type, error_code, code, status, status_code, http_status,
        provider, recoverable, is_retriable, retry_after, reset_at, ...
      ← `subtype` and `exit_code` are NOT in this set
  → shape check fails → quota regex never runs → no pause
  → all 14 ACs hard-fail
```

Two structural facts make this a contract bug, not a tuning bug:

1. **The information existed and was thrown away.** The 429 carried a reset
   time. `hermes_runtime.py:521` discards the HTTP status and reason, keeping
   only a Unix `exit_code`.
2. **The producer and consumer disagree on the schema.** The classifier is
   correct to demand structured fields; hermes simply never provides them on
   this path. Note the inconsistency *within hermes itself*: the timeout path
   at `hermes_runtime.py:513` **does** set `error_type="TimeoutError"`, so
   timeouts are recognized — only the generic non-zero-exit path (where 429s
   land) is opaque.

**Fix:** define a typed runtime-error contract (below) and have every runtime
adapter populate it. This single change closes 1.1 and makes 1.2-class failures
classifiable too.

---

## R1 — Leaky provider abstraction (issues 1.2, 1.3, 1.5)

- `providers/base.py:110` — the `LLMAdapter` protocol exposes only `complete()`.
  There is **no capability query**. `response_format` is just a field on
  `CompletionConfig` (`base.py:69`).
- `providers/litellm_adapter.py:284` — `response_format` is forwarded to
  litellm **unconditionally**, with no `litellm.drop_params=True` and no
  per-provider filtering. When Z.AI/GLM rejects it, litellm raises
  `UnsupportedParamsError` and the call hard-fails (issue 1.2).
- `providers/litellm_adapter.py:278` — the *only* provider-specific handling is
  an inline string check (`"anthropic" in model_lower`) for the top_p quirk.
  Provider differences are scattered as ad-hoc conditionals rather than declared
  in a capability table.

A coarse backend-level capability model exists
(`config/loader.py:1197`, `get_backend_capability().supports_llm`), but it does
not describe **parameter-level** support, which is where 1.2 bit.

---

## R3 — Rate governance exists, but for the wrong provider and window

This corrects the original claim of "no global budget." A real shared governor
exists: `SharedRateLimitBucket` (`orchestrator/rate_limit.py`), a sliding-window
RPM/TPM budget shared across concurrent workers. Its limitations are what
matter:

- **Provider-blind.** `orchestrator/adapter.py:956` `_build_rate_limit_bucket`
  is hardcoded to Anthropic ceilings (`OUROBOROS_ANTHROPIC_RPM_CEILING` = 40,
  `OUROBOROS_ANTHROPIC_TPM_CEILING` = 32_000). On a hermes→Z.AI run it applies
  Anthropic's numbers, which bear no relation to Z.AI's actual limits. The
  docstring even reads "the shared **Anthropic** rate-limit bucket."
- **Wrong window.** `RATE_LIMIT_WINDOW_SECONDS = 60` (`rate_limit.py:11`). It is
  a 60-second burst smoother. The actual failure was a **5-hour rolling quota**;
  a per-minute limiter cannot model or prevent that.
- **Process-local and amnesiac.** The bucket is in-memory per adapter instance
  (`rate_limit.py:49`). It has zero knowledge of quota already consumed by prior
  runs, so a fresh run starts believing it has full budget and fires all 14 ACs
  into an already-exhausted upstream quota — matching the observed "instant,
  zero-token" failures.

The user's own mitigation (`scripts/ooo_preflight.sh`) is the only component in
the loop that actually inspects live quota state before fan-out — a
responsibility the system should own.

---

## R2 / R5 — Config bound at construction; inconsistent reload

- `orchestrator/adapter.py:912` — `permission_mode` is returned from
  `self._permission_mode`, captured at adapter construction (MCP-connect time).
  Editing `~/.ouroboros/config.yaml` does nothing to a live adapter until it is
  reconstructed, i.e. an `/mcp` reconnect (issues 2.4, 2.5).
- Yet some values **are** read live from disk (`get_usage_limit_pause_seconds`,
  `config/loader.py:760`). The result is the worst case for a user's mental
  model: *some* edits apply immediately, *some* require a reconnect, with no
  signal which is which. This precisely produces the "I already fixed it twice
  and it's still broken" loop logged in 5/2.5.

This is one defect viewed twice: structural runtime state binds config at object
construction and is never invalidated.

---

## S — Zombie jobs (issue 3.1): liveness exists but isn't authoritative

This corrects the original "no liveness contract" claim. A file-based PID-lock
heartbeat exists (`orchestrator/heartbeat.py`): a runner writes
`{pid}:{boot_time}` to `~/.ouroboros/locks/{session_id}`, and `is_holder_alive`
verifies both PID existence and boot time (guarding against PID recycling).

The defect is **when** it runs:

- `is_holder_alive` is consulted **lazily**, only on specific read paths
  (`mcp/job_manager.py:1046`, `:1107`).
- There is **no startup/reconnect reconciliation sweep** that enumerates jobs in
  `running` state, checks their locks, and transitions lock-orphaned jobs to a
  terminal/recoverable status.

So when the MCP server restarted mid-build (~09:44Z), the workers died but the
DB row stayed `running` indefinitely, requiring manual cancellation. The control
plane (persisted job status) and data plane (actual worker liveness) are
reconciled opportunistically, not authoritatively.

---

## Unifying thesis

Across R1, R3, R4, and S, the mechanisms largely exist. They fail because the
system **reconstructs meaning heuristically at boundaries where a typed contract
was required**:

- a regex over free-form text instead of a typed `error_kind` (R4),
- Anthropic-shaped assumptions instead of the connected provider's declared
  limits (R3),
- blind parameter forwarding instead of a capability query (R1),
- lazy liveness reads instead of authoritative reconciliation (S).

Fix the contracts and the symptoms collapse together.

---

## Prioritized remediation plan

### P0 — Typed runtime-error contract (closes 1.1, makes 1.2 classifiable)

Define a structured error shape every runtime adapter must populate on failure,
e.g. a frozen dataclass surfaced in `AgentMessage.data`:

```
error_type: str            # "RateLimitError", "UnsupportedParam", ...
http_status: int | None    # 429, 400, ...
provider: str | None       # "zai", "anthropic", ...
retry_after | reset_at     # carried through, never dropped
recoverable: bool
```

- Change `orchestrator/hermes_runtime.py:521` (and the other runtime adapters'
  non-zero-exit paths) to parse the subprocess failure and emit these fields,
  rather than collapsing to `{"subtype", "exit_code"}`.
- The existing classifier (`runner.py:1074`/`:1130`) already keys on exactly
  these fields, so populating them at the source is sufficient — keep the regex
  only as a last-resort fallback.
- Add a regression test: a hermes 429 with a reset timestamp must produce a
  `usage_limit` pause with the correct `resume_after`.

### P1 — Parameter-level capability negotiation (closes 1.2, 1.3)

- Add a capability descriptor per provider (does it accept `response_format`,
  which model namespaces it owns, etc.).
- In `litellm_adapter.py:_build_completion_kwargs`, drop or translate
  unsupported params (or set `litellm.drop_params=True`) based on the
  descriptor instead of forwarding blindly (`:284`).
- Replace the inline `"anthropic" in model_lower` checks (`:278`) with lookups
  against the same descriptor.

### P1 — Provider-aware quota tracking (fixes the bucket; mitigates 1.1 recurrence)

- Generalize `_build_rate_limit_bucket` (`adapter.py:956`) beyond Anthropic:
  source RPM/TPM from the connected backend's profile, not a hardcoded constant.
- Add a **long-horizon quota probe** (the responsibility currently carried by
  `scripts/ooo_preflight.sh`) that runs before Deliver-phase fan-out and pauses
  rather than stampedes when the rolling window is already exhausted.

### P2 — Authoritative job reconciliation (kills zombies, 3.1)

- On MCP server startup/reconnect, sweep all `running` jobs, check each against
  its lock via `is_holder_alive`, and transition orphans to a terminal or
  resumable state. Surface the result so resume state is no longer opaque (3.5).

### P2 — Coherent config reload (kills 2.5 / "fixed but still broken")

- Either rebuild runtime-bound objects (adapter, rate bucket, worker pool) on
  config change, or make config read live consistently. Failing that, **detect**
  that a structural field changed since connect and tell the user a reconnect is
  required — never let a no-op edit pass silently.

### Already mitigated (keep or fold into the above)

`max_parallel_workers: 1`, `api_max_retries: 1`, `permission_mode:
bypassPermissions`, `--python 3.12`, and `scripts/ooo_preflight.sh`. The P1
quota work should absorb the preflight script so it is no longer a manual
convention.
