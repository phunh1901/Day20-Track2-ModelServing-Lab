# Bonus - Quantization sweep (Qwen3.5 0.8B, Unsloth Dynamic ladder)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=12` `ngl=99` · metric `tg128`

| Quantization | Size (GB) | tg128 (tok/s) | vs UD-Q4_K_XL | tok/s per GB |
|:--|--:|--:|--:|--:|
| UD-Q2_K_XL | 0.39 | 2.0 | 0.06x | 5.3 |
| UD-Q4_K_XL | 0.52 | 31.8 | 1.00x | 61.2 |
| UD-Q6_K_XL | 0.72 | 25.1 | 0.79x | 34.8 |

Decode is memory-bandwidth-bound, so fewer bytes per weight usually means more
tokens per second -- the "tok/s per GB" column shows how much of that you are
actually getting back per gigabyte spent.

Speed is only half the trade. The other half is quality, and no benchmark here
measures it. Serve two of these (`make serve` and
`.venv/bin/python labs/02-serve/serve.py --compare`) and ask each the same three questions
before you claim a winner.

## Your finding

Dựa trên kết quả khảo sát thang lượng tử hóa Unsloth Dynamic ladder trên Qwen3.5 0.8B (Challenge C5):

1. **Phân tích đánh đổi Tốc độ vs Dung lượng (Trade-off):**
   - **`UD-Q4_K_XL` (0.52 GB)** là "điểm ngọt" (sweet spot) tối ưu nhất toàn diện: đạt tốc độ sinh token cao nhất **31.8 tok/s** và hiệu suất sử dụng dung lượng vượt trội **61.2 tok/s per GB**.
   - **`UD-Q6_K_XL` (0.72 GB)**: Dung lượng tăng thêm 38% khiến lượng dữ liệu CPU phải nạp từ RAM qua mỗi bước decode tăng lên, làm tốc độ giải mã giảm về **25.1 tok/s** (đạt 79% so với Q4).
   - **`UD-Q2_K_XL` (0.39 GB)**: Mặc dù tiết kiệm được 0.13 GB RAM, tốc độ lại sụp đổ nghiêm trọng xuống **2.0 tok/s** (chậm hơn gần 16 lần so với Q4). Lý do cốt lõi là trên CPU, chi phí tính toán giải nén (dequantize) các block trọng số 2-bit phi tuyến lớn hơn rất nhiều so với lượng băng thông bộ nhớ tiết kiệm được.

2. **Lựa chọn triển khai thực tế & Điểm gãy chất lượng (Challenge C5):**
   - **Mức em lựa chọn triển khai thực tế:** **`UD-Q4_K_XL`**. Mức này giữ được sự cân bằng hoàn hảo giữa tốc độ cực nhanh (31.8 tok/s), kích thước gọn nhẹ (0.52 GB) và bảo toàn chất lượng câu trả lời mạch lạc, đúng cú pháp logic.
   - **Điểm gãy chất lượng (Quality Breakdown):** Chất lượng suy giảm mạnh ở **`UD-Q2_K_XL`** (2-bit). Khi thử nghiệm sinh câu trả lời, bản 2-bit xuất hiện hiện tượng lặp từ ngữ, mất liên kết ngữ pháp và hallucination rõ rệt. Kết hợp với tốc độ cực chậm (2 tok/s), bản 2-bit hoàn toàn không thể sử dụng trong thực tế.
   - **Kết luận:** Mức lượng tử hóa nhỏ nhất còn hữu ích trên phần cứng này là **4-bit (`UD-Q4_K_XL`)**.
