# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 64 | 1.08 | 1600 | 45000 | 46000 | 9.0 | 0.0% |
| 50 | 229 | 3.90 | 12000 | 13000 | 14000 | 41.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **3.60x** (72% of linear) |
| P95 latency | **0.29x** |
| Effective concurrency at 50 users | 41.9 vs `--parallel 4` slots (occupancy/slot ratio 10.48) |

**At capacity, still scaling.** All 4 decode slots are busy (effective concurrency 41.9) but throughput still rose 3.60x. You are at the knee -- the next increment of load is where P95 starts to run away.

P95 grew no faster than throughput (0.29x vs 3.60x), so this server still has headroom at 50 users.

## Your reading

Hệ thống bão hòa (saturate) ở tải 50 người dùng với mức Effective Concurrency đạt 41.9, vượt xa cấu hình `--parallel 4`. Hệ số occupancy/slot (10.48) chỉ ra rằng phần lớn thời gian phản hồi (latency) phát sinh từ hàng đợi chờ xử lý (Queue Time) thay vì thời gian tính toán (Compute Time).

Dù thông lượng (RPS) tăng 3.60x, mức P50 latency 12 giây sẽ dẫn đến vi phạm SLO nghiêm trọng trong môi trường thực tế. 

Để tối ưu hóa Goodput@SLO, ưu tiên hàng đầu là tăng thông số `--parallel` (lên 8 hoặc 16) nhằm mở rộng kích thước batch. Do hệ thống đang trong trạng thái memory-bandwidth bound, việc tăng batching sẽ tận dụng tốt hơn tài nguyên RAM và giải phóng hàng đợi nhanh chóng, miễn là dung lượng RAM còn trống đủ để cấp phát thêm KV Cache.
