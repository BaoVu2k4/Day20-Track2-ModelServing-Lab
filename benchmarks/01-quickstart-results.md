# 01 — Quickstart Results

Settings: `n_threads=6`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=0`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 2190 | 257 / 351 | 65.7 / 122.7 | 4398 / 8027 / 8419 | 15.2 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 614 | 324 / 490 | 58.3 / 76.1 | 4013 / 5244 / 5841 | 17.1 |

## Observations

- Q2_K decoded at 17.1 tok/s versus 15.2 tok/s for Q4_K_M, a 12.5% speedup on this laptop. Its E2E P95 was also lower (5,244 ms versus 8,027 ms).
- Q4_K_M had lower TTFT P50 (257 ms versus 324 ms), so the smaller quantization is not automatically faster in every phase; model layout, cache behavior, and the short prompt distribution matter.
- Q4_K_M remains the primary serving model because its higher precision is preferable when the modest decode-rate cost fits the latency budget.
- All measurements used the six physical cores rather than the twelve logical threads, since decode is primarily constrained by memory bandwidth.
