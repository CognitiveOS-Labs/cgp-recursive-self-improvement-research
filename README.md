# Recursive Self-Improvement via .cgp Cognitive Patches

**Research repo — source of truth** for the project *Recursive Self-Improvement
via .cgp Cognitive Patches*. The **code representation** lives in
[`CognitiveOS-Labs/cgp-rsi`](https://github.com/CognitiveOS-Labs/cgp-rsi).

[![Build paper](https://github.com/CognitiveOS-Labs/cgp-recursive-self-improvement-research/actions/workflows/tex.yml/badge.svg)](https://github.com/CognitiveOS-Labs/cgp-recursive-self-improvement-research/actions/workflows/tex.yml)
[![License: MIT](https://img.shields.io/badge/code-MIT-blue.svg)](LICENSE)
[![Paper: CC-BY-4.0](https://img.shields.io/badge/paper-CC--BY--4.0-lightgrey.svg)](LICENSE_PAPER)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24432175.svg)](https://doi.org/10.5281/zenodo.24432175)

> **What "recursive self-improvement" means here.** An LLM packaged as a
> CognitiveOS `.cgp` (Cognitive Patch) artifact drives its own improvement loop:
> it generates training data, an **external deterministic verifier** filters it,
> a LoRA adapter (`ΔW_t`) is trained via `cpm tune`, fused into the FP16/BF16
> master weights, and promoted
> ($M_{t+1} = \mathrm{Merge}(M_t, \Delta W_t)$) **if and only if** a zero-trust
> promotion gate passes. Unverified synthetic data never enters training —
> this is the design barrier against model collapse.

---

## Project overview

CognitiveOS ships models and skills as `.cgp` archives managed by the
Cognitive Package Manager (`cpm`). This project asks whether that artifact —
not a bespoke training harness — can carry a closed-loop recursive
self-improvement (RSI) process end to end: packaging, fine-tuning, merging,
quantization, benchmarking, promotion, and rollback.

| | |
|---|---|
| **Research repo (this)** | Source of truth: research design, report, paper, protocol, decisions |
| **Code repo** — [`CognitiveOS-Labs/cgp-rsi`](https://github.com/CognitiveOS-Labs/cgp-rsi) | Executable pipeline, `.cgp` packages, CI, benchmarks, anchors |
| **Reference blueprint** | [`agentic-framework-research/drl-ai-orchestration`](https://github.com/agentic-framework-research/drl-ai-orchestration) (paper repo pattern) |
| **Technical report** | [`RSI_CGP_PROOF.md`](RSI_CGP_PROOF.md) |
| **Paper (built by CI)** | `paper/main.tex` → `main.pdf`; tagged releases publish it via `.github/workflows/release.yml` |
| **AHSE blueprint** | [`BLUEPRINT.md`](BLUEPRINT.md) |

## Research objectives

1. Design, implement, and empirically validate a closed-loop **Recursive
   Self-Improvement pipeline** using `.cgp` packages and `cpm`.
2. Demonstrate that an AI model packaged as a `.cgp` artifact can autonomously
   generate **verified** training signals and sustain measurable capability
   gains across **N ≥ 5 recursive generations** without model collapse or
   catastrophic forgetting (goal; execution-gated, see *Status*).
3. Prove recursive necessity with an ablation study:
   **(A)** recursive `.cgp` loop, **(B)** non-recursive single-shot LoRA,
   **(C)** unverified autophagous loop.
4. Publish the method as an arXiv-ready paper built reproducibly by CI.

## System architecture

```text
        ┌─────────────────────────── RSI loop (t = 0..N-1) ──────────────────────────┐
        │                                                                            │
  M_t ──▶ GENERATE ──▶ VERIFY ──▶ MIX (20% anchors) ──▶ cpm tune ──▶ MERGE(FP16) ──▶ GGUF
        (LLM output,   (AST +      (replay buffer,     (LoRA ΔW_t)   (PEFT)      (llama.cpp)
         seed-fixed)    sandbox)     seeded, fixed)                                    │
                                                                                       ▼
        ┌─────────── base-model-v{t+1}.cgp  ◀── PROMOTE ◀── GATE ◀── EVALUATE B_target/B_general
        │                                        │
        └─────────── rollback via cpm  ◀─────────┘  (fail ⇒ rollback + log + hyperparameter adjust)
```

**Three invariants, enforced in code (not convention):**

1. **Precision policy.** The master checkpoint is always FP16/BF16
   `.safetensors`. LoRA is fused in FP16; **only** the fused master is
   quantized. LoRA is never merged into GGUF (quantization drift
   compounding). — `ADR-001`
2. **Verification policy.** 100% of synthetic training samples pass an
   **external deterministic verifier** (AST allowlist + sandboxed subprocess +
   hidden unit tests). Zero unverified samples enter `cpm tune`. — `ADR-002`
3. **Promotion policy.** `M_{t+1}` replaces `M_t` **iff**
   `Score(M_{t+1}, B_target) > Score(M_t, B_target) + ε`  (ε = 0.02) **and**
   `Score(M_{t+1}, B_general) ≥ Score(M_t, B_general) − δ`  (δ = 0.01).
   Failure ⇒ automatic rollback via `cpm`, logged failure mode, bounded
   hyperparameter adjustment, retry. — `ADR-003`

## Execution steps (how to run / reproduce)

The full protocol lives in [`experiments/PROTOCOL.md`](experiments/PROTOCOL.md).
Quickstart for the machinery (no GPU required):

```sh
git clone https://github.com/CognitiveOS-Labs/cgp-rsi
cd cgp-rsi
python3 -m venv .venv && . .venv/bin/activate
pip install -e '.[dev]'
pytest                       # 20 unit tests
python3 scripts/mock_e2e.py  # full state machine, MOCK artifacts
```

Multi-generation run (GPU host required): see [`experiments/PROTOCOL.md`](experiments/PROTOCOL.md)
and the code repo's `docs/REPRODUCIBILITY.md`.

## Required deliverables

| # | Deliverable | Location | Status |
|---|---|---|---|
| 1 | Working `.cgp` recursive pipeline (scripts, manifests, merge utilities) | [`cgp-rsi`](https://github.com/CognitiveOS-Labs/cgp-rsi) | ✅ implemented, `cpm pack`/`cpm verify` ✓, 20/20 unit tests ✓, mock e2e ✓ |
| 2 | Quantitative proof table & learning curves, M₀→M₆ vs baselines | this repo, `experiments/RESULTS.md` | ⏳ **pending GPU run** (no numbers claimed) |
| 3 | Failure & safety analysis (rejected generations, quantization error, rollback logs) | [`RSI_CGP_PROOF.md`](RSI_CGP_PROOF.md) §6, `experiments/FAILURES.md` | ✅ design + mechanism; empirical entries pending GPU run |
| 4 | Reproducible technical report | [`RSI_CGP_PROOF.md`](RSI_CGP_PROOF.md) | ✅ written (honest status) |
| 5 | arXiv-ready paper, CI-built PDF, Zenodo DOI on release | [`paper/`](paper/) | ✅ source + CI; archived on Zenodo — DOI [10.5281/zenodo.24432175](https://doi.org/10.5281/zenodo.24432175) |
| 6 | Versioned `.cgp` artifacts | [`cgp-rsi/packages/`](https://github.com/CognitiveOS-Labs/cgp-rsi/tree/main/packages) + GitHub Releases | ✅ `base-model-v0`, `rsi-controller` (v0.1.0) |

## Constraints

- All code MUST be compatible with CognitiveOS and `cpm >= v1.0.9-alpha`.
- All experiments MUST be reproducible and documented here; raw logs
  (`runs/metrics.jsonl`, `runs/quantization-drift.json`) are copied verbatim.
- All generated `.cgp` artifacts are versioned and stored in
  [`cgp-rsi`](https://github.com/CognitiveOS-Labs/cgp-rsi) (in-repo + releases).
- No fabricated measurements: unrun experiments are marked ⏳ and claims are
  limited to what was actually executed and logged. — `ADR-005`
- Documentation follows this SPECv1-style README; paper follows the
  reference paper-repo layout (LaTeX + BibTeX, CI-built PDF).

## Status & honest scope

> **Framework-only releases (v0.1.x).** Framework complete; empirics pending.
> These releases are framework releases — not evidence of recursive
> self-improvement. Archived on Zenodo:
> [10.5281/zenodo.24432175](https://doi.org/10.5281/zenodo.24432175).
> See [MISSION.md](MISSION.md) (Lap 3 definition), [HUD.md](HUD.md) (three
> signals + triggers), and [AUDIT-REPORT.md](AUDIT-REPORT.md) (SPECv1 audit).

This repository was produced in an environment **without a GPU and without a
training toolchain** (no `torch` wheels for the system Python 3.14, no base
model weights, `cpm` installed during this work). What exists and is verified:
the complete pipeline implementation, its unit tests, the CI mock end-to-end
run (promote and rollback paths), and the `.cgp` packaging of both patches.
What does **not** exist yet: empirical M₀→M₆ measurements. Those require the
GPU-host run defined in `experiments/PROTOCOL.md`; when executed, results are
logged verbatim into `experiments/RESULTS.md` and the paper's results section.

## References for further reading

- CognitiveOS Product Specs — <https://github.com/CognitiveOS-Project/product-specs>
  (`specs/cgp-format.md`, `specs/manifest-fields.md`, `specs/tutorials/cpm-tune.md`,
  `specs/tutorials/sandboxed-cgp-development.md`, `specs/tutorials/download-weights.md`)
- CPM — <https://github.com/CognitiveOS-Project/cpm>
- `.cgp` ecosystem: [`CognitiveOS-Labs/fingerprint-gt511c3`](https://github.com/CognitiveOS-Labs/fingerprint-gt511c3),
  [`CognitiveOS-Labs/face-recognition-tune`](https://github.com/CognitiveOS-Labs/face-recognition-tune),
  [`CognitiveOS-Labs/face-recognition`](https://github.com/CognitiveOS-Labs/face-recognition)
- Hu et al., *LoRA: Low-Rank Adaptation of Large Language Models*, 2021. arXiv:2106.09685
- Dettmers et al., *QLoRA: Efficient Finetuning of Quantized LLMs*, 2023. arXiv:2305.14314
- Shumailov et al., *AI models collapse when trained on recursively generated data*, *Nature* 631, 755–759, 2024.
- Wang et al., *Self-Instruct: Aligning Language Models with Self-Generated Instructions*, 2022. arXiv:2212.10560
- Zelikman et al., *STaR: Bootstrapping Reasoning With Reasoning*, 2022. arXiv:2203.14465
- Ouyang et al., *Training language models to follow instructions with human feedback*, 2022. arXiv:2203.02155
- Wei et al., *Training Verifiers to Solve Math Word Problems*, 2022. arXiv:2110.14168

## How to contribute

See [CONTRIBUTING.md](CONTRIBUTING.md). In short: fork, topic branch from
`development` (`feature/…`), PR into `development`; `cpm pack && cpm verify`
MUST pass for any `.cgp` change; never commit directly to `main` or
`development`. Bug reports: <https://github.com/CognitiveOS-Labs/cgp-recursive-self-improvement-research/issues>.
