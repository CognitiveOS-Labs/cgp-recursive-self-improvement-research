# ADR-004: Two repositories — research (source of truth) and code representation

- Status: Accepted (2026-11-17)
- Context: Paper/report and executable code evolve on different cadences and have different licensing (CC-BY-4.0 vs MIT) and CI shapes (TeX pipeline vs pytest/cpm pipeline). The reference pattern (drl-ai-orchestration ↔ OSMv6) separates them the same way.
- Decision: `cgp-recursive-self-improvement-research` holds design, report, paper, protocol, results. `cgp-rsi` holds the pipeline, packages, CI, benchmarks, logs. READMEs cross-link each other; the report cites the code repo at a tag; the code repo references the report; artifact releases happen in the code repo, paper releases in this one.
- Consequences: Duplication of the architecture description (kept consistent via links, not copies); tags aligned at release (`v*.*.*` in both); Zenodo archive per paper release.
- Related: ADR-005
