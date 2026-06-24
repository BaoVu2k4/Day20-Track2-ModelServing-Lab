# 02 — Server Load-Test Results

Server: `python -m llama_cpp.server` with Qwen2.5-1.5B-Instruct Q4_K_M, CPU-only, `n_threads=6`, `n_ctx=2048`, port 8080.

| Concurrency | Duration | Completed requests | Failures | Total RPS | E2E P50 (ms) | E2E P95 (ms) | E2E P99 (ms) |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 10 | 60 s | 11 | 0 | 0.21 | 25,000 | 42,000 | 42,000 |
| 50 | 60 s | 12 | 0 | 0.22 | 27,000 | 43,000 | 43,000 |

Notes:

- Locust exercised the OpenAI-compatible `/v1/chat/completions` endpoint with 80% short and 20% long-RAG prompts.
- The Python `llama_cpp.server` implementation returned `404` for `/metrics`; native `llama-server --metrics` was not built for this run, so native batching gauges are unavailable.
- Increasing offered concurrency from 10 to 50 did not increase completed RPS on this CPU-only setup. It increased queueing latency instead, while all completed requests succeeded.
