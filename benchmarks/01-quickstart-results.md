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

Trên cấu hình này (Intel Core i5-14400F, không dùng GPU cho decode, chạy với 10 threads), phiên bản `UD-Q2_K_XL` (2-bit) thực sự **chậm hơn** so với `UD-Q4_K_XL` (4-bit) (Decode tốc độ 102.3 tok/s so với 114.8 tok/s). Hiện tượng này xảy ra do máy tính bị giới hạn bởi năng lực tính toán (compute-bound) trong bước giải nén (dequantization) lượng bit dày đặc của định dạng 2-bit, thay vì bị nghẽn băng thông bộ nhớ (memory-bandwidth bound) như thông thường.

Ngoài ra, chất lượng câu trả lời của bản 2-bit thường kém hơn rõ rệt so với 4-bit do mất mát thông tin. Vì mức tiết kiệm RAM chỉ là 0.73 GB nhưng lại bị đánh đổi bởi cả độ trễ cao hơn và chất lượng sinh text thấp hơn, phiên bản `UD-Q4_K_XL` (4-bit) hoàn toàn là lựa chọn tốt nhất để sử dụng trên hệ thống này.
