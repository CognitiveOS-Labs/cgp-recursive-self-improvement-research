# ADR-005: No fabricated results — CI mock runs and honest status reporting

- Status: Accepted (2026-11-17)
- Context: The authoring environment has no GPU, no torch/peft/transformers for its CPython 3.14, and no base-model weights. Claiming multi-generation measurements would be fabrication — both a scientific and an ethical failure.
- Decision: (i) Every capability claim in report/paper is labelled by provenance: "executed 2026-11-17" (pipeline validation) vs "pending GPU run" (M₀→M₆). (ii) CI runs the full state machine against a deterministic mock model, explicitly labelled MOCK; it proves mechanics, not capability. (iii) `experiments/RESULTS.md` and the paper's results section hold only values imported from executed-run logs. (iv) Pre-registered hypotheses H5–H7 are falsifiable and reported either way.
- Consequences: Slower publication of the headline claims; trust preserved; reviewers can reproduce every §5.1 row; the $5.2-pending$ label makes the remaining work a concrete, fundable task.
- Related: ADR-004
