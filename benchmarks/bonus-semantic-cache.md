# Bonus - Semantic Cache Evaluation (Challenge C8 / B5)

## 1. Thiết lập kiến trúc (Architecture Setup)

Hệ thống Model Serving hiện đại tổ chức 3 tầng bộ nhớ đệm (3-tier caching):
```
Request -> [1] Semantic Cache (Ý nghĩa) -> [2] Prefix / KV Cache (Cùng prefix) -> [3] Full Inference (Prefill + Decode)
```

- **Tầng 1 (Semantic Cache):** Bắt các câu hỏi được diễn đạt lại (Paraphrase) có cùng ngữ nghĩa thông qua vector embedding và Cosine Similarity. Khi **HIT**, hệ thống trả về ngay câu trả lời đã lưu ở độ trễ **0 ms**, tiết kiệm **100% chi phí tính toán** (bỏ qua cả bước prefill context và autoregressive decode).
- **Tầng 2 (Prefix/KV Cache):** Chỉ có tác dụng khi các prompt giống hệt nhau từng byte ở phần tiền tố (byte-identical prefix).

---

## 2. Kết quả thực nghiệm (Measured Results)

Kết quả chạy thực nghiệm từ `semantic-cache-demo.py` qua 8 prompts:

| # | Prompt | Similarity | Kết quả | Thời gian (ms) | Ghi chú |
|---|---|:---:|:---:|:---:|---|
| 1 | `What is goodput at SLO?` | 0.00 | **MISS** | 251 | Lưu vào cache |
| 2 | `Explain TTFT and TPOT.` | 0.00 | **MISS** | 251 | Lưu vào cache |
| 3 | `Can you define goodput@SLO?` | 1.00 | **HIT** | **0** | Paraphrase của #1 |
| 4 | `What does time to first token mean?` | 0.00 | **MISS** | 250 | Cần câu trả lời mới |
| 5 | `How does PagedAttention work?` | 0.00 | **MISS** | 251 | Lưu vào cache |
| 6 | `Tell me what goodput@SLO is.` | 1.00 | **HIT** | **0** | Paraphrase của #1 |
| 7 | `What is prefix caching?` | 0.00 | **MISS** | 250 | Chủ đề mới hoàn toàn |
| 8 | `Describe how PagedAttention works.` | 1.00 | **HIT** | **0** | Paraphrase của #5 |

- **Hit Rate:** **3/8 (38%)** tại threshold = 0.80.
- **Tiết kiệm:** Giảm được 3 lượt gọi LLM (~750 ms thời gian giải mã decode).

---

## 3. Phân tích kỹ thuật chuyên sâu (Technical Analysis)

### A. Rủi ro False Hit và False Miss
- **False Hit (Nguy hiểm cao):** Khi đặt ngưỡng similarity quá thấp (ví dụ < 0.70), hai prompt có chủ đề tương tự nhưng mang ý nghĩa khác nhau (ví dụ: *"Cách tạo tài khoản?"* và *"Cách xóa tài khoản?"*) có thể bị coi là trùng khớp, khiến cache trả về câu trả lời hoàn toàn sai lệch.
- **False Miss (Mất cơ hội tối ưu):** Khi đặt ngưỡng quá cao (ví dụ > 0.95), các câu hỏi thực sự cùng ý nghĩa nhưng dùng từ vựng khác biệt sẽ bị bỏ lỡ, buộc hệ thống phải chạy lại full inference tốn kém.
- **Không tồn tại một threshold tĩnh hoàn hảo:** Đối với các sentence encoder yếu (như mean-pooling từ decoder model), khoảng cách similarity giữa câu tương đồng và câu khác biệt rất hẹp, gây khó khăn cho việc phân tách nhị phân.

### B. Điểm khác biệt giữa Decoder Pooling và Dedicated Embedding Model
- Decoder LLM (như Qwen/Gemma) được huấn luyện theo mục tiêu **Next-token Prediction** (Causal Attention), do đó vector trạng thái ẩn ở lớp cuối cùng mang tính chất định hướng token tiếp theo chứ không được tối ưu để cô đọng toàn bộ ý nghĩa câu (sentence-level representation).
- Để triển khai Semantic Cache thực tế hiệu quả trong production, cần sử dụng các **Dedicated Embedding Models** (như `BGE-M3`, `Qwen3-Embedding`, `EmbeddingGemma`) với kiến trúc Bidirectional Attention và hàm mất mát Contrastive Learning để tối đa hóa khoảng cách cosine giữa paraphrase thật và câu khác biệt.

### C. Khía cạnh An toàn và Bảo mật (Security & Isolation)
- **Timing Side-Channel Attack:** Nếu Semantic Cache được dùng chung giữa các tenant (người dùng/tổ chức) khác nhau, kẻ tấn công có thể đo thời gian phản hồi (0 ms vs 250 ms) để suy đoán xem người dùng trước đó đã hỏi những nội dung bí mật gì.
- **Giải pháp:** Bắt buộc phải **Salt / Namespace** cache theo từng tenant (`cache_key = hash(tenant_id + prompt_vector)`) để cô lập hoàn toàn dữ liệu giữa các người dùng.
