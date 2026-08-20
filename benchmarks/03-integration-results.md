# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 15725.4 | 15725.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 278.7 | 278.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 285.8 | 285.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **5430.0** · total **5430.0**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


Phân tích thành phần RAG pipeline:
- N16 (Embedding Model): **Stubbed** (kích hoạt fallback keyword overlap, độ trễ 0.0 ms).
- N17 (Vector Database): **Stubbed** (dữ liệu `TOY_DOCS` cấp phát tĩnh trên RAM).
- N18 (Retrieval Algorithm): **Stubbed** (thuật toán word overlap, độ trễ tiệm cận 0 do tải lượng thấp).
- N19 (LLM Inference Server): **Real** (giao tiếp HTTP POST tới `llama-server` chạy Gemma 4 E2B).

Điểm nghẽn (bottleneck) hệ thống tập trung hoàn toàn tại khâu **llm** (chiếm 100% tổng thời gian, trung bình 5430.0 ms). Kết quả này phản ánh chính xác cấu trúc pipeline do các dịch vụ phụ trợ đã được giả lập (stub).

Để tối ưu hóa giảm 50% độ trễ tổng thể, trọng tâm duy nhất là thành phần **llm**. Do giai đoạn Prefill xử lý ngữ cảnh dài là tác vụ tiêu tốn chi phí tính toán (compute-bound) lớn nhất trong RAG, kỹ thuật **Prompt Caching** (như Prefix/KV Cache hoặc Semantic Cache) là giải pháp tối ưu. Cơ chế này cho phép triệt tiêu hoàn toàn chu kỳ tính toán Prefill cho các tài liệu (context) truy xuất trùng lặp, tạo tiền đề cắt giảm 50-80% thời gian phản hồi.
