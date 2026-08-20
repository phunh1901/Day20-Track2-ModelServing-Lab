# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 7506.8 | 7506.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4340.1 | 4340.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8202.7 | 8202.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **6683.2** · total **6683.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **goodput** is more useful than raw throughput because it specifically accounts for the requests per second that met the **TTFT** (Target Throughput for Fullness) and **TPOT** (Target Throughput for Partial Overhead) targets.

In contrast, **raw throughput** ignores SLOs (Service Level Objects), meaning it does not consider how much of the capacity is actually utilized.

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** by storing the KV cache in non-contiguous pages.

This is achieved by removing the internal fragmentation that would otherwise waste most GPU memory, allowing the engine to utilize more available memory for the actual computation.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the **compute-bound** nature of the prefill step conflicts with the **memory-bandwidth-bound** nature of the decode step.

Specifically, this optimization occurs in scenarios where:
1.  **Prefill is compute-bound**: It requires significant processing power (e.g., GPU inference or heavy model quantization) but does not consume much memory.
2.  **Decode is memory-bandwidth-bound**: It requires high bandwidth to fetch weights for each generated token.

## Which N16-N19 pieces are real

| Module | Thành phần | Trạng thái | Ghi chú |
|---|---|---|---|
| **N16** Cloud/IaC | Kubernetes / Cloud stack | **Stub** | Chạy trực tiếp trên localhost |
| **N17** Data pipeline | Ingestion pipeline | **Stub** | Danh sách tài liệu in-memory |
| **N18** Lakehouse | Bảng Delta / Iceberg | **Stub** | Dict tài liệu mẫu `TOY_DOCS` |
| **N19** Vector + features | Vector search / Feature store | **Stub** | Thuật toán fallback keyword overlap |
| **N20** Model Serving | `llama-server` API | **Real** | Endpoint OpenAI HTTP thực thi Qwen3.5 0.8B |

### Phân tích Dominant Stage:
1. **Khớp với kỳ vọng:** Giai đoạn LLM chiếm tới **100%** tổng thời gian thực thi (6683.2 ms / 6683.4 ms), trong khi bước retrieve từ điển in-memory chỉ tốn 0.1 ms. Điều này hoàn toàn khớp với bản chất của RAG: giai đoạn sinh sinh từ mô hình ngôn ngữ luôn là điểm nghẽn nặng nhất do phải prefill toàn bộ context và decode tuần tự từng token.
2. **Chiến lược giảm độ trễ 2x:** Nếu cần giảm 2 lần độ trễ của pipeline, em sẽ tập trung toàn lực tối ưu **stage LLM** bằng cách:
   - Áp dụng **Prompt Caching** (cố định system prompt để server tái sử dụng KV prefix mà không phải tính lại prefill).
   - Tối ưu số lượng token sinh ra (`max_tokens` vừa đủ) và chắt lọc context retrieval ngắn gọn hơn để giảm chi phí tính toán prefill và giải mã autoregressive.
