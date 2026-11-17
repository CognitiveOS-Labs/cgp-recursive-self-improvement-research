# Contributing

Thanks for your interest in this research. This repo is the **source of truth**
for the project; code lives in [`cgp-rsi`](https://github.com/CognitiveOS-Labs/cgp-rsi).

## Ground rules

- **Never commit directly to `main` or `development`.** Branch from
  `development`: `feature/<topic>` (or `fix/…`, `docs/…`, …), then PR into
  `development`. Promotion to `main` happens via a release PR.
- **No fabricated results.** Numbers in docs/paper MUST trace to
  `experiments/logs/<run-id>/` or to §5.1-style executed checks. See ADR-005.
- **Benchmarks are frozen.** `b_target.jsonl` / `b_general.jsonl` change only
  for a *next* run, never retroactively, and only via PR with a note in
  `experiments/ANCHORS.md`.
- **`cpm pack` purity.** Any `.cgp` manifest change must pass `cpm pack` and
  `cpm verify` in CI.
- **Docs style.** README files keep the SPECv1 section set (overview,
  objectives, architecture, steps, deliverables, constraints, references).
  ADRs follow the existing format (Context/Decision/Consequences).

## Workflow

1. Fork → clone → `git fetch origin && git checkout -b feature/<topic> development`.
2. Make your change; keep commits atomic with clear messages.
3. `gh pr create --base development` with: what/why, links to issues, test
   evidence (or protocol deviation notes).
4. CI must be green (paper builds on `tex.yml`; code tests run in `cgp-rsi`).
5. Review focuses: correctness of protocol, reproducibility, honesty of
   claims — not style.

## Reporting issues

Use the issue tracker with:

- environment (GPU, cpm version, OS),
- command + full logs (redact secrets — never paste `RSI_API_KEY`),
- expected vs observed behavior,
- whether it reproduces on the mock path (`python3 scripts/mock_e2e.py`).

## Suggesting improvements

Open a discussion first for protocol changes (new arms, new benchmarks, gate
changes). They require amending `experiments/PROTOCOL.md` (a PR titled
`protocol: …`) and a versioned decision note in `docs/adr/`.
