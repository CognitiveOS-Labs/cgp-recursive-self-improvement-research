# ADR-001: FP16/BF16 master checkpoint; LoRA never merged into GGUF

- Status: Accepted (2026-11-17)
- Context: Quantized GGUF weights accumulate error under repeated merge cycles; each GGUF round-trip compounds quantization noise with LoRA merge error, drifting the master beyond recovery.
- Decision: The canonical master at every generation t is FP16/BF16 `.safetensors`. Adapters are fused with `merge_and_unload()` in FP16; only the fused master is quantized to GGUF for deployment (Q8_0 or Q6_K). Merging LoRA into GGUF is prohibited.
- Consequences: ~2× storage per generation vs quantized (mitigated by Q8_0 deployment artifacts); merge verification step (`rsi/merge.py` checks dtypes); per-generation drift measurement required (`experiments/FAILURES.md` F7).
- Related: ADR-002, ADR-003
