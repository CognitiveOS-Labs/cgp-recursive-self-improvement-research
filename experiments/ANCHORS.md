# Anchor replay buffer & benchmark provenance

## Anchors (`cgp-rsi/data/anchors.jsonl`)

- **Purpose**: fixed replay buffer mixed at 20% into every training mix to
  guard against catastrophic forgetting.
- **Size**: 16 problems (an-001…an-016), authored for this project.
- **Verification**: every anchor's solution passes the deterministic verifier
  (checked in CI: `.github/workflows/ci.yml`, `package` job) — anchors are
  *verified* training data, not merely trusted data.
- **Domain match**: Python algorithmic synthesis — same domain as the target,
  disjoint problem instances from `b_target.jsonl`.
- **Determinism**: sampling into mixes is seeded (`RSI_SEED`); file order
  takes precedence in `mix_with_replay`.

| ID | Function | Tests |
|---|---|---|
| an-001 | `sum_list` | sum of list |
| an-002 | `factorial` | n! for n ≥ 0 |
| an-003 | `is_prime` | primality |
| an-004 | `reverse_words` | word order reversal |
| an-005 | `count_occurrences` | element frequency |
| an-006 | `max_subarray_sum` | Kadane's |
| an-007 | `char_frequency` | char counts → dict |
| an-008 | `flatten` | one-level flatten |
| an-009 | `celsius_to_fahrenheit` | unit conversion |
| an-010 | `is_anagram` | anagram check |
| an-011 | `digits_sum` | decimal digit sum |
| an-012 | `capitalize_words` | per-word capitalize |
| an-013 | `binary_search` | sorted search |
| an-014 | `merge_sorted` | merge two sorted lists |
| an-015 | `double_evens` | even numbers ×2 |
| an-016 | `unique_sorted` | dedupe + sort |

## Benchmarks (`cgp-rsi/benchmarks/`)

### `b_target.jsonl` — B_target (held-out, 8 items)
`fib`, `count_vowels`, `second_largest`, `gcd`, `roman_to_int`,
`longest_run`, `is_palindrome_clean`, `sum_of_squares_evens`.
Held out: disjoint from anchors and from any synthesis prompt. Validated by
executing reference solutions through the verifier (8/8 pass). Problems were
authored for this project; none are derived from `b_general.jsonl`.

### `b_general.jsonl` — B_general (held-out, 8 items)
Short instruction-following tasks (single-word factual answers, arithmetic,
translation, echo) with `expected_substring` scoring at temperature 0.
These are weak proxies for general capability — stated limitation, see
`PROTOCOL.md` §5 and `RSI_CGP_PROOF.md` §7.

## Integrity policy

- Benchmarks and anchors are **read-only**; edits require a PR into
  `development` and a note in this file.
- `b_target.jsonl` MUST NOT be used in synthesis prompts, verification
  training mixes, or hyperparameter tuning — it is consumed by the gate only.
- If any benchmark item is found ambiguous or broken, it is replaced in the
  *next* run only (benchmarks are frozen per run), never retroactively.
