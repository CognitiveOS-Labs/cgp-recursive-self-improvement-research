# Mission Script — RSI via .cgp Cognitive Patches (Lap 3)

> Frequency: **Stealth/Precision** — correctness over speed; this mission touches
> research claims (the currency of trust). The machine must refuse rather than
> guess. Any task on this repo inherits this Frequency unless its own script
> overrides it explicitly.

## Mission Name

RSI .cgp — Lap 3: verified GPU-run results or an honest downgrade.

## Outcome (definition of done — binary, checkable)

One of two outcomes closes Lap 3:

- **(a) Verified run**: a GPU run of `t = 0..5` completes; `runs/metrics.jsonl`,
  `runs/quantization-drift.json`, and gate/rollback logs are merged via PR;
  `experiments/RESULTS.md` and the paper's §5.2 + tables are filled
  **verbatim** from those logs; release `v0.2.0` carries the rebuilt paper.
- **(b) Honest park**: a "parked pending compute" decision is merged to `main`;
  release/claims language is downgraded (framework-only pre-release), and
  capability language is stripped from the README and paper.

No third path: silent drift or marketing language is failure.

## Constraints

- No fabricated numbers. Every claim MUST trace to `experiments/logs/<run-id>/`.
- Benchmarks and anchors are frozen (`b_target.jsonl`, `b_general.jsonl`,
  anchors at 20%, ε = 0.02, δ = 0.01, `RSI_SEED=1337`).
- `cpm pack` / `cpm verify` MUST stay green in CI.
- Topic branches only (`feature/`, `fix/`, …); PRs into `development`;
  promotion to `main` via release PR. No direct pushes to `main`/`development`.
- `cpm >= v1.0.9-alpha`; Python ≥ 3.10; GPU stages only on the GPU host.

## Acceptance

A reviewer regenerates every number in `RSI_CGP_PROOF.md` §5.2 and the paper
tables from `experiments/logs/<run-id>/` alone. 20/20 unit tests pass. CI is
green, including the encapsulated Alpine cpm job.

## Verification

1. `pytest -q` (20/20).
2. `cpm verify` on both `.cgp` archives.
3. Gate decisions in `metrics.jsonl` match merged promote/rollback calls.
4. Paper rebuilt from logs only; no hand-transcribed numbers.
5. Tag `v0.2.0` → release workflow attaches PDF.
