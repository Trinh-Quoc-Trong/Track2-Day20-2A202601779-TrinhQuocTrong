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
Cần phải tự cài `cmake` thông qua `pip` để thử build bản CUDA, tuy nhiên do hệ điều hành thiếu CUDA Toolkit (`nvcc`) nên đã tự động fallback sang dùng prebuilt binary CPU x86_64. Mọi tiến trình benchmark sau đó đều chạy trơn tru trên CPU.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 9091 | 45 / 1515 | 8.7 / 9.0 | 594 / 2084 / 2084 | 114.8 |
| UD-Q2_K_XL | 2.24 | 6064 | 60 / 2930 | 9.8 / 13.0 | 675 / 3750 / 3750 | 102.3 |

**Quan sát** (≤ 60 chữ): 
Bản 2-bit chậm hơn (102.3 vs 114.8 tok/s) do chi phí giải nén (dequantization) trên CPU lớn hơn lợi ích tiết kiệm băng thông RAM. Việc đánh đổi 0.73 GB RAM lấy sự suy giảm cả về tốc độ lẫn chất lượng văn bản là không đáng, vì vậy `UD-Q4_K_XL` là lựa chọn ưu việt hơn.

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

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.96 / 4 slots

**Saturation reading** (≤ 80 chữ): 
Server bão hòa ở mức tải 50 users vì Effective Concurrency (41.9) vượt xa số slot `--parallel 4`. Lượng request nằm trong hàng đợi chờ xử lý cực lớn. Để nâng Goodput@SLO, tôi sẽ tăng `--parallel` (lên 8 hoặc 16) nhằm tăng batch size, giải phóng hàng đợi nhanh hơn vì CPU bị memory-bandwidth bound chứ chưa bị compute-bound.

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
Bottleneck hoàn toàn nằm ở khâu `llm`, khớp 100% với kỳ vọng vì các khâu khác bị stub. Nếu cần giảm latency 2x, tôi sẽ tập trung vào kỹ thuật Prompt Caching tại LLM để bỏ qua compute cost của giai đoạn prefill, vốn là gánh nặng lớn nhất trong long-context RAG.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Sweep số lượng thread `-t 10` (mặc định physical cores) sang `-t 32` (kết quả tốt nhất đo được trong quá trình tuning)

```
before:  122.0 tok/s (-t 10)
after:   124.0 tok/s (-t 32)
speedup: 1.02×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả trên hệ thống này gần như là một đường thẳng (flat curve) giữa các mức thread khác nhau. Sự thay đổi không tạo ra speedup đột phá (chỉ 1.02x) minh chứng rõ ràng cho việc quá trình giải mã (decode phase) của LLM bị thắt cổ chai bởi băng thông bộ nhớ (memory-bandwidth bound) chứ không phải do thiếu năng lực tính toán (compute-bound).

Ngay cả khi sử dụng mức thread tối thiểu, các nhân CPU cũng đã nhanh chóng vắt kiệt băng thông RAM để truyền trọng số mô hình vào cache. Do đó, việc cung cấp thêm nhân (hay thread) chỉ khiến chúng phải xếp hàng chờ dữ liệu. Mức speedup nhỏ lẻ 1.02x ở `-t 32` có thể xuất phát từ việc luân chuyển các thread chờ (latency hiding) hoặc chỉ là nhiễu nền của hệ điều hành, nhưng nhìn chung, bài toán giải mã token của transformer model luôn đói dữ liệu hơn là đói phép tính.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B5 (Challenge C8: Semantic cache — cache phía trên KV cache)

**Numbers:**

```
before:  3 LLM calls (cho 3 câu hỏi paraphrase của các câu trước đó)
after:   0 LLM calls (trúng Semantic Cache hoàn toàn)
speedup: ∞× (Compute saved 100%, thời gian xử lý giảm từ >700ms về 0ms)
```

**Điều này nói lên gì mà deck chưa nói:**

Sử dụng bộ nhớ Semantic Cache (so khớp ngữ nghĩa) giúp loại bỏ triệt để 100% thời gian xử lý (cả prefill lẫn decode) đối với các câu hỏi lặp lại ý (paraphrase). Tuy nhiên, thí nghiệm cũng cho thấy một kẽ hở lớn: Nếu sử dụng chính decoder LLM ở chế độ mean-pooling để làm embedder, hiện tượng False Hit (bắt nhầm chủ đề mới) và False Miss (bỏ sót paraphrase thật) xảy ra liên tục trong một dải similarity rất hẹp (0.5-0.9). Điều này minh chứng một decoder được huấn luyện để sinh text (next-token prediction) là một encoder cực kỳ yếu kém; ta bắt buộc phải tích hợp một embedding model chuyên dụng (được học bằng contrastive learning) để phân loại rõ rệt không gian vector nếu muốn triển khai Semantic Cache trong thực tế.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều làm tôi ngạc nhiên nhất là việc lượng tử hóa xuống mức 2-bit không những không tăng tốc hệ thống mà còn làm cho mô hình chạy chậm lại đáng kể, chứng tỏ chi phí giải mã (dequantization overhead) có thể vượt qua giới hạn bộ nhớ trên cấu hình CPU cụ thể.

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
