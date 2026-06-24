# Reflection — Lab 20 (Personal Report)

**Họ tên:** Vũ Quang Bảo
**Mã học viên :** 2A202600610
**Cohort:** Chưa cung cấp — điền trước khi submit  
**Ngày submit:** 2026-06-24

## 1. Hardware spec

- **OS:** Windows 11 (AMD64)
- **CPU:** AMD Ryzen 5 5500U with Radeon Graphics
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, F16C, FMA (không có AVX-512)
- **RAM:** 15.4 GB usable
- **Accelerator:** CPU only; không có GPU rời
- **llama.cpp backend:** CPU
- **Recommended model tier:** Qwen2.5-1.5B-Instruct Q4_K_M

**Setup story:** Máy không có GPU rời nên tôi dùng backend CPU, giữ `LAB_N_GPU_LAYERS=0`, `LAB_N_CTX=2048`, `LAB_N_BATCH=512` và dùng 6 physical cores thay vì 12 logical threads. Wheel `llama-cpp-python` mới nhất đã bật AVX-512 và gây lỗi illegal instruction trên Ryzen 5500U; tôi dùng wheel CPU 0.3.19, tương thích AVX2/FMA.

## 2. Track 01 — Quickstart numbers

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| Qwen2.5-1.5B Q4_K_M | 2,190 | 257 / 351 | 65.7 / 122.7 | 4,398 / 8,027 / 8,419 | 15.2 |
| Qwen2.5-1.5B Q2_K | 614 | 324 / 490 | 58.3 / 76.1 | 4,013 / 5,244 / 5,841 | 17.1 |

**Quan sát:** Q2_K tăng decode rate từ 15.2 lên 17.1 tok/s (1.13×) và có E2E P95 thấp hơn, nhưng Q4_K_M lại có TTFT P50 thấp hơn. Quantization nhỏ hơn không tự động nhanh ở mọi phase: prefill và decode có nút thắt cache/băng thông khác nhau. Tôi giữ Q4 làm model phục vụ chính vì chất lượng tốt hơn với chi phí decode vừa phải.

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.21 | Không đo riêng | 42,000 | 42,000 | 0 |
| 50 | 0.22 | Không đo riêng | 43,000 | 43,000 | 0 |

`python -m llama_cpp.server` đã serve `/v1/chat/completions` thành công; smoke test trả lời trong 2,199 ms. Python server trả 404 cho `/metrics`, vì vậy không có native metric `llamacpp:n_busy_slots_per_decode`. Tôi không build native `llama-server` trong lượt chạy này; bằng chứng load test nằm trong `benchmarks/02-server-results.md` và log Locust.

**Batching observation:** Khi tăng từ 10 lên 50 users, completed RPS hầu như không đổi (0.21 → 0.22) trong khi E2E P95 tăng nhẹ (42 → 43 s). CPU decode đã bão hòa; request bổ sung chủ yếu nằm trong queue. Không thể khẳng định peak busy slots vì Python server không expose native metrics, nhưng hình dạng kết quả phù hợp với queueing dưới tải hơn là throughput scaling.

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub — localhost only, không có Kubernetes/cloud deployment.
- **N17 (Data pipeline):** stub — in-memory documents trong skeleton.
- **N18 (Lakehouse):** stub — không dùng lakehouse; provenance lấy từ dataset nội bộ của pipeline.
- **N19 (Vector + Feature Store):** TOY_DOCS keyword-overlap retriever của skeleton.

Pipeline chạy đủ 3 query và in provenance contexts. Timing đo được:

| Query | Retrieve (ms) | llama-server (ms) | Total (ms) |
|---|---:|---:|---:|
| Why is goodput more useful than throughput? | 0.0 | 15,435.3 | 15,435.5 |
| What problem does PagedAttention actually solve? | 0.1 | 5,984.6 | 5,984.8 |
| When should I think about disaggregated serving? | 0.1 | 5,144.2 | 5,144.3 |

**Reflection:** Retrieval gần như không đáng kể vì TOY_DOCS là in-memory keyword overlap. Llama-server là bottleneck rõ ràng; thời gian phụ thuộc số token trả về và hàng đợi CPU, đúng với kỳ vọng rằng decode CPU bị giới hạn bởi băng thông bộ nhớ.

## 5. Bonus — The single change that mattered most

**Change:** Đổi quantization từ Q4_K_M sang Q2_K trên cùng laptop, vẫn giữ 6 physical cores, context 2048 và CPU-only.

```text
before: Q4_K_M = 15.2 tok/s, TPOT P50 = 65.7 ms
after:  Q2_K   = 17.1 tok/s, TPOT P50 = 58.3 ms
speedup: 1.13× decode throughput
```

Decode của LLM chủ yếu đọc weights và KV cache lặp lại cho từng token, nên thường bị memory-bandwidth bound hơn là compute-bound. Q2_K dùng weights nhỏ hơn Q4_K_M; lượng dữ liệu phải kéo từ RAM/cache trong mỗi decode step giảm, vì vậy TPOT P50 giảm và decode rate tăng. Tuy nhiên, lợi ích không đồng đều ở mọi metric: TTFT P50 của Q2_K lại cao hơn Q4_K_M trong run này. Prefill có batching, prompt layout và cache behavior khác decode, nên “quant nhỏ hơn luôn nhanh hơn” là một mental model sai.

Tôi cũng chạy C8 semantic-cache offline. Với threshold 0.80, cache hit 3/8 request (38%) và bỏ qua ba simulated 250 ms LLM calls. Semantic cache có thể loại bỏ toàn bộ prefill + decode cho một paraphrase, nhưng threshold thấp quá sẽ trả lời sai/stale; vì vậy hit-rate không phải metric duy nhất để tối ưu.

## 6. Điều ngạc nhiên nhất

50 users không làm throughput tăng đáng kể so với 10 users. Trên CPU-only, tăng concurrency nhanh chóng chuyển thành queueing latency thay vì khả năng decode thêm token.

## 7. Self-graded checklist

- [x] `hardware.json` đã tạo và phản ánh đúng CPU/RAM qua PowerShell CIM fallback.
- [x] `models/active.json` trỏ tới hai GGUF hợp lệ.
- [x] `benchmarks/01-quickstart-results.md` đã tạo từ benchmark thật.
- [x] `benchmarks/02-server-results.md` đã tổng hợp hai lượt Locust.
- [x] `benchmarks/bonus-semantic-cache.md` đã ghi threshold sweep.
- [ ] Chụp và lưu tối thiểu 6 screenshots vào `submission/screenshots/`.
- [ ] Chạy `make verify` lại sau khi có screenshots.
- [ ] Đặt repo GitHub ở chế độ public rồi nộp URL lên LMS.
