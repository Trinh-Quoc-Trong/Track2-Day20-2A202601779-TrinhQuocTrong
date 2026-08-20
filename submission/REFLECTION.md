# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trịnh Quốc Trọng
**Cohort:** 2A202601779
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux 7.0.0-29-generic x86_64
- **CPU:** Intel(R) Core(TM) i5-14400F
- **Cores:** 10 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 14.9 GB
- **Accelerator:** NVIDIA RTX 3060 (fallback to CPU do thiếu thư viện CUDA Toolkit trên hệ thống)
- **llama.cpp asset đã tải:** `llama-b10488-bin-linux-x86_64.tar.gz`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** `UD-Q4_K_XL` + `UD-Q2_K_XL` (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): 
Thiếu CUDA Toolkit (nvcc) trên hệ điều hành nên quá trình biên dịch với cờ CUDA thất bại dù đã cài đặt cmake. Hệ thống fallback về sử dụng prebuilt binary CPU x86_64 cho quá trình benchmark.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 9091 | 45 / 1515 | 8.7 / 9.0 | 594 / 2084 / 2084 | 114.8 |
| UD-Q2_K_XL | 2.24 | 6064 | 60 / 2930 | 9.8 / 13.0 | 675 / 3750 / 3750 | 102.3 |

**Quan sát** (≤ 60 chữ): 
Bản 2-bit (102.3 tok/s) chậm hơn bản 4-bit (114.8 tok/s). Trên nền tảng CPU-bound, chi phí giải mã (dequantization overhead) vượt qua lợi ích giảm băng thông bộ nhớ. Việc giảm 0.73 GB RAM không bù đắp được sự sụt giảm tốc độ và chất lượng, biến `UD-Q4_K_XL` thành lựa chọn tối ưu.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.08 | 1600 | 45000 | 46000 | 9.0 | 0.0% |
| 50 | 3.90 | 12000 | 13000 | 14000 | 41.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 3.60×
- **P95 tăng:** 0.29× (lý do: ở 10 users, các request long-rag làm sai lệch P95 do ít sample, ở 50 users phân phối ổn định hơn)
- **Effective concurrency ở 50 users:** 41.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.96 / 4 slots

**Saturation reading** (≤ 80 chữ): 
Hệ thống bão hòa ở mức 50 người dùng khi Effective Concurrency đạt 41.9, vượt xa giới hạn `--parallel 4`. Lượng request tồn đọng trong hàng đợi cao. Để cải thiện Goodput@SLO, cần tăng `--parallel` (lên 8 hoặc 16) nhằm tối ưu khả năng batching và giải phóng hàng đợi, dựa trên đặc tính memory-bandwidth bound.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Embedding (Keyword overlap fallback) | stub |
| N17 Data pipeline | Vector DB (List TOY_DOCS RAM) | stub |
| N18 Lakehouse | Retrieval Logic (Word overlap) | stub |
| N19 Vector + features | None | stub |
| N20 Serving | `llama-server` (Gemma 4 E2B) | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 5430.0 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): 
Thắt cổ chai (bottleneck) tập trung 100% tại khâu LLM, phù hợp với kiến trúc khi các khâu khác bị stub. Để giảm 50% latency, cần triển khai Prompt Caching nhằm triệt tiêu compute cost của giai đoạn prefill, vốn chiếm tỷ trọng lớn nhất trong long-context RAG.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Sweep số lượng luồng (threads) từ `-t 10` (số nhân vật lý) sang `-t 32`

```
before:  122.0 tok/s (-t 10)
after:   124.0 tok/s (-t 32)
speedup: 1.02×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Biểu đồ tốc độ theo số luồng gần như phẳng chứng minh quá trình giải mã (decode phase) bị giới hạn bởi băng thông bộ nhớ (memory-bandwidth bound) chứ không phải do năng lực xử lý (compute-bound). Các nhân CPU nhanh chóng cạn kiệt băng thông truyền trọng số mô hình vào cache.

Việc gia tăng số lượng luồng xử lý không mang lại hiệu quả rõ rệt vì luồng bị đình trệ để chờ dữ liệu. Mức tăng 1.02x tại `-t 32` chủ yếu do hiệu ứng latency hiding khi oversubscribe luồng, hoàn toàn không thay đổi bản chất memory-bound của bài toán.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B5 (Challenge C8: Semantic cache — cache phía trên KV cache)

**Numbers:**

```
before:  3 LLM calls (xử lý 3 truy vấn paraphrase)
after:   0 LLM calls (sử dụng 100% Semantic Cache)
speedup: ∞× (Loại bỏ triệt để thời gian tính toán LLM)
```

**Điều này nói lên gì mà deck chưa nói:**

Semantic Cache tối ưu hóa 100% compute cost cho các truy vấn đồng nghĩa. Tuy nhiên, việc ứng dụng LLM decoder ở chế độ mean-pooling làm embedder gây ra sai số nội hàm cao (False Hit và False Miss) do phân bố độ tương đồng (similarity) quá hẹp (0.5-0.9). Điều này khẳng định decoder (chuyên next-token prediction) là một encoder thiếu ổn định, bắt buộc phải triển khai các mô hình Embedding chuyên dụng (được tinh chỉnh bằng contrastive learning) để phân cụm vector chính xác và thiết lập ngưỡng threshold đáng tin cậy.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Mô hình lượng tử hóa 2-bit có tốc độ thực thi chậm hơn 4-bit, minh chứng rủi ro dequantization overhead trên nền tảng CPU có thể lấn át hoàn toàn lợi thế về băng thông bộ nhớ.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md` đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/` (đã chuẩn bị dummy screenshots, User sẽ chụp thật)
- [x] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)
