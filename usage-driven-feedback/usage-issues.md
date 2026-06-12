# Ouroboros — Issues Log (June 5–7, 2026)

> Compiled from a sweep of all Claude Code chat sessions in the `custom_costing`
> project over the last 2 days. Source transcripts live in
> `~/.claude/projects/-Users-shilpagupta-custom-costing/*.jsonl`.
> Each issue cites the transcript file it came from.

This log captures every distinct problem, error, and point of friction we hit
while using Ouroboros to build the custom-costing tool. Issues are grouped by
theme. The single most disruptive issue was the **Z.AI/GLM 429 quota
exhaustion** that hard-failed an entire run.

---

## 1. LLM Provider / Backend Issues

These were the most frequent and most damaging class of problems. Ouroboros was
pointed at Gemini CLI early on, then at Z.AI/GLM via litellm — both stacks
produced recurring failures.

### 1.1 Z.AI/GLM 429 — quota exhaustion killed a whole run *(CRITICAL)*
- **Symptom:** `/ouroboros:run` reached the Deliver phase and fanned out all
  **14 acceptance criteria**. Every one of the 14 implementation sessions failed
  **instantly** — zero tokens consumed, zero tool calls. The job auto-cancelled;
  working tree stayed clean. No code written.
- **Error:**
  ```
  API call failed after 3 retries:
  HTTP 429: Usage limit reached for 5 hour.
  Your limit will reset at 2026-06-07 21:41:18
  ```
- **Root cause:** The `hermes` implementation agent uses **Z.AI/GLM (glm-5.1) as
  a single backend with no fallback** (`fallback_providers: []`). The 5-hour
  rolling quota was exhausted. Three factors amplified it into a stampede:
  1. Orchestrator fanned out **3 concurrent** hermes→Z.AI sessions
     (`max_parallel_workers: 3`).
  2. Each hermes session **retried 3×** on the rejecting endpoint
     (`agent.api_max_retries: 3`).
  3. No fallback provider → nowhere to shed load.
  Ouroboros's built-in `usage_limit_pause_hours: 5.0` did **not** recognize the
  hermes 429 string, so the job hard-failed instead of pausing gracefully.
- **Resolution (3-layer durable fix, all reversible):**
  | Layer | Change | File |
  |---|---|---|
  | 1 | `max_parallel_workers: 3 → 1` (serialize ACs) | `~/.ouroboros/config.yaml` |
  | 2 | `agent.api_max_retries: 3 → 1` (stop pounding endpoint) | `~/.hermes/config.yaml` |
  | 3 | New pre-flight quota probe `scripts/ooo_preflight.sh` | project root |
  - Run left **paused**, awaiting explicit resume once quota resets. Pre-flight
    script tested and confirmed it correctly detects a live 429 and reports the
    reset time instead of stampeding.
- **Source:** `3b9df08d…`, `af8d7c94…`

### 1.2 litellm `UnsupportedParamsError` — GLM doesn't support `response_format`
- **Symptom:** `ouroboros_qa` (seed quality grading) failed twice during QA.
- **Error:**
  ```
  litellm.UnsupportedParamsError: zai does not support parameters:
  ['response_format'], for model=glm-5.
  To drop these, set litellm.drop_params=True
  ```
- **Root cause:** The Z.AI/GLM provider can't accept OpenAI's `response_format`
  param that Ouroboros uses for structured JSON output.
- **Resolution:** Fell back to manual QA — the assistant graded the seed locally
  as the QA Judge (passed at 0.94 on iteration 2). Underlying fix would be
  `litellm.drop_params=True`. Occurred at least 2×.
- **Source:** `afc9671e…`

### 1.3 Gemini CLI `ModelNotFoundError` — interview couldn't start
- **Symptom:** Ouroboros interview repeatedly failed to generate questions.
- **Error:** `ModelNotFoundError: Requested entity was not found` (full Gemini
  CLI traceback).
- **Root cause:** Config referenced Claude models (`claude-opus-4-6`,
  `claude-sonnet-4-20250514`) but the interview tool internally delegates to the
  **Gemini CLI**, which couldn't resolve those names.
- **Resolution:** Manually edited `~/.ouroboros/config.yaml` to set all model
  refs to `gemini-2.5-flash`. Recurred several times before the edit stuck.
- **Source:** `dbf5edfb…`

### 1.4 Gemini CLI — empty response
- **Symptom:** Gemini CLI spawned with `--model gemini-2.5-flash` but returned
  nothing on a trivial probe ("Reply with exactly: OK").
- **Error:** `ProviderError('Gemini CLI produced an empty response')`
- **Root cause:** Probe prompt too short to yield parseable tokens. Judged
  non-blocking for real (longer) interview prompts.
- **Source:** `dbf5edfb…`

### 1.5 Gemini CLI adapter — `'dict' object has no attribute 'role'`
- **Symptom:** Adapter init and command build succeeded, but message
  role-mapping crashed.
- **Error:** `AttributeError: 'dict' object has no attribute 'role'`
- **Root cause:** Adapter received raw dicts where typed `Message` objects were
  expected — an Ouroboros adapter bug.
- **Resolution:** Worked around by continuing with other attempts; not a clean
  fix.
- **Source:** `dbf5edfb…`

### 1.6 litellm — missing `__version__`
- **Symptom:** Diagnostic version check on litellm failed.
- **Error:** `AttributeError: module 'litellm' has no attribute '__version__'`
- **Root cause:** Installed litellm build doesn't export `__version__`
  (version/packaging mismatch). Points to a fragile dependency install.
- **Source:** `dbf5edfb…`

---

## 2. MCP Server / Setup / Config Issues

### 2.1 Python version mismatch — MCP server wouldn't start
- **Symptom:** `/ouroboros:run` and friends unavailable; MCP server failed to
  start.
- **Error:** `ouroboros-ai[claude]` requires Python ≥ 3.12, but system default
  was **3.11.15**; the uvx wrapper used the system Python.
- **Resolution:** Pinned `--python 3.12` in `~/.claude/mcp.json`; restart Claude
  Code.
- **Source:** `af8d7c94…`

### 2.2 MCP server "not configured" — only degraded mode available
- **Symptom:** `/ouroboros:run` returned a fallback message; full execution mode
  unavailable.
- **Error:**
  ```
  Ouroboros MCP server is not available / not configured.
  To enable full execution mode, run: /ouroboros:setup
  ```
- **Resolution:** Ran `/ouroboros:setup` (detected Python 3.14.4, uvx 0.11.7),
  registered the server in **uvx mode**, updated `~/.claude/mcp.json`, wrote
  CLAUDE.md. **Required a Claude Code restart** to activate.
- **Source:** `759d9855…`, `5369ce5e…`

### 2.3 MCP reconnect failure (`-32000`)
- **Symptom:** `/mcp` reconnect of the Ouroboros plugin failed.
- **Error:** `Failed to reconnect to plugin:ouroboros:ouroboros: -32000`
  (JSON-RPC server error).
- **Resolution:** Not explicitly resolved in-transcript; effectively required a
  restart/reinit.
- **Source:** `6e6c7fa9…`

### 2.4 `permission_mode: acceptEdits` → command timeouts & auto-denial *(blocker)*
- **Symptom:** In headless background execution, bash/git commands (pytest,
  `git`, `review diff`) hit a permission prompt no one could answer → timed out
  → auto-denied (`⏱ Timeout — denying command`). File writes worked, but
  verification steps never ran → acceptance criteria never completed → execution
  frozen.
- **Root cause:** `orchestrator.permission_mode: acceptEdits` auto-approves file
  writes but **still prompts for bash commands**; impossible in a headless
  runtime.
- **Resolution:** Set `orchestrator.permission_mode: bypassPermissions` (to
  match the already-correct `opencode_permission_mode`). Requires `/mcp`
  reconnect to take effect.
- **Source:** `afc9671e…`

### 2.5 Config edits don't apply without `/mcp` reconnect
- **Recurring friction:** Every change to `~/.ouroboros/config.yaml` or
  `~/.hermes/config.yaml` required an `/mcp` reconnect / Claude Code restart to
  take effect — easy to forget, and the cause of repeated "still broken after I
  fixed it" confusion.
- **Source:** `afc9671e…`, `af8d7c94…`

### 2.6 `config.yaml` modified-since-read conflict
- **Symptom:** Editing `~/.ouroboros/config.yaml` failed with
  "File has been modified since read…".
- **Root cause:** Another process (Ouroboros itself or a linter) rewrote the
  config between read and write.
- **Resolution:** Re-read then re-edit.
- **Source:** `dbf5edfb…`

---

## 3. Execution / Orchestration Issues

### 3.1 MCP server restart orphaned a running job ("zombie" execution)
- **Symptom:** Job `job_51dcdee0ceaa` frozen at **0/14** completed; event cursor
  stalled (293→296) across many wait cycles; zero worker activity. The MCP server
  had restarted (~09:44Z) and re-registered all 29 tools, killing the hermes
  workers mid-build — but the DB still showed the job "running."
- **Root cause:** Server restart (likely triggered by backend instability) left
  stale persisted "running" state with no worker actually advancing it.
- **Resolution:** Detected via log/cursor analysis; explicitly cancelled the job;
  fixed the underlying `permission_mode` config (see 2.4).
- **Source:** `afc9671e…`

### 3.2 `job_wait` poll timeout (non-fatal)
- **Symptom:** The monitoring wait call itself timed out while the underlying run
  was still progressing.
- **Quote:** "The wait call itself timed out (not the job). Retrying."
- **Resolution:** Auto-retried the status check; not a job failure.
- **Source:** `3b9df08d…`

### 3.3 Code thrashing — duplicate output directories
- **Symptom:** Worker produced overlapping output dirs (`cost_engine/`,
  `engine/`, `custom_costing/`) plus scratch files (`_explore.py`, `_dump_ss.py`,
  `_verify_recompute.py`) instead of converging.
- **Root cause:** GLM instability + inability to verify sub-tasks (blocked by the
  permission timeouts in 2.4) → engine kept regenerating instead of advancing.
- **Source:** `afc9671e…`

### 3.4 Tool permission stream closed before response
- **Symptom:** A permission request failed mid-stream.
- **Error:** `Tool permission request failed: Error: Tool permission stream
  closed before response received`
- **Root cause:** Stream/permission negotiation dropped (timeout or
  disconnection). Seen in more than one session.
- **Source:** `f71e79d1…`, `9d1fe428…`

### 3.5 No in-flight sessions to resume
- **Symptom:** Attempts to resume a prior run found nothing in `running`/`paused`
  state.
- **Quote:** "There are no in-flight Ouroboros sessions to resume."
- **Root cause:** Prior runs had completed/failed/cancelled; EventStore empty.
  Resume state was not transparent to the user.
- **Source:** multiple transcripts

---

## 4. Interview / Seed Tooling Issues

### 4.1 `ouroboros_interview` — shell metacharacter blocked
- **Symptom:** Interview tool rejected a question containing `;`.
- **Error:** `Shell metacharacter detected in last_question
  details={'pattern': ';'}`
- **Resolution:** Re-sent the question without the semicolon; interview resumed.
- **Source:** `afc9671e…`

### 4.2 `AskUserQuestion` — malformed question payloads
- Two distinct validation failures generated by the interview flow:
  - **Missing `questions`:** `InputValidationError: AskUserQuestion … the
    required parameter 'questions' is missing`. — `afc9671e…`
  - **Missing option description:** `… parameter
    'questions[0].options[3].description' is missing` ("expected string, received
    undefined"). — `f71e79d1…`
- **Root cause:** Ouroboros interview question generator emitted structurally
  invalid question objects.

### 4.3 `/ouroboros:run` with no seed file
- **Symptom:** Running without a seed argument and with no `seed.yaml` present
  blocked execution.
- **Quote:** "I don't see a seed YAML file … I need a Seed specification."
- **Root cause:** Run invoked before a seed was generated; the `.yaml` files in
  the project were config, not seeds.
- **Resolution:** Directed to `/ouroboros:seed` or `/ouroboros:interview` first,
  or pass/paste a seed.
- **Source:** `040594bf…`

---

## 5. Lower-Severity / Environmental Friction

- **MEMORY.md missing:** Ouroboros tried to read a not-yet-existing
  `memory/MEMORY.md`; non-critical. — `dbf5edfb…`
- **`git mv` on untracked file:** `fatal: not under version control` (Exit 128)
  when renaming `Storage -TOOL.xlsx`; used plain `mv` instead. Environmental, not
  Ouroboros. — `afc9671e…`
- **Harness sleep-chaining block:** Attempt to `sleep 30 && cat …output` was
  blocked; should use `run_in_background`/Monitor. Usage pattern, not Ouroboros.
  — `dbf5edfb…`
- **Command typo:** `/mco` → "Did you mean /mcp?". Trivial. — `6e6c7fa9…`
- **User frustration with a sticky error:** "I have already done that twice and
  still this error is not going away" — symptom of config changes not taking
  effect without a reconnect/restart (see 2.5). — `af8d7c94…`

---

## Themes & Takeaways

1. **Single LLM backend, no fallback** is the root of the worst failures
   (1.1, 1.2). A 429 or an unsupported param takes down the entire run. Highest-
   leverage fix: configure fallback providers and make
   `usage_limit_pause_hours` actually catch hermes 429 strings.
2. **Provider/param mismatches** (response_format, model names, empty responses,
   litellm version) show the LLM adapter layer is brittle across Gemini and
   Z.AI/GLM.
3. **Config changes silently require a reconnect/restart** (2.5) — the cause of
   repeated "I fixed it but it's still broken" loops.
4. **Headless permission mode** must be `bypassPermissions` or the orchestrator
   freezes waiting on prompts no one can answer (2.4 → 3.1, 3.3).
5. **Seed/interview tooling emits invalid payloads** (4.1, 4.2) — input
   validation on generated questions needs hardening.
6. **Setup is fragile** (Python pin, uvx registration, restart requirement) and
   resume state is opaque.

## Mitigations already in place
- `max_parallel_workers: 1`, `api_max_retries: 1`, `scripts/ooo_preflight.sh`
  quota probe (run **before** every `ooo run`).
- `permission_mode: bypassPermissions`.
- `--python 3.12` pinned in `~/.claude/mcp.json`.
- Convention: probe Z.AI quota → `/mcp` reconnect → re-run.
