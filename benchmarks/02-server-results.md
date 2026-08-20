# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 34 | 0.59 | 14000 | 23000 | 26000 | 8.7 | 0.0% |
| 50 | 42 | 0.78 | 33000 | 50000 | 53000 | 23.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.32x** (26% of linear) |
| P95 latency | **2.17x** |
| Effective concurrency at 50 users | 23.0 vs `--parallel 4` slots (occupancy/slot ratio 5.75) |

**Saturated.** Throughput delivered only 1.32x for 5x the offered load, and effective concurrency (23.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.32x while P95 moved 2.17x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

1. **Điểm bão hoà (Saturation):** Server bão hoà rõ rệt khi tải vượt qua 10 users và đạt đỉnh nghẽn ở 50 users. Khi tải tăng **5x**, throughput thực tế chỉ tăng **1.32x** (từ 0.59 lên 0.78 RPS, chỉ bằng 26% tốc độ tăng tuyến tính), cho thấy năng lực phục vụ đã plateau (đi ngang).
2. **Queue time vs Compute time:** Bằng chứng thuyết phục nhất là **Effective Concurrency = 23.0**, vượt gấp **5.75 lần** tổng số slot (`--parallel 4`). Điều này chứng minh tại 50 users, luôn có trung bình ~19 request phải chờ đợi. Do đó, mức tăng P95 từ **23.0s lên 50.0s (tăng 2.17x)** chủ yếu là **thời gian chờ trong hàng đợi (queue time)** chứ không phải do tốc độ tính toán (compute time) giảm.
3. **Knob tối ưu Goodput@SLO:** Nếu đặt mục tiêu SLO P95 ≤ 25s, knob em sẽ ưu tiên thay đổi đầu tiên là **tăng `--parallel` lên 6 hoặc 8** (nếu dung lượng RAM cho phép) để tăng số slot decode đồng thời, hoặc áp dụng **Admission Control** giới hạn concurrency nhằm loại bỏ queue time, bảo toàn goodput hợp lệ cho các request trong ngưỡng SLO.
