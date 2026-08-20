# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 13 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.92 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a - not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4895 |

Highest sampled value was **3.92 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Dựa trên số liệu đo lường thực tế từ `/metrics` trong quá trình load test 50 users:

1. **Bằng chứng rõ ràng về Continuous Batching:** Chỉ số `n_busy_slots_per_decode` đạt đỉnh **3.92 / 4 slots** (tương đương **98%** công suất slot tối đa). Điều này chứng minh scheduler của `llama-server` đã liên tục ghép các request đồng thời vào cùng một bước decode forward pass thay vì xử lý tuần tự từng request đơn lẻ.
2. **Trạng thái bão hòa và hàng đợi (Queuing):** Toàn bộ 4 slots xử lý đều bận (`requests_processing = 4`), đồng thời `requests_deferred` đạt tới **46 requests**. Số lượng 50 users gửi request dồn dập đã vượt xa dung lượng 4 slots hiện có, dẫn đến hiện tượng hàng đợi phình to.
3. **Phân biệt Queue Time và Compute Time:** Phần lớn độ trễ tăng cao ở P95/P99 khi tải 50 users là **thời gian chờ trong hàng đợi (queue time)** của các request bị deferred, chứ không phải do tốc độ tính toán (compute time) trên mỗi token bị chậm đi.
