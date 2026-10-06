# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` � `--parallel 4` � 14 samples over
60s at 2.0s intervals � raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 4.00 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a � not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4096 |

Highest sampled value was **4.00 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width đo được là **4.00/4 slots (100%)**: scheduler nhồi đủ 4 request đồng
thời vào mỗi bước decode, đúng bằng `--parallel 4`. Đây là bằng chứng continuous
batching chạy thật — server **không** phục vụ tuần tự từng request. `requests_processing`
đạt 4 (kín slot) và `requests_deferred = 46` cho thấy 50 user vượt xa 4 slot nên 46
request phải xếp hàng chờ.

**So với `02-server-results.md`:** effective concurrency (Little's Law) ở 50 user là
**16.2**, trong khi peak busy slots chỉ **4.00**. Hai số **không mâu thuẫn** vì đo hai
thứ khác nhau:

- **Busy slots (4.00)** = concurrency *bên trong engine*, bị chặn cứng ở số slot
  (`--parallel 4`). Đo trực tiếp phía server.
- **Effective concurrency (16.2)** = RPS × average latency, tính *cả* request đang xếp
  hàng chờ slot, không chỉ request đang chạy. Đây là "occupancy", vượt số slot là hợp lệ.

Phần chênh lệch (16.2 so với 4, tỉ lệ ~4.05) chính là **queue time** — 46 request bị
deferred phải chờ, và thời gian chờ đó nằm trong P95 (56s) của bước load-50.

**Tôi tin số nào?** Tùy câu hỏi: busy slots (4.00) để khẳng định "engine có batch không"
(đo trực tiếp, đáng tin cho câu đó); effective concurrency (16.2) để trả lời "hệ thống
đang gánh bao nhiêu" (phản ánh cả hàng đợi). Kết hợp cả hai: server đã bão hoà ở đúng 4
slot, mọi tải thêm chỉ làm dài hàng đợi chứ không tăng throughput.
