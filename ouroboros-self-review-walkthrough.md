# Ouroboros, End-to-End — A Self-Review Walkthrough

> A techno-functional understanding of how Ouroboros is designed and architected,
> developed through a worked example: **Ouroboros reviewing its own current branch**
> (`feat/runtime-param-negotiation`).
>
> Format: Socratic interview (one question at a time). Depth: architectural/conceptual.
> Each section captures the *elicited understanding* (what the user reasoned out),
> *grounded against the actual code/docs*, with corrections and extensions noted.

---

## The spine of the example

We carry one task through every stage of the system:

> "Review the current branch of this repository for correctness and regressions."

This is deliberately a **brownfield** task on an existing codebase (this repo),
which exercises the entry-point dispatch, the interview, seed crystallization,
execution against a runtime, the evaluation gate, and the evolutionary loop —
i.e. the full breadth of the system.

## Breadth axis (stages we will traverse)

| # | Stage | Core subsystem(s) | Phase |
|---|-------|-------------------|-------|
| 0 | Entry & framing | Skill dispatch, MCP hub, OS/Apps/Shell layering | — |
| 1 | Interview | `bigbang/` — Socratic interviewer, ambiguity score | Phase 0: Big Bang |
| 2 | Seed | `core/seed.py`, AC tree, ontology | Phase 0 (output) |
| 3 | Route | `routing/` — PAL Router, complexity, tiers | Phase 1 |
| 4 | Execute | `execution/` Double Diamond, runtime adapters, resilience | Phase 2 + 3 |
| 5 | Evaluate | `evaluation/` 3-stage gate | Phase 4 |
| 6 | Evolve | drift, lineage, secondary loop | Phase 5 + loop |
| X | Cross-cutting | Event sourcing / Ledger, control contract, observability | all |

---

## Stage 0 — Entry & framing

**Probe:** What do you type to "review this branch," what receives it first, and is review a core skill / a UserLevel plugin / an MCP route?

**Elicited model:** Invoke `ooo review`; the system interprets it as a whole workflow distinct from `build`, setting the pathway for the rest — therefore it "must be a plugin."

**Grounded correction:**

1. **`ooo review` is not a real command.** The 20 core skills are fixed; there is no `review`. "Review this branch" is *composed* from primitives — most directly `ooo qa` (qa-judge agent) or `ooo evaluate` (3-stage gate), typically after `ooo brownfield` registers the repo as interview context.

2. **The "plugin" intuition is the correct layer-model — but half-shipped.** Domain workflows like PR review are explicitly earmarked for the **UserLevel Programs Layer** (`Q00/ouroboros-plugins`, e.g. `github-pr-ops`), *not* core. But the third-party install surface (`ooo plugin add`) does **not exist on `main`** yet — only first-party UserLevel programs ship today (`ooo auto`, `ooo run`, `ooo pm`). Conceptually review → plugin; today review → composed core skills.

3. **First-hop component — the central architectural fact.** A *skill is a declarative dispatch shim, not the executor.* Two paths:
   - **Claude Code**: `CLAUDE.md` instructs the agent to read `skills/<name>/SKILL.md` and follow it directly (no router code).
   - **Codex/Hermes/OpenCode**: the stateless `ouroboros.router` parses the `ooo <skill>` prefix → resolves a `SKILL.md` → reads `mcp_tool:`/`mcp_args:` frontmatter → returns dispatch metadata → runtime invokes that **MCP tool**.
   - Real work lives behind **MCP tools** in core (`ouroboros_execute_seed`, `ouroboros_evaluate`, `ouroboros_interview`). The MCP server is intentionally unaware of `ooo` syntax.

   **Three distinct layers the initial model collapsed into "plugin":**

   ```
   ooo review (text)
      └─► Skill (SKILL.md)      ← declarative dispatch shim     [Registry / Section 1]
             └─► MCP tool       ← execution entry-point to core
                    └─► core phases (interview/seed/route/execute/evaluate)
      (UserLevel plugin/app)    ← composes MCP tools into a domain workflow [Section 7]
   ```

**Key takeaway:** *skill ≠ MCP tool ≠ plugin.* Dispatch is declarative; execution is MCP tools over a runtime-agnostic core; domain workflows are a separate composable layer.

### Why this split? (design rationale)

**Probe:** Why keep the MCP server ignorant of `ooo` syntax and make skills thin shims rather than housing logic in the skill?

**Elicited model:** Skills are agent-side primitives; moving logic to core gives full capability regardless of host agent (portability), enables headless/programmatic use, and immunizes against host-agent version churn.

**Grounded + extended:**
- **Runtime portability** — orchestrator speaks only the `AgentRuntime` protocol over normalized `AgentMessage`/`RuntimeHandle`/`TaskResult`; 8 adapters satisfy it; core never sees backend internals.
- **Headless access** — execution surface is MCP tools, not prompts, so `ooo auto`/CI drive identically with no human.
- **Version immunity** — the contract is the MCP tool signature (ouroboros's cadence), not the host CLI's.
- **No logic drift (added)** — thinness *is* the contract: "adding a command is a `SKILL.md`-only change; no runtime parser branches or MCP special cases." Single source of truth in core.
- **Replayability requires logic-in-core (added)** — every MCP-tool action becomes a Seed-bound, ledger-recorded event (event-sourced SQLite, append-only, replayable). Logic in the agent layer would be unrecorded/non-deterministic. *Core-resident logic is what makes the system auditable;* plugins declare scoped capabilities against that contract.
- **Analogy:** core = kernel (syscalls = MCP tools); skills/ourocode/Claude Code = interchangeable shells; plugins = installable programs.

**Nuances:** (a) "skill belongs to the agent" holds for Claude Code (host-native); for Codex/Hermes/OpenCode ouroboros *installs* skills and intercepts via its own router — a boundary object projected *into* the agent. (b) Honest cost: feature parity across runtimes is *not* guaranteed (differing tool sets/permissions/streaming); the adapter abstraction is acknowledged-leaky.

---

## Stage 1 — Interview (Phase 0: Big Bang)

**Probe:** Why force a Socratic interview for a review task where the goal "feels obvious"? What ambiguity exists in "review this branch"?

**Elicited model:** "Review for *what*? — security / architectural anti-patterns / best-practice / lint / malicious code?" Plus constraints. Skipping it → AI makes silent assumptions, spends tokens, produces an answer to an ask the user never had → rework. *"AI optimizes to obey, not to be right."*

**Grounded + extended:** Ambiguity is an **LLM-judged score over weighted dimensions**, gated at `AMBIGUITY_THRESHOLD = 0.2` (`bigbang/ambiguity.py`). Dimensions differ by mode:

| Dimension | Greenfield | Brownfield (our case) |
|---|---|---|
| Goal clarity | 40% | weighted |
| Constraints | 30% | 0.25 |
| Success criteria | 30% | 0.25 |
| **Context clarity** | — | 4th dimension |

- "Review for what" = **Goal** (heaviest). "Constraints if any" = **Constraints**. The dimension the initial model missed: **Success criteria** — what a *finished* review looks like (output format, severity taxonomy, areas covered, explicit out-of-scope).
- **Per-dimension floor** (`qualifies_for_seed_completion`, `get_completion_floor_failures`): each dimension must independently clear a minimum — you can't mask a vague goal behind crisp constraints. The "silent assumption about goal" failure is structurally blocked.
- **User controls termination**: scoring starts after 3 rounds; seed-closer pressure at `0.25`; `round_number` has *no upper cap*. It is a convergence loop toward ≤0.2, not a fixed questionnaire.
- Questions rotate through structured **Socratic perspectives** (`InterviewPerspective` + strategies), not ad hoc.
- Brownfield adds a 4th **context-clarity** dimension — the system formally recognizes that reviewing existing code requires understanding the existing code (baseline/diff scope/intent).

### Where review context comes from (two layers)

**Probe:** Does the human supply context, does ouroboros read the repo, or both? Why is context first-class rather than folded into goal/constraints?

**Elicited model:** Context is an *independent variable* affecting the review regardless of how clear the other 3 dimensions are. Folding it in would give a *false sense of clarity* when context is poorly understood but the other dimensions are crisp. (Orthogonality / aggregate-masking argument — correct.)

**Grounded — both sources, two layers:**
- **Registry layer** (`ooo brownfield` → `bigbang/brownfield.py`): DB-backed; scans a root for git repos + *linked worktrees* (`git worktree list --porcelain`), detects origin/GitHub remotes, sets **default repos** = interview context (selects *which* code is in scope; registers this repo for self-review).
- **Extraction layer** (`bigbang/explore.py` → `CodebaseExplorer`): scans config markers → tech stack/deps; discovers key type defs (struct/class/interface/enum/const); **LLM-summarizes** protocols/patterns; injects a *condensed summary into the interview system prompt*. Each dir has a **role**: `primary` (modify) vs `reference` (read-only) — self-review repo = `primary`.
- **Why this vindicates the orthogonality point:** context-clarity is scored against *machine-extracted ground truth*, an independent evidence source. The human can't merely *assert* clarity — the explorer reads the actual repo. Context therefore *cannot* be folded in; it has its own evidence.
- **Precision:** explorer = structural grounding for the *spec*; the branch *diff* (`git diff main...HEAD`) is read at *execution* time, not baked into the explorer summary.

---

## Stage 2 — Seed (Big Bang output / immutable contract)

**Anatomy** (real artifact `seed_78c8e6e41813.yaml`): `goal`, `task_type`, `brownfield_context` (`context_references` with `role: primary|reference` + summaries, `existing_patterns`, `existing_dependencies` — straight from registry+explorer), `constraints`, `acceptance_criteria` (flat; tree built at execution), `ontology_schema` (typed `fields` = structure of outputs), `evaluation_principles` (**weighted** rubric summing to 1.0 = Stage-2 scoring), `exit_conditions`, `metadata` (`ambiguity_score: 0.1105` = gate cleared, `interview_id` = provenance, `version`, `parent_seed_id`).

**Probe:** How is an *immutable* seed compatible with an *evolutionary loop*? What does `parent_seed_id` imply happens when a review shows the spec was wrong? Why immutable-with-lineage vs editable?

**Elicited model:** Evolution = a chain of immutable seeds linked by `parent_seed_id`; a wrong spec → mint a new seed pointing at the old one and proceed; rationale = auditability + replayability. (Correct.)

**Grounded + extended:**
- **Deep immutability:** *every* nested model is `frozen=True` (ExitCondition, EvaluationPrinciple, OntologyField, OntologySchema, ContextReference, BrownfieldContext, SeedMetadata). The whole spec graph is one immutable value → hashable, replayable, safe to reference from event history.
- **Lineage is a subsystem** (`core/lineage.py`): `GenerationRecord` (chains generations), `OntologyLineage` + `OntologyDelta.compute(old,new)` (measures structural delta between generations), `RewindRecord` (rewind via `ouroboros_evolve_rewind`), `EvaluationSummary`/`FeedbackMetadata`/`seed_quality_canary_feedback` (eval feeds the next generation — the loop).
- **Authorization (the unnamed piece):** minting a new seed isn't free. Consensus trigger #1 = "seed modification requires consensus"; #2 = "ontology evolution." So spec correction is **consensus-ratified**, which is *why* `OntologyDelta.compute` exists (detect when a change is structural enough to demand consensus).
- **Governance triad:** immutable (no silent drift) + lineage (auditable/replayable) + consensus-gated mutation (ratified, not unilateral). Evolution = *append a consensus-ratified child whose delta from parent is measured* — never in-place edit. (Actually occurs at the Evolve stage, post-evaluation.)

---

## Stage 3 — Route (Phase 1: PAL Router)

**Probe:** What signals should set the tier for a review task? Is frugal-first the right default for review?

**Elicited model:** Signals = (1) nature of task (docs ≪ architectural review), (2) scope/size (module vs whole codebase). Review is inherently complex → frugal-first risks missing intricate/elusive vulnerabilities.

**Grounded + extended:**
- Actual formula: `complexity = 0.30·norm(tokens/4000) + 0.30·norm(tools/5) + 0.40·norm(depth/5)` → `<0.4` FRUGAL · `0.4–0.7` STANDARD · `>0.7` FRONTIER (or "critical" forces FRONTIER). Only **three inputs**: `token_count`, `tool_dependencies`, `ac_depth`. **No `task_type`.**
- "Nature of task" is **not measured directly** — it surfaces *emergently* via structure (a thorough review → deeper AC tree + more tools → higher depth/tool components; `ac_depth` is the heaviest weight at 40%). Router measures **structural proxies, not semantics**.
- "Scope/size" maps directly to `token_count` + `ac_depth`. ✓
- Escalation: `FAILURE_THRESHOLD = 2` consecutive failures → next tier; Frontier failure → `escalation.stagnation.detected`. Downgrade after sustained success; similar tasks (Jaccard ≥ 0.80) inherit tier.
- **"Review breaks frugal-first" is a genuine seam:** (a) a *shallowly-specified* review scores < 0.4 → routes FRUGAL despite being high-stakes; safety depends on decomposition deepening the tree. (b) Escalation is **failure-driven**, but review failures are **silent false-negatives** — a cheap model emits plausible markdown that passes mechanical + semantic checks while *missing* the bug; no failure fires → no escalation. Frugal-first assumes *observable* failure (constructive tasks: won't compile / tests red); review is a *judgment* task, not mechanically falsifiable.
- **System's real answer to the critique:** not routing but the **multi-model consensus** at evaluation (expensive, trigger-gated). Routing can't catch judgment-quality misses, so the cost is paid at the gate. (Reverse-derives the rationale for Stage 4 consensus.)

---

## Stage 4 — Execute (Phase 2: Double Diamond + decomposition)

> **Correction (2026-06-11, per discussion [#1385](https://github.com/Q00/ouroboros/discussions/1385)):**
> this stage describes the **Double Diamond design**, which the owner's verification
> found is **off the live path** — `DoubleDiamondExecutor`, `execution/atomicity.py`,
> and `execution/decomposition.py` have **no non-test caller in `src/`** (recent runs:
> 115 `execution.ac.completed` events, *zero* atomicity-check events). The live
> executor is **`orchestrator/parallel_executor.py`**, with its own profile-based split
> (`axis: testable_unit`, `min_unit`, `atomic_verifier_verdict`). The `atomicity.py` /
> `decomposition.py` modules are slated for **deletion** (not repair), and the
> per-node "tier routing" below is reshaped to a **reasoning-effort dial**. See
> [`investment-plans/06`](investment-plans/06-atomicity-decomposition-reliability.md)
> and [`investment-plans/04`](investment-plans/04-frugality-and-spend-decision.md).

**Probe:** Why decompose a review into a tree vs one-shot? Purpose of the atomicity gate? What must hold for sibling sub-reviews to run in parallel?

**Elicited model:** Decomposition makes large tasks fit smaller context windows; siblings need clean separation of concerns.

**Grounded + extended:**
- **MECE** (decomposition prompt): children **M**utually **E**xclusive (= "separation of concerns" ✓) **and** **C**ollectively **E**xhaustive (the missed half — together cover the *entire* parent → no silent coverage gaps; "find bugs" has no completeness guarantee, a MECE tree does).
- **Decomposition buys more than context fit** (weakest reason): (a) *manufactured verifiability* — can't evaluate "find all bugs," can evaluate a leaf claim; each atomic leaf gets its own `ACResult` verdict → makes Stage 4 possible; (b) *independent tier routing* per node; (c) *resilience isolation* — stuck branch gets local lateral thinking; (d) *drift containment*.
- **Atomicity gate** = recursion base case + reliability gate. `AtomicityCriteria` is multi-threshold (`max_complexity`, `max_tool_count`, `max_duration_seconds`), not file-count. Prevents *under-decomposition* (sprawling one-shot) and *unbounded recursion*.
- **Parallel safety is declared + enforced**, not assumed: each child emits zero-based indices of siblings it depends on (`[]` = independent); `dependency_analyzer` topo-sorts into **dependency levels**; `parallel_executor` runs parallel *within* a level, sequential *across*. Review is "embarrassingly parallel" (read-only, no shared-state writes) unlike build tasks.
- **Doc drift flagged:** `decomposition.py` has `MAX_DEPTH = 2`, `COMPRESSION_DEPTH = 3`; `architecture.md` claims `MAX_DEPTH = 5` (stale).


