# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 18782 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

Peak batch width ghi nhận được là 3.96 trên tổng số 4 slots, cho thấy Continuous Batching đã chạy gần mức tối đa công suất của server (99% số slot được sử dụng).

Con số 3.96 này thấp hơn rất nhiều so với Effective Concurrency (41.9) được đo trong `02-server-results.md`. Sự khác biệt này là do định nghĩa đo lường:
- `n_busy_slots_per_decode` (3.96) chỉ đo số lượng request **đang được xử lý đồng thời** bởi mô hình, và nó bị giới hạn cứng bởi cấu hình `--parallel 4`.
- Effective Concurrency (41.9) theo định luật Little's Law bao hàm **toàn bộ request đang nằm trong hệ thống**, bao gồm cả những request đang phải chờ trong hàng đợi (như ghi nhận `requests_deferred = 45`).

Cả hai con số đều đúng và phản ánh bức tranh toàn cảnh: Server đang hoạt động hết công suất phần cứng cho phép (3.96/4 slot) nhưng vẫn không đủ sức phục vụ tức thời lượng tải đổ vào, dẫn đến hàng đợi phình to (41.9 request trong hệ thống).
