# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **12 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 25.4 | 78% |
| 6 | 27.6 | 85% |
| 12 | 29.8 | 92% |
| 16 | 30.3 | 93% |
| 32 | 32.5 | 100% |

**Best**: `-t 32` at 32.5 tok/s
**Slowest tested**: `-t 1` at 25.4 tok/s (1.28x spread)
**Against the physical-core default** (`-t 12`, 29.8 tok/s): 1.09x

Use this in your run:

```bash
LAB_N_THREADS=32 make bench
```

## Your explanation

1. **Khảo sát đường cong hiệu năng (Scaling Curve):** Tốc độ giải mã (`tg128`) tăng dần từ **25.4 tok/s** tại `-t 1` lên **29.8 tok/s** tại `-t 12` (mức physical cores mặc định) và tiếp tục nhích nhẹ lên **32.5 tok/s** tại `-t 32` (tăng tổng cộng **1.28x** so với 1 thread).
2. **Phân tích điểm Knee và Kiến trúc CPU:** CPU Intel Core i7-1260P sử dụng kiến trúc lai (Alder Lake: 4 P-cores + 8 E-cores = 12 physical / 16 logical cores). 
   - Điểm chuyển tiếp (knee) bắt đầu xuất hiện từ `-t 12` đến `-t 16`: tốc độ tăng chậm lại rõ rệt (từ 29.8 lên 30.3 tok/s, chỉ tăng ~1.6%) do các E-cores có băng thông IPC thấp hơn và các luồng bắt đầu cạnh tranh kênh bộ nhớ L3 cache chung.
   - Khi nâng lên `-t 32` (oversubscription), tốc độ vẫn đạt đỉnh 32.5 tok/s vì model Qwen3.5 0.8B có kích thước rất nhỏ (0.50 GB), vừa vặn nằm phần lớn trong cache/băng thông RAM nhanh, giúp hệ điều hành xen kẽ lập lịch các luồng tính toán ma trận hiệu quả mà chưa bị nghẽn memory bus nghiêm trọng.
