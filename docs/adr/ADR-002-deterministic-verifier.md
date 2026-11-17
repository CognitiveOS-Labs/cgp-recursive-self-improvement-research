# ADR-002: 100% of synthetic samples pass an external deterministic verifier

- Status: Accepted (2026-11-17)
- Context: Recursively training on unverified model output causes model collapse (Shumailov et al., Nature 2024). LLM-as-judge reintroduces the stochastic failure mode at the filter itself.
- Decision: Every synthetic (problem, solution) pair admitted to training MUST pass `rsi/verifier.py`: AST allowlist, banned-call/dunder rejection, sandboxed subprocess execution with rlimits and wall-clock timeout, and the problem's full hidden unit-test suite. No judge, no heuristic, no exception. Samples failing anywhere are discarded (fail-closed).
- Consequences: Domain MUST be execution-checkable (Python algorithmic synthesis chosen); verifier rejects some correct-but-unusual solutions (measured as pass-rate, reported per generation); benchmark suites MUST stay hidden from synthesis prompts.
- Related: ADR-001, ADR-003
