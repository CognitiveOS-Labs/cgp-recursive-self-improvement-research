# Experimental Protocol (frozen 2026-11-17)

This document is the pre-registered protocol for the multi-generation
experiment. Changes to it MUST go through PR to `development` and MUST be
versioned (this version: protocol v1.0.0).

## 1. Claim under test

A `.cgp`-packaged LLM can recursively self-improve on a verifiable domain
across N ≥ 5 (here: 6) generations without model collapse, holding
general-capability retention within δ.

**Primary hypothesis (H5).** In promoted generations, B_target accuracy
strictly increases by > ε over the previous generation (by gate construction)
**and** the final promoted model M₆ outperforms M₀ by ≥ 5 percentage points
on B_target, with B_general retention ≥ −δ.

**Secondary hypotheses.**
- H6a: Arm C (verifier bypassed) degrades on B_target and/or collapses
  (pass rate → 0, repetitive outputs), demonstrating verifier necessity.
- H6b: Arm B (non-recursive single-shot, equal total rank-steps) underperforms
  Arm A on B_target at equal compute, demonstrating recursive necessity.
- H7: FP16-master merging yields quantization drift < 2 points B_target
  (Q8_0 export), measured per generation.

**Falsifiers.** H5 is falsified if the gate never promotes (no recursion),
or if promoted models regress on B_target by > ε, or B_general drops below
−δ. H6 is falsified if arm C ≈ arm A and arm B ≈ arm A. Either outcome is a
valid scientific result and MUST be reported as such (see `RSI_CGP_PROOF.md`
§7).

## 2. Fixed configuration

| Knob | Value |
|---|---|
| Base model | `base-model-v0.cgp` — wide_model `google/gemma-3-1b-it`, remote safetensors (FP16/BF16 master) |
| Domain | Python algorithmic synthesis (unique I/O, execution-checkable) |
| Generations | 6 cycles (t = 0..5) → M₀ baseline, M₁..M₆ artifacts |
| Samples per generation | 64 candidate problems (as budget allows; log actuals) |
| LoRA | rank 16, α 32, epochs 2, lr 2e-4, min_samples 50; retry on gate failure: lr/2 (both-fail also rank/2, floor 4) |
| Replay anchors | 16 verifier-passing problems, 20% mix, seeded |
| Seeds | `RSI_SEED=1337` (primary), `2026`, `42` (error bars on headline) |
| Gate | ε = 0.02 (strict), δ = 0.01; strict-greater semantics |
| Benchmarks | `b_target.jsonl` (8 held-out), `b_general.jsonl` (8 held-out) — read-only, never in prompts/mixes |
| Verifier | AST allowlist + sandboxed subprocess (`RLIMIT_CPU=5`, `RLIMIT_AS=256MB`, 6 s wall) |
| Quantization | FP16 master → Q8_0 GGUF per generation; drift measured (top-1 agreement, KL) |
| Latency | inference ms per benchmark item logged |

## 3. Procedure

0. **Provisioning** — GPU host; `cpm install base-model-v0.cgp`;
   `cpm install rsi-controller.cgp`; inference endpoint FP16, ≥16k ctx.
1. **Baseline** — measure B_target/B_general of M₀ @ temperature 0, fixed
   seed. Record in `runs/baseline.json`.
2. **Per generation t** — run `python3 scripts/run_generation.py
   --generations 1 --out runs/gen-<t>` (or the full loop in one invocation):
   generate → verify → mix → `cpm tune base-model-v0` → merge (FP16) →
   quantize → evaluate → gate.
   - Pass: fuse, export, package `base-model-v{t+1}.cgp` (CI release), re-point
     endpoint, continue. Log to `runs/metrics.jsonl`.
   - Fail: automatic rollback via `cpm install <previous>.cgp`; log failure
     mode; apply hyperparameter adjustment; retry once at reduced lr; if the
     retry fails the gate, **stop the arm** and record the failure.
3. **Ablations** — Arm A (above). Arm B: single LoRA over M₀, rank = 16 ×
   total-steps/… (equal compute: same total optimized LoRA steps as arm A
   would spend across all 6 generations; only M₀-generated, verified data;
   one training run; one evaluation). Arm C: as A, verifier verdicts ignored
   (all samples admitted), gate unchanged.
4. **Quantization drift** — after each merge, export Q8_0 and measure
   top-1 agreement / KL on 64 fixed B_target prompts vs FP16 master; save
   `runs/quantization-drift.json`.
5. **Replication** — headline metrics for arms A/B/C under seeds
   {1337, 2026, 42} if budget allows; else report seed 1337 with explicit
   limitation.

## 4. Logging & reporting (verbatim)

- `runs/metrics.jsonl` — one JSON object per generation per arm.
- `runs/quantization-drift.json` — per generation.
- `runs/baseline.json`, `runs/promotions/` — gate decisions + rollback logs.
- Copy all into this repo `experiments/logs/<run-id>/` in a PR titled
  `results: run <run-id>`; link the PR in `experiments/RESULTS.md`.
- Paper tables/figures regenerate from those logs only (no hand transcription
  beyond table formatting; numbers by script, e.g. the `render_table` helper).

## 5. Threats to validity

- Benchmark size (8/8) — small; absolute numbers noisy; report n and CI.
- Test-suite-as-spec — verifier proves execution agreement, not general
  correctness; scale tests for headline claims.
- GPU non-determinism — multi-seed reporting.
- Domain locality — Python algorithms; no claim beyond it.
- Verifier allowlist strictness — MAY reject correct-but-unusual solutions;
  counted and reported (verifier-rejection rate).
- Promotion gate asymmetry — favors retention; a too-strict gate stalls
  recursion; interpret "no promotions" accordingly (report gate internals).
