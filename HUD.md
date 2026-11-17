# HUD — RSI via .cgp Cognitive Patches

Heads-Up Display definition and current snapshot. Vibe Leading framework:
three signals, live or it does not exist. The full methodology lives in the
`hud-setup` skill; this file is the project's concrete instantiation.

## The three signals

### 1. Traction — Intent-Result Alignment (IRA)

    IRA = (deliverables landed) / (deliverables specified in the Mission Script)

- ≥ 95% — green; hold the grip, keep the pace.
- ~70–80% — yellow; investigate the Mission Script, not the machine.
- < 60% — red; pull the machine back to the Vibe before merging anything.

### 2. Circuit Delta — Lap Time change vs previous Lap

    Δ = T(Lap n+1) − T(Lap n)

- Negative — winning. Baseline 15 min (Lap 2). Target for Lap 3: ≤ 12 min.
- Positive — stalled; inspect friction before adding scope.

### 3. Hallucination Debt — unverified-claim accumulation

    Debt = count of material claims in release/paper/README without a logs trace

- ≥ 1 requires explicit flagging; ≥ 4 trips the red band: freeze merges
  until debt is cleared.

## Signals & bands (this project)

| Signal | Green | Yellow | Red | Current (2026-11-17) |
|---|---|---|---|---|
| IRA | ≥ 95% | 70–80% | < 60% | **~80% (yellow)** — framework/CI/packages/releases green; quantified M₀→M₆ proof table pending GPU (outcome (a)/(b) of Lap 3 decides) |
| Circuit Delta | ≤ 12 min/Lap | ≤ baseline 15 min | growing | **−50% vs Lap 1** (Lap 1 ≈ 30 min → Lap 2 ≈ 15 min) |
| Hallucination Debt | 0 | 1–3 | ≥ 4 | **2 items** — (1) BLUEPRINT STAGE-999 overclaim (fixed in this branch); (2) v0.1.0 release lacked pre-release marking (fixed — releases set to pre-release) |

## Escalation triggers

- IRA < 95% at any merge → Risk Spotter blocks the PR until the gap is
  explained or fixed.
- Debt ≥ 4 → freeze all releases; only debt-clearing PRs merge.
- Circuit Delta > 0 for two consecutive Laps → scope review with the Pilot.

## Observer Access (View-Only)

The HUD is the replacement for status meetings. To be granted in Lap 3:

- Any stakeholder who used to attend this project's standup gets a **read-only**
  view of this file, the HUD snapshot in the paper trail, and the release
  list. No write access. No edit mandate.
- Practical channel: GitHub repository (read) + `gh release list` +
  `experiments/RESULTS.md` on `development`. No separate dashboard exists yet;
  one is unnecessary at this Lap count.

## How to refresh the snapshot

1. Re-score IRA against `MISSION.md` (count landed vs specified; partial = 0.5).
2. Compute Lap time from commit timestamps: `git log --format='%ci %s'`.
3. Re-count Hallucination Debt: grep release/paper/README for claims; every
   claim MUST cite `experiments/logs/<run-id>/` or `§5.1`-style executed
   checks. Update the table above via PR.
4. Log the refresh as a `pit-stop` entry in the paper trail.
