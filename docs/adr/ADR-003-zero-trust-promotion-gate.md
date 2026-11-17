# ADR-003: Zero-trust promotion gate with automatic rollback

- Status: Accepted (2026-11-17)
- Context: Recursive promotion must be decidable without trusting the artifact being promoted. Self-evaluation is not evidence.
- Decision: Promotion iff `s(M_{t+1}, B_target) > s(M_t, B_target) + 0.02` AND `s(M_{t+1}, B_general) >= s(M_t, B_general) − 0.01`. Strict-greater semantics with `math.isclose(abs_tol=1e-9)` for float noise. Failure ⇒ automatic `cpm install <previous>.cgp` rollback, failure mode logged, bounded hyperparameter adjustment, one retry, then stop the arm.
- Consequences: A generation is never a commitment; hyperparameter search is guided by failure mode (target-miss → lr/2; general-regression → lr/2 ∧ T/2; both → lr/2 ∧ rank/2); the gate is the sole promotion path in `RsiController`.
- Related: ADR-001, ADR-002
