# Results

**Status as of 2026-11-17: empirical multi-generation results do not exist yet.**
The authoring environment had no GPU, no training stack, and no base-model
weights. This file exists to receive verbatim logs from the GPU run defined in
[`PROTOCOL.md`](PROTOCOL.md). It MUST NOT contain numbers that did not come
from an executed run.

## What IS measured (2026-11-17, authoring environment)

| Item | Result | Command |
|---|---|---|
| Unit tests (verifier, gate, replay, controller, metrics-render) | 20/20 pass | `pytest` in `cgp-rsi` |
| Mock e2e — happy path (2 gen) | 2/2 promoted; metrics JSONL written | `python3 scripts/mock_e2e.py` |
| Mock e2e — rollback path | gen 1 rejected; rollback invoked; failure_mode logged | `python3 scripts/mock_e2e.py --fail-at 1` |
| Anchor replay set | 16/16 pass the deterministic verifier | see `cgp-rsi` CI `package` job |
| Benchmark consistency | 8/8 B_target problems solvable (reference solutions, via verifier) | ad-hoc, logged in PR |
| `cpm pack` / `cpm verify` | both packages valid | `packages/*/` in `cgp-rsi` |

None of the above are model-capability measurements. They validate the
machinery and the contract of the gate/verifier.

## Pre-registered results table (to be filled from logs)

| Gen | pass% | loss | ‖ΔW‖ | B_target | Δ | B_general | Δ | verdict | rollback |
|---|---|---|---|---|---|---|---|---|---|
| 0 (M₀) | — | — | — | TBD | — | TBD | — | baseline | — |
| 1 | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD |
| 2 | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD |
| 3 | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD |
| 4 | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD |
| 5 | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD |
| 6 | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD | TBD |

## Ablation comparison (to be filled from logs)

| Arm | M₀ B_target | M₆ B_target | Δ B_target | M₆ B_general | Δ B_general | Total steps | Notes |
|---|---|---|---|---|---|---|---|
| A (recursive) | TBD | TBD | TBD | TBD | TBD | TBD | |
| B (single-shot) | TBD | TBD | TBD | TBD | TBD | equal | |
| C (unverified) | TBD | TBD | TBD | TBD | TBD | TBD | expect collapse |

## Quantization drift (to be filled from `runs/quantization-drift.json`)

| Gen | FP16→Q8_0 top-1 agreement | KL(FP16‖GGUF) | B_target FP16 | B_target GGUF | pass? |
|---|---|---|---|---|---|
| 1..6 | TBD | TBD | TBD | TBD | TBD |

## How to fill this file

1. Execute the GPU run (see `PROTOCOL.md` §3).
2. Open a PR titled `results: run <run-id>` copying `runs/` logs verbatim into
   `experiments/logs/<run-id>/`.
3. A small script (committed with the PR) renders the tables above from the
   logs; paste/refresh between rows only from its output.
4. Update `RSI_CGP_PROOF.md` §5.2 and the paper's Results section with the
   same numbers; open a discussion thread referencing the run's PR.
