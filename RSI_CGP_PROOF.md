# RSI_CGP_PROOF — Recursive Self-Improvement via .cgp Cognitive Patches

**Technical Report · v0.1.0 · 2026-11-17 · Jean Machuca (CognitiveOS-Labs)**
**Code:** [`CognitiveOS-Labs/cgp-rsi`](https://github.com/CognitiveOS-Labs/cgp-rsi) · **Paper:** [`paper/`](paper/) (CI-built PDF) · **Protocol:** [`experiments/PROTOCOL.md`](experiments/PROTOCOL.md)

> **Epistemic status of this report.** Everything in §5.1 is real: it was
> executed on 2026-11-17 in the authoring environment and is re-verifiable
> with the commands shown. Everything in §5.2 (M₀→M₆) is **not yet executed**:
> the authoring environment had no GPU, no training stack for its Python
> (no torch/peft/transformers wheels for CPython 3.14), and no base-model
> weights on disk. No model-scale numbers are claimed anywhere in this
> document. §7 lists exactly what remains to run and how.

## Abstract

We package a large-language-model improvement loop as versioned CognitiveOS
`.cgp` artifacts and specify a zero-trust, verifier-gated recursive
self-improvement (RSI) pipeline over them. Synthetic training data is admitted
only through an external deterministic verifier; each generation's LoRA
adapter is fused into the FP16/BF16 master (never into GGUF); promotion is
decided by an isolated two-benchmark gate with automatic `cpm` rollback. This
report delivers: (i) the complete pipeline, unit-tested and CI-verified;
(ii) the frozen protocol for the M₀→M₆ experiment (N=6, t=0..5) and its three-arm ablation;
(iii) failure-and-safety design for promotion rollback and quantization
drift. Empirical multi-generation evidence is pending the GPU run specified
in §7; the paper's results section mirrors this status.

## 1. Introduction

Self-training improves LLMs when the model's own outputs are filtered by a
reliable correctness signal [8, 10]. Without such a signal, recursive
self-consumption degenerates performance and diversity — "model collapse" in
the terminology of Shumailov et al. [6]. CognitiveOS distributes models and
skills as `.cgp` archives with an install-time lifecycle (`cpm`). If a `.cgp`
can carry its improve-yourself loop — verifier, gate, rollback — as auditable
artifact content, then "safe self-improvement" becomes an ecosystem property
rather than a per-lab bespoke harness.

**Contributions.**

1. A `.cgp`-native RSI architecture with three enforced invariants (precision,
   verification, promotion) — §2.
2. `rsi-controller.cgp`, a working MCP stdio controller implementing the
   deterministic verifier, the promotion gate, the `cpm tune` Controller
   Pattern, and pipeline status — §3.
3. A frozen, executable experimental protocol (M₀→M₆; arms A/B/C; two isolated
   benchmarks; fixed seeds; verbatim logging) — §4, `experiments/PROTOCOL.md`.
4. Failure & safety design: rollback semantics, hyperparameter adjustment
   mapping, quantization-drift measurement plan — §6.

## 2. System architecture

```text
        ┌─────────────────────────── RSI loop (t = 0..N-1) ──────────────────────────┐
        │                                                                            │
  M_t ──▶ GENERATE ──▶ VERIFY ──▶ MIX (20% anchors) ──▶ cpm tune ──▶ MERGE(FP16) ──▶ GGUF
```

**Invariant 1 — precision.** Master stays FP16/BF16 `.safetensors`.
`rsi/merge.py` refuses to proceed if any merged tensor is non-F16/BF16.
Quantization happens once, after the merge. ADR-001.

**Invariant 2 — verification.** All synthetic samples pass
`rsi/verifier.py`: (a) AST allowlist (`math`, `collections`, `itertools`,
`functools`, `heapq`, `bisect`, `string`, `re`, `operator`, `typing`,
`dataclasses`); (b) no `exec`/`eval`/`open`/`__import__`/`input`/`compile`, no
dunder attribute access; (c) subprocess sandbox (`RLIMIT_CPU=5`,
`RLIMIT_AS=256MB`, `RLIMIT_NPROC=64`, `RLIMIT_FSIZE=1MB`, wall-clock 6s,
isolated cwd); (d) the problem's full hidden suite MUST pass.
Unverified samples are discarded — fail closed. ADR-002.

**Invariant 3 — promotion.**

$$\text{promote} \iff s(M_{t+1}, B_{\text{target}}) > s(M_t, B_{\text{target}}) + \varepsilon \;\land\; s(M_{t+1}, B_{\text{general}}) \ge s(M_t, B_{\text{general}}) - \delta$$

with ε = 0.02, δ = 0.01, strict-greater with `math.isclose(abs_tol=1e-9)`.
On rejection: `cpm install <previous>.cgp` rollback, failure mode logged,
hyperparameters adjusted (target-miss → LR/2; general-regression → LR/2 ∧
temperature/2; both → LR/2 ∧ rank/2, floor 4). ADR-003.

**Isolation.** `B_target` (8 held-out algorithmic problems) and `B_general`
(8 held-out instruction-following tasks) are fixed JSONL, read-only, never
shown to the synthesis prompt, the verifier, or the tuner. The 20% anchor
replay buffer (16 verifier-passing problems, seeded) guards against
forgetting. Both benchmark sets were validated for internal consistency by
executing reference solutions through the same verifier (8/8 pass).

## 3. Implementation

**Packages (both `cpm pack`-ed and `cpm verify`-ed, 2026-11-17):**

| Package | Contents | Size |
|---|---|---|
| `base-model-v0` 0.1.0 | wide_model manifest, remote FP16 safetensors (HF `google/gemma-3-1b-it`), `brain.training` (rank 16, α 32, epochs 2, lr 2e-4, min_samples 50) | 805 B |
| `rsi-controller` 0.1.0 | MCP stdio server: `cognitiveos.rsi.verify` / `.gate` / `.tune` / `.status`; system prompt; tools 0755 | 5.5 kB |

**`cpm tune` Controller Pattern.** `cpm tune base-model-v0` reads
`brain.training`, invokes `cognitiveos.rsi.tune` through the system daemon.
On hosts without the GPU stack the tool returns `E_HW_UNAVAILABLE` — it never
fakes training. On provisioned hosts it validates inputs and hands off to the
host runner (`docs/REPRODUCIBILITY.md` in the code repo).

**Spec ↔ implementation divergences** (measured, documented in
`docs/cpm-implementation-notes.md`): `brain.training` is read by the
implementation but absent from the schemas; `dependencies` is rejected by
cpm's validator (V006) — requirement documented in prose instead;
`brain.wide_model.routing` has divergent spec/implementation shapes — omitted.

## 4. Experimental protocol (frozen)

**Domain.** Python algorithmic synthesis (verifiable by execution; unique
I/O pairs).

**Setup.** `base-model-v0` (gemma-3-1b-it remote FP16 master); seeds fixed
(`RSI_SEED=1337`); ε = 0.02, δ = 0.01; anchor fraction 0.2; samples/gen 64;
generations 6 (t = 0..5, producing M₁..M₆; table reports M₀ baseline + M₁..M₆).

**Metrics per generation** (table reports M₁..M₆). synthetic pass rate · train loss · ‖ΔW_t‖
(adapter weight norm) · B_target accuracy · B_general retention · promotion
decision · inference latency (ms). Logged verbatim to `runs/metrics.jsonl`.

**Arms.** A recursive `.cgp` loop; B single-shot LoRA over M₀ with equivalent
total compute/rank on M₀-generated verified data only; C as A with the
verifier bypassed. All arms share benchmarks, seeds, anchors, budget.

**Promotion rule** is invariant across arms A and C (arm C additionally logs
verdicts-but-admits-all).

## 5. Results

### 5.1 Pipeline validation (executed 2026-11-17; re-verifiable)

| Check | Result |
|---|---|
| Unit tests (verifier, gate, replay, controller, metrics) | **20/20 pass** (`pytest`) |
| CI mock e2e, happy path (2 generations) | promote/promote; metrics JSONL well-formed; table renders |
| CI mock e2e, gate-rejection path (`--fail-at 1`) | gen 1 **rejected**, rollback invoked, `failure_mode` logged |
| Anchor replay set | **16/16** pass the deterministic verifier |
| Benchmarks | 8/8 solvable by reference solutions (via the verifier); JSONL valid; fixed & held-out |
| `cpm pack` / `cpm verify` | both packages ✓ (amd64; rsi-controller tools at 0755) |
| Float-noise ε semantics | strict gate preserved under `isclose(abs_tol=1e-9)`; tested |

### 5.2 Multi-generation experiment (NOT executed — pending GPU)

**No M₀→M₆ numbers exist yet.** The placeholders below are the *pre-registered*
comparison (6 cycles, t = 0..5, M₀ baseline through M₆); they are filled from
verbatim `runs/metrics.jsonl` after the GPU run. Any value appearing before
then would be fabrication.

| Gen | pass% | loss | ‖ΔW‖ | B_target | ΔB_target | B_general | ΔB_general | verdict |
|---|---|---|---|---|---|---|---|---|
| 0 → M₀(base) | — | — | — | PENDING | — | PENDING | — | baseline |
| 1 | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| 2 | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| 3 | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| 4 | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| 5 | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |
| 6 | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING |

**Pre-registered hypotheses** (analysis plan, not results):
H5 — monotone non-decreasing B_target in promoted generations, with B_general
retention ≥ −δ; H6 — arm C < arm A on B_target (collapse prevention), arm B
≤ arm A (recursive necessity). Rejection of H5/H6 falsifies the method as
implemented, and will be reported as such.

## 6. Failure & safety analysis

1. **Gate rejection** (by construction): rollback to the last promoted
   artifact; failure mode in `metrics.jsonl`; bounded adjustment
   (`gate.suggest_hyperparameter_adjustment`). A generation is *evidence*, not
   a *commitment*.
2. **Verifier fail-closed**: timeout/crash/disallowed construct ⇒ sample
   discarded; gate cannot see unverified data.
3. **Quantization drift**: per-generation FP16→GGUF top-1 agreement / KL is a
   required measurement (`docs/REPRODUCIBILITY.md` §6); promoted GGUF MUST
   score within noise of the FP16 master. If drift exceeds noise, the export
   step fails the generation and rolls back.
4. **Collapse surface**: arm C exists to exhibit it; the verifier is the
   intended countermeasure, together with 20% anchor replay.
5. **Rollback is artifact-level** (`cpm install`), not adapter-swap-level —
   no reliance on undocumented `/api/adapter` behavior.
6. **Catastrophic forgetting**: anchor coverage is logged; if replay coverage
   < 20%, the controller aborts the generation.
7. **Documented failure modes to date** (all pre-empirical, fixed and tested):
   float-ε promotion leak (PIVOT-2, BLUEPRINT); benchmark I/O-arg shape bug
   (PIVOT-1); mock client prompt ambiguity (PIVOT-3); cpm `dependencies`
   rejection (PIVOT-4).

## 7. Limitations & remaining work

- **Empirical gap** (primary limitation): Section 5.2 requires the GPU run.
  Minimum: 1× 24 GB GPU, ~6–12 h, ≥ 200 GB disk. Full runbook: code repo
  `docs/REPRODUCIBILITY.md` (sections 3–7).
- **Single domain** (Python algorithmic synthesis); transfer to tool-use
  routing is future work.
- **Single base model** (gemma-3-1b-it); generality across scales untested.
- **non-determinism**: GPU training is not bitwise reproducible; multi-seed
  error bars are planned for the headline table.
- **Verifier strength**: allowlist + hidden tests catch execution failures;
  they do not prove semantic correctness beyond the test suite (tests are the
  spec). Adversarial solution overfitting to visible tests is mitigated by
  keeping suites hidden from synthesis (see `rsi/generator.py` — solutions are
  generated from the problem statement, never from tests).
- **`cpm tune` hot-swap** relies on daemon behavior not yet spec-stable; our
  rollback path intentionally avoids it.

## 8. Conclusion

The `.cgp` format is sufficient to carry not just weights and prompts but a
full, auditable self-improvement lifecycle: verifier, gate, rollback, version,
registry, release. The machinery exists, is packaged, and is tested. The
scientific claim — sustained recursive gains with no collapse — is now a
GPU-hours question, precisely protocolized, with hypotheses and ablations
pre-registered. We report what is true, package the path to what is not yet
known, and make both verifiable.

## References

See `paper/references.bib` (Hu 2021 LoRA; Dettmers 2023 QLoRA; Shumailov 2024
Nature; Wang 2022 Self-Instruct; Zelikman 2022 STaR; Ouyang 2022 InstructGPT;
Wei 2022 verifiers; cpm + product-specs). Paper: `paper/main.tex`.

## Appendix A — verification log (2026-11-17)

```text
$ pytest -q                     → 20 passed
$ python3 scripts/mock_e2e.py   → gen0 promote, gen1 promote; table printed
$ python3 scripts/mock_e2e.py --fail-at 1
                                → gen0 promote, gen1 NO (rollback), failure_mode logged
$ cpm pack --manifest cognitive.json        (base-model-v0)   → ✓ valid
$ cpm pack --bin tools                      (rsi-controller)  → ✓ valid, tool 0755
$ cpm verify … .cgp             (both)                         → ✓
```

*Report maintained by CognitiveOS-Labs. Changes via PR to `development`.*
