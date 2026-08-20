# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7814 | 633 / 798 | 36.3 / 40.5 | 2872 / 3254 / 3254 | 27.6 |
| UD-Q2_K_XL | 0.39 | 9295 | 1284 / 1557 | 530.4 / 576.7 | 34842 / 37810 / 37810 | 1.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **14.53x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên máy của em (Windows AMD64, thực thi chủ yếu trên CPU), bản lượng tử hoá 2-bit (`UD-Q2_K_XL`) **hoàn toàn không đáng để sử dụng**.

1. **Hiệu năng suy giảm nghiêm trọng:** `UD-Q2_K_XL` chỉ đạt tốc độ giải mã **1.9 tok/s**, chậm hơn **14.53 lần** so với **27.6 tok/s** của `Q4_K_M`. Chi phí giải mã mỗi token (TPOT P50) tăng vọt từ **36.3 ms** lên **530.4 ms**, đồng thời độ trễ trả về token đầu tiên (TTFT P50) tăng gấp đôi từ **633 ms** lên **1284 ms**.
2. **Cơ chế cốt lõi:** Việc giảm số bit (từ 4-bit xuống 2-bit) chỉ giúp tăng tốc độ khi hệ thống bị nghẽn băng thông bộ nhớ (memory bandwidth-bound). Trên CPU hiện tại, hệ thống rơi vào trạng thái **nghẽn tính toán (compute-bound)**: lượng RAM tiết kiệm được vỏn vẹn **0.11 GB** (0.39 GB so với 0.50 GB) không bù đắp nổi chi phí tính toán rất lớn để giải nén (dequantize) các trọng số 2-bit phi tuyến trên các thanh ghi CPU ALU.
3. **Kết luận:** `Q4_K_M` là lựa chọn tối ưu vượt trội cho việc serving trên cấu hình phần cứng này, đảm bảo tốc độ phản hồi mượt mà và chất lượng sinh câu tốt với mức tiêu tốn RAM gần như không đáng kể.
