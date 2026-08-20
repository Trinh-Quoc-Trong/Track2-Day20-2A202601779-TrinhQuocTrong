# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **10 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 115.0 | 93% |
| 5 | 121.6 | 98% |
| 10 | 122.0 | 98% |
| 16 | 121.1 | 98% |
| 32 | 124.0 | 100% |

**Best**: `-t 32` at 124.0 tok/s
**Slowest tested**: `-t 1` at 115.0 tok/s (1.08x spread)
**Against the physical-core default** (`-t 10`, 122.0 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=32 make bench
```

Đồ thị hiệu năng theo số lượng luồng (thread) thể hiện xu hướng tiệm cận phẳng (flat curve), với biên độ chênh lệch giữa `-t 1` và `-t 32` chỉ đạt 1.08x (115.0 tok/s đến 124.0 tok/s). Không ghi nhận điểm gãy (knee) rõ rệt.

Kết quả phân tích chỉ ra rằng pha giải mã (decode) bị giới hạn hoàn toàn bởi băng thông bộ nhớ (memory-bandwidth bound) thay vì năng lực tính toán của CPU (compute-bound). Dù cấu hình ở mức 1 hay 5 luồng, hệ thống đã đạt đến giới hạn truyền tải trọng số từ RAM vào bộ nhớ đệm CPU. Việc gia tăng số luồng xử lý không đem lại hiệu quả do hiện tượng thắt cổ chai I/O. Mức cải thiện vi mô tại `-t 32` chủ yếu bắt nguồn từ cơ chế latency hiding của luồng oversubscribe, khẳng định bản chất bài toán là tối ưu I/O thay vì mở rộng tính toán.
