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

Đường cong kết quả trên hệ thống này gần như đi ngang (flat), với khoảng cách giữa `-t 1` và `-t 32` chỉ là 1.08x (115.0 tok/s lên 124.0 tok/s). Điểm "knee" (điểm gập) dường như không xuất hiện một cách rõ rệt.

Kết quả khác biệt với kỳ vọng thông thường (peak tại số lượng nhân vật lý - physical cores) này cho thấy quá trình decode (tg128) trên máy không bị giới hạn bởi năng lực tính toán của CPU (compute-bound), mà bị nghẽn thắt cổ chai ở băng thông bộ nhớ (memory-bandwidth bound). Chỉ với 1 hoặc 5 luồng, CPU đã gần như tiêu thụ cạn kiệt băng thông RAM để truyền trọng số (weights) của mô hình. Do đó, việc thêm nhiều luồng tính toán hơn không thể đẩy nhanh quá trình vì chúng phải xếp hàng chờ dữ liệu từ RAM. Mức tăng nhỏ ở `-t 32` có thể đến từ khả năng latency hiding tốt hơn khi oversubscribe, hoặc chỉ là nhiễu hệ thống, nhưng về bản chất đây vẫn là một quá trình memory-bound.
