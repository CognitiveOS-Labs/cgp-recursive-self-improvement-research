# AHSE Blueprint — RSI via .cgp Cognitive Patches

Produced per the AHSE skill pipeline (STAGE-100 → STAGE-700 complete;
STAGE-999 **deferred** — see the terminal section). This is a **research
synthesis blueprint**, grounded in the actual state of this workspace (no GPU
available at authoring time; lab organization structure; cpm v1.0.9-alpha
implementation truth).

## STAGE-100: EMPATHIC_NEED_DISCOVERY

* **Stage Code**: `STAGE-100`
* **Focus**: Who & Why
* **Outputs**:
  - **Target users**: CognitiveOS ecosystem builders (package authors, distro
    integrators, on-device AI product teams) who need models that improve on
    their private data *without* catastrophic forgetting, server-side drift,
    or unverifiable self-training.
  - **Latent need**: A distribution unit (the `.cgp`) that can carry its own
    improvement lifecycle — not just weights+prompts — including the safety
    rails (verifier, gate, rollback) that make self-training deployable.
  - **Root problem statement**: Recursive self-training pipelines collapse
    (Shumailov et al., 2024) because synthetic samples are ingested without an
    external correctness check and without regression guards. .cgp + cpm make
    an artifact-level, auditable gate possible — but no closed-loop,
    verifier-gated RSI artifact existed.

## STAGE-200: 5W2H_STRATEGIC_MATRIX

| 5W2H | Answer |
|---|---|
| **WHAT** | A closed-loop RSI pipeline realized as `.cgp` packages: `base-model-v0.cgp` (M₀) + `rsi-controller.cgp` (verifier/gate/tune/status). |
| **WHY** | Prove .cgp can hold a full improve-yourself lifecycle with zero-trust promotion — the packaging-level answer to model collapse. |
| **WHERE** | CognitiveOS ecosystem (on-device/edge distros, lab CI, GPU training hosts); Python-algorithmic-synthesis domain. |
| **WHEN** | Research cycle now (framework + paper); multi-generation run on GPU host; per-generation `base-model-v{t+1}.cgp` releases thereafter. |
| **WHO** | Jean Machuca (CognitiveOS-Labs) + CognitiveOS-Labs community; consumers: CognitiveOS distro integrators; audience: ML systems researchers (arXiv). |
| **HOW** | Generate → Verify (deterministic) → Mix (20% anchors) → `cpm tune` (LoRA) → FP16 merge → GGUF export → Evaluate (B_target/B_general) → Gate (ε/δ) → Promote/Rollback via `cpm`. |
| **HOW MUCH** | ≤ $0 compute for this repo (framework, CPU); 1× 24 GB GPU for the M₀→M₆ run (est. 6–12 GPU-hours); FP16 master storage ~4 GB × 6 generations; $0-paper-publication budget (arXiv + Zenodo). |

## STAGE-300: VISION_MISSION_SYNTHESIS

* **Vision**: A world where every CognitiveOS device carries a model that
  safely keeps getting better at its owner's work — with its improvement
  history auditable in versioned `.cgp` artifacts.
* **Mission**: Deliver and validate the first zero-trust, verifier-gated RSI
  loop over `.cgp` packages, publish the method and the artifacts, and make
  the experiment replicable by any lab with one GPU.

## STAGE-400: HW_SW_ARCHITECTURAL_IDEATION

* **Hardware architecture**: reference training host — 1× NVIDIA GPU 24 GB,
  64–128 GB RAM, NVMe ≥ 200 GB; inference endpoint via CognitiveOS
  `code-inference` (http://api:8000/v1) or vLLM (FP16, ≥16k ctx).
* **Software architecture**:
  - *Distribution*: `.cgp` archives via `cpm` (pack/verify/install/tune);
    artifacts as GitHub Releases.
  - *Control plane*: `rsi-controller.cgp` MCP stdio server
    (`cognitiveos.rsi.verify/gate/tune/status`) + orchestrator `rsi/controller.py`.
  - *Data plane*: `b_target.jsonl`, `b_general.jsonl` (held-out),
    `anchors.jsonl` (20% replay), JSONL metrics stream.
  - *Verification plane*: stdlib AST policy + rlimit-sandboxed subprocess
    unit-test runner (no LLM judge anywhere in the verdict path).
* **HW/SW boundary**: GPU host exposes an OpenAI-compatible endpoint; the
  orchestrator never holds weights inline — artifacts flow through `cpm` and
  the registry, mirroring the ecosystem's install path.

## STAGE-500: LEAN_MVP_HYPOTHESIS_MAPPING

| # | Riskiest assumption | Validation |
|---|---|---|
| H1 | `cpm pack`/`verify` accept a manifest-only remote-weights model and an MCP `#!` tool (exec bit). | **VALIDATED locally** — both packages verified ✓ (2026-11-17). |
| H2 | A stdlib-only verifier is sufficient for 100% deterministic gating of synthetic Python data. | **VALIDATED** — 20/20 unit tests; anchors 16/16 verified by it. |
| H3 | The promotion gate logic is exactly implementable (strict ε, δ bound) and rolls back deterministically. | **VALIDATED at unit level** — gate tests + rollback dry-run test ✓. |
| H4 | An LLM-generated (problem, solution) stream can yield ≥50% verified-pass rate at temperature 0.9 on this domain. | **UNVALIDATED — requires GPU/endpoint.** Measured in the GPU run (pass-rate metric is logged per generation). |
| H5 | M₀→M₆ shows monotonic B_target gains with B_general retention (primary hypothesis). | **UNVALIDATED — requires GPU run.** |
| H6 | Arm C (no verifier) collapses/regresses, arm B plateaus below A. | **UNVALIDATED — requires GPU run.** |

*MVP specification*: exactly H1–H3 (the trustworthy machinery) + full paper —
shipped. H4–H6 are the funded next step, fully protocolized in
`experiments/PROTOCOL.md`.

## STAGE-600: BUILD_MEASURE_LEARN_SIMULATION

* **Simulation Run 1 (this environment, no GPU)**: built the complete
  pipeline; 20/20 unit tests green; mock e2e promotes 2/2 generations; mock
  gate-rejection exercises rollback; cpm pack/verify ✓ for both packages;
  benchmarks self-consistent (8/8 solvable); anchors verified 16/16.
  → **PERSEVERE** on the architecture; the loop shape is sound.
* **Pivot log**:
  - PIVOT-1: Initial benchmark/anchor JSON had malformed `args` wrappers
    → fixed; all items re-verified with the actual verifier (an-001).
  - PIVOT-2: Float-noise broke strict ε semantics (`0.52-0.50 > 0.02`)
    → gate now uses `math.isclose(abs_tol=1e-9)`; strictness preserved and
    tested.
  - PIVOT-3: Mock client couldn't distinguish synthesize vs solve prompts
    → detect on prompt markers; mock e2e green incl. rollback path.
  - PIVOT-4: `cpm`'s bundled schema rejects the `dependencies` manifest field
    → documented in `docs/cpm-implementation-notes.md`; requirement documented
    in READMEs instead. Routing field omitted (spec/implementation divergence).
* **Burnout check**: none of the pivots is fatal; H4–H6 are unproven but not
  impossible — they gate on GPU access, not on viability.

## STAGE-700: FINAL_PRODUCT_BLUEPRINT

* **Executive summary**: A verifier-gated, promotion-gated RSI loop
  distributable as two `.cgp` packages, with a CI-built paper and a frozen,
  reproducible GPU-run protocol. (No novelty/priority claim is made here;
  see `docs/adr/ADR-005-no-fabricated-results.md`.)
* **Software stack diagram**: see README *System architecture*.
* **Deployment roadmap**:
  1. **Now**: repos live; paper CI green; packages verified; protocol frozen.
  2. **GPU run**: baseline M₀ → 6 generations → arms A/B/C → drift measurement.
  3. **Release**: tag `v0.2.0` (not `v0.1.0`, which is the framework-only
     release); paper PDF + `.cgp` artifacts attached; Zenodo DOI minted;
     arXiv submission; results table written from verbatim logs.
  4. **Next iterations**: scale base model (2–8B), swap domain
     (tool-use/intent routing), multi-seed error bars, registry publishing.

## STAGE-999: SUCCESS_DELIVERY — DEFERRED

Terminal code: **deferred**. STAGE-700 (final blueprint) is complete;
STAGE-999 is NOT granted. H4–H6 are unvalidated and the multi-generation run
is awaiting GPU access (`experiments/PROTOCOL.md`). Asserting
`SUCCESS_DELIVERY` now would claim measurements that do not exist. See
`RSI_CGP_PROOF.md` §5 and `experiments/RESULTS.md` for the honest ledger.
STAGE-999 closes only when §5.2 is filled from executed-run logs.
