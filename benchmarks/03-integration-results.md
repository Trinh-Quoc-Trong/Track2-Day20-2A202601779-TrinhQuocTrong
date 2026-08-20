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


Trong pipeline RAG này:
- N16 (Embedding Model): **Stubbed** (hoàn toàn bỏ qua, hệ thống rơi vào luồng keyword overlap fallback, tốn 0.0 ms).
- N17 (Vector Database): **Stubbed** (dữ liệu `TOY_DOCS` được lưu cứng dưới dạng một list dictionary trong RAM).
- N18 (Retrieval Algorithm): **Stubbed** (sử dụng thuật toán đếm từ vựng trùng lặp cơ bản, độ trễ 0.0 ms do dữ liệu quá nhỏ).
- N19 (LLM Inference Server): **Real** (thực hiện HTTP POST request tới `llama-server` ở `localhost:8080` đang chạy Gemma 4 E2B).

Giai đoạn chiếm thời gian áp đảo là **llm** (100% thời lượng, trung bình 5430.0 ms). Điều này hoàn toàn khớp với kỳ vọng vì các khâu còn lại bị stub và chỉ tốn vài chục micro giây để xử lý chuỗi trên RAM.

Nếu phải giảm một nửa độ trễ (latency) của pipeline này, tôi sẽ tấn công duy nhất vào khâu **llm**. Vì nó chiếm 100% thời gian, mọi tối ưu ở các khâu khác là vô nghĩa. Đối với LLM trong tác vụ RAG, phần tốn thời gian nhất (compute-bound) là quá trình Prefill (đọc ngữ cảnh). Do đó, tôi sẽ áp dụng kỹ thuật **Prompt Caching** (như Semantic Cache hoặc Prefix/KV Cache) để bỏ qua hoàn toàn giai đoạn tính toán Prefill đối với các context được retrieve trùng lặp, từ đó có thể dễ dàng cắt giảm 50-80% thời gian phản hồi.
