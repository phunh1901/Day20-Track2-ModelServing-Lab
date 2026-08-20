# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Ngô Hoàng Phú
**Cohort:** K3B
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** 12th Gen Intel(R) Core(TM) i7-1260P
- **Cores:** 12 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 8 GB (7.7 GB usable)
- **Accelerator:** Vulkan
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của em

**Setup story** (≤ 80 chữ): Em chọn model nhẹ Qwen3.5 0.8B để phù hợp RAM 8GB. Khi chạy trên PowerShell, em fix lỗi ký tự bảng mã ANSI trong lab.ps1 để chạy trơn tru các target.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7814 | 633 / 798 | 36.3 / 40.5 | 2872 / 3254 / 3254 | 27.6 |
| UD-Q2_K_XL | 0.39 | 9295 | 1284 / 1557 | 530.4 / 576.7 | 34842 / 37810 / 37810 | 1.9 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit không đáng dùng: chậm hơn 14.5x (1.9 vs 27.6 tok/s), TTFT tăng gấp đôi do CPU nghẽn tính toán dequantization dù giảm 0.11 GB. Q4_K_M tối ưu hơn nhiều.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.59 | 14000 | 23000 | 26000 | 8.7 | 0.0% |
| 50 | 0.78 | 33000 | 50000 | 53000 | 23.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.32×
- **P95 tăng:** 2.17×
- **Effective concurrency ở 50 users:** 23.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.92 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hoà ở 50 users khi throughput chỉ tăng 1.32x còn P95 tăng 2.17x. Concurrency đạt 23.0 vượt xa 4 slots, chứng minh độ trễ tăng vọt là queue time do request phải chờ slot. Để nâng goodput@SLO, em sẽ tăng `--parallel` hoặc áp dụng admission control nhằm giảm hàng đợi.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Localhost | stub |
| N17 Data pipeline | In-memory list | stub |
| N18 Lakehouse | In-memory dict | stub |
| N19 Vector + features | Keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 6683.2 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM chiếm 100% thời gian (6.7s), hoàn toàn khớp kỳ vọng vì decode autoregressive trên CPU rất nặng. Để giảm độ trễ 2x, em sẽ tấn công trực tiếp vào stage LLM bằng Prompt Caching và chắt lọc context ngắn gọn hơn.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng số lượng thread từ `-t 1` lên `-t 32` (sweep qua `llama-bench`)

```
before:  25.4 tok/s (-t 1)
after:   32.5 tok/s (-t 32)
speedup: 1.28×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Em thực hiện sweep thread từ 1 đến 32 trên CPU Intel Core i7-1260P (kiến trúc lai 4 P-cores + 8 E-cores, tổng cộng 12 physical / 16 logical cores). Tốc độ giải mã decode tăng tuyến tính từ 25.4 tok/s lên 29.8 tok/s tại 12 physical cores, sau đó xuất hiện điểm chuyển tiếp (knee) khi tốc độ tăng chậm lại rõ rệt ở 16 logical cores (30.3 tok/s, chỉ tăng ~1.6%) do các E-cores có năng lực tính toán thấp hơn và các luồng bắt đầu chia sẻ tài nguyên bộ nhớ đệm L3 cache.

Khác với các mô hình lớn (như 7B-8B) thường bị sụt giảm tốc độ khi oversubscribe quá số physical cores do nghẽn băng thông bộ nhớ (memory bandwidth saturation), model Qwen3.5 0.8B có dung lượng chỉ 0.50 GB, phần lớn trọng số nằm trọn trong cache và bộ đệm nhanh. Do đó, mức `-t 32` giúp CPU tận dụng triệt để các chu kỳ xử lý rảnh rỗi và cơ chế lập lịch của hệ điều hành để đạt đỉnh 32.5 tok/s (tăng 1.28x) mà chưa bị nghẽn bus bộ nhớ.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (Sweep Quantization ladder `sweep-quant.py`), B3 (Speedup analysis), B4 (Challenge C5: Model nhỏ nhất vẫn hữu ích), B5 (Challenge C8: Semantic Cache `semantic-cache-offline`)

**Numbers:**

```
before:  2.0 tok/s (UD-Q2_K_XL, 0.39 GB)
after:   31.8 tok/s (UD-Q4_K_XL, 0.52 GB)
speedup: 15.9×
```

**Điều này nói lên gì mà deck chưa nói:**

1. **Về Quantization (B2/B4 - C5):** Lý thuyết trong deck nêu rằng giảm bit luôn giúp tăng tốc độ giải mã (decode) do giảm dung lượng bộ nhớ. Tuy nhiên, trên CPU không có kernel INT2 chuyên biệt, chi phí tính toán giải nén (dequantization) của CPU ALU lớn hơn rất nhiều so với băng thông RAM tiết kiệm được (chỉ tiết kiệm 0.13 GB). Tốc độ tại 2-bit bị sụp đổ (2.0 tok/s so với 31.8 tok/s ở 4-bit, chậm hơn gần 16 lần) và chất lượng bị gãy. Mức 4-bit (`UD-Q4_K_XL`) mới chính là điểm cân bằng tối ưu.
2. **Về Semantic Cache (B5 - C8):** Semantic Cache giúp bắt các câu paraphrase cùng ý nghĩa và trả kết quả ở độ trễ 0 ms (tiết kiệm 100% chi phí prefill và decode). Tuy nhiên, rủi ro False Hit / False Miss rất nhạy cảm với threshold similarity khi dùng mean-pooling từ decoder model thay vì embedding model chuyên dụng. Ngoài ra, cần áp dụng Salting per-tenant để tránh timing side-channel attack giữa các người dùng.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
