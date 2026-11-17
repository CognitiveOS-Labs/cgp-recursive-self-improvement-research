# Failure & Safety Analysis

This document enumerates how the pipeline fails, how each failure is caught,
and what the operator observes. Empirical entries (from the GPU run) are
added after `experiments/logs/<run-id>` lands; the mechanisms are implemented
and unit-tested today.

## Failure taxonomy & response

| # | Failure | Detection | Response (implemented) | Operator-visible signal |
|---|---|---|---|---|
| F1 | Gate rejection — insufficient B_target gain | `evaluate_gate` strict ε | rollback via `cpm install <prev>.cgp`; lr←lr/2; retry once | `metrics.jsonl: promoted=false, failure_mode="target margin …"` |
| F2 | Gate rejection — B_general regression > δ | same | rollback; lr←lr/2 ∧ data_temperature←T/2; retry | `failure_mode="general regression …"` |
| F3 | Gate rejection — both | same | rollback; lr←lr/2 ∧ rank←rank/2 (floor 4) | `failure_mode="…; …"` |
| F4 | Invalid/adversarial synthetic sample | AST allowlist / sandbox / hidden tests | sample discarded pre-training; counted in pass-rate | `pass%` column drops; verifier-rejection rate |
| F5 | Verifier crash / timeout | runner subprocess exit/timeout | fail-closed: sample discarded | `error` in per-item verdict log |
| F6 | Training divergence (NaN loss / exploding adapter norm) | tuner returns non-finite loss/norm; gate cannot pass without B_target gain | gate rejects; rollback | `train_loss` / `lora_weight_norm` fields; `promoted=false` |
| F7 | Quantization drift (FP16→GGUF) exceeds noise | drift measurement (top-1, KL) per export | generation not promoted; investigate export settings | `runs/quantization-drift.json`; failing gate on quantized eval |
| F8 | Catastrophic forgetting | anchor-coverage < 20% or B_general < −δ | abort generation; gate blocks promotion | B_general delta; anchor coverage check in controller |
| F9 | Endpoint unreachable / malformed model output | client raises / `extract_json` returns None | candidates skipped; if pass-rate collapses to 0, generation yields no training data and the gate has nothing to evaluate → no promotion | endpoint error on stderr of run |
| F10 | Rollback itself fails (cpm install error) | rollback_via_cpm returns non-zero/empty | **fail loudly** — operator intervention required; previous artifact remains current because promotion never happened | stderr; absence of `promoted=true` |

## Rollback semantics

Promotion is an artifact-level act (`cpm install base-model-v{t+1}.cgp`), not
a weight-level mutation of a running server. Therefore rollback =
`cpm install base-model-v0<t>.cgp` of the last promoted artifact — atomic at
the package boundary, with the daemon/inference engine re-loading the
artifact. Precedence rule: **the gate is the only path to promotion**;
`RsiController.run_generation` has no other promote branch.

## Hyperparameter adjustment mapping (as implemented)

```text
target-miss only        → lr ← lr/2
general-regression only → lr ← lr/2, data_temperature ← T/2  (≈ sharper replay reliance)
both                    → lr ← lr/2, lora_rank ← rank/2 (floor 4)
```

Both-fail cap: after one retry at reduced budget fails the gate, the arm is
stopped and the failure recorded (no infinite retry).

## Quantization error measurement plan

Per generation, after FP16 merge: export Q8_0, then on 64 fixed B_target
prompts compute (i) top-1 agreement rate, (ii) mean KL(FP16‖GGUF) over next
-token distributions. Store raw measurements in `runs/quantization-drift.json`.
Acceptance criterion: top-1 agreement ≥ 99% **and** B_target(GGUF) within
0.5 point of B_target(FP16); else the generation does not promote. Tune
`--outtype` if violated (Q8_0 → Q6_K) and document.

## Safety argument summary

- No unverified data trains the model (F4/F5 fail closed).
- No promotion without an isolated two-benchmark win (F1–F3).
- No silent quantization damage (F7).
- No silent forgetting (F8 + anchor coverage check).
- Rollback is always available and is the only success/failure exit for a
  generation (plus abort-stop on retry failure) (F10).

## Open safety questions

- Does verifier-gated self-training shift failure modes *outside* the tested
  distribution of "passing" programs (Goodharting the test suite)? Mitigation
  candidates: hidden-test rotation, AST shape constraints, held-out suites.
- Registry-level trust for `.cgp` promotions across machines (signatures) —
  out of scope here; see product-specs `specs/security-model.md`.
