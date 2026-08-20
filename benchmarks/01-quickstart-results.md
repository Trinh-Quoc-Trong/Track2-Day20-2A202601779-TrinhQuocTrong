# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 9091 | 45 / 1515 | 8.7 / 9.0 | 594 / 2084 / 2084 | 114.8 |
| UD-Q2_K_XL | 2.24 | 6064 | 60 / 2930 | 9.8 / 13.0 | 675 / 3750 / 3750 | 102.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.12x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

Trên hệ thống Intel Core i5-14400F (chỉ dùng CPU, 10 threads), phiên bản `UD-Q2_K_XL` (102.3 tok/s) có tốc độ giải mã (decode) thấp hơn `UD-Q4_K_XL` (114.8 tok/s). Hiện tượng này bắt nguồn từ chi phí tài nguyên giải mã (dequantization overhead) của định dạng 2-bit vượt ngưỡng năng lực tính toán, dẫn đến tình trạng compute-bound cục bộ tại pha giải mã thay vì memory-bandwidth bound.

Do mức tiết kiệm bộ nhớ (0.73 GB) không bù đắp được sự sụt giảm về cả tốc độ sinh token lẫn chất lượng văn bản, phiên bản `UD-Q4_K_XL` (4-bit) được xác định là cấu hình tối ưu cho hệ thống này.
