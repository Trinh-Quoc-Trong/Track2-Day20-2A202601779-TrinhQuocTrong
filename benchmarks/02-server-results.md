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

Server đã bão hòa (saturate) khi lượng truy cập tăng lên 50 users. Bằng chứng rõ nhất là Effective Concurrency đạt 41.9 trong khi máy chủ chỉ cấu hình `--parallel 4`. Hệ số occupancy/slot ratio lên tới 10.48 cho thấy phần lớn thời gian (latency) của các request là thời gian nằm chờ trong hàng đợi (Queue Time) chứ không phải thời gian tính toán thực tế.

Mặc dù hệ thống báo cáo "At capacity, still scaling" do RPS vẫn tăng 3.60x, nhưng với P50 latency ở mức 12 giây cho 50 users, trải nghiệm người dùng thực tế sẽ bị vi phạm SLO nghiêm trọng. 

Để tăng Goodput@SLO, thay đổi đầu tiên cần thực hiện là tăng `--parallel` (ví dụ lên 8 hoặc 16) để server có thể tiếp nhận batch lớn hơn, tận dụng việc CPU đang bị memory-bandwidth bound (như phát hiện ở bước Tune). Việc tăng parallel sẽ giúp giải phóng hàng đợi nhanh hơn và giảm Queue Time, miễn là hệ thống vẫn còn đủ RAM để chứa thêm KV cache cho các context bổ sung.
