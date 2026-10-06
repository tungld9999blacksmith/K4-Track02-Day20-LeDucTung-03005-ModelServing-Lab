# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=4` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 37 | 0.65 | 12000 | 21000 | 21000 | 8.1 | 0.0% |
| 50 | 29 | 0.50 | 37000 | 56000 | 57000 | 16.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.78x** (16% of linear) |
| P95 latency | **2.67x** |
| Effective concurrency at 50 users | 16.2 vs `--parallel 4` slots (occupancy/slot ratio 4.05) |

**Saturated.** Throughput delivered only 0.78x for 5x the offered load, and effective concurrency (16.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.78x while P95 moved 2.67x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server bão hoà ở hoặc dưới mức 50 user — thực ra đã quá tải ngay từ 10 user.** Bằng
chứng mạnh nhất: khi offered load tăng **5×** (10 → 50 user), throughput thực nhận
**giảm** từ 0.65 xuống 0.50 RPS (**0.78×**, chỉ ~16% của mức tuyến tính). Throughput
không những không tăng mà còn tụt — đây là dấu hiệu rõ ràng của một hệ thống đã vượt
điểm bão hoà.

**Con số thuyết phục tôi:** effective concurrency ở 50 user = **16.2**, gấp ~4.05 lần số
slot thật (`--parallel 4`). Nghĩa là trung bình có ~16 request "trong hệ thống" nhưng chỉ
4 cái được decode, 12 cái còn lại đang **xếp hàng**. Khớp với `02-server-batching-u50.md`:
busy slots đạt trần 4.00/4 và 46 request bị deferred. Phần vượt quá 4 slot chính là queue
time, và nó hiện ra ở P95: **21s → 56s (2.67×)**.

**Lập luận goodput@SLO.** Throughput chỉ nhích 0.78× trong khi P95 phồng 2.67×: sau bão
hoà, mọi "throughput" mua thêm đều phải trả bằng latency. Giả sử tôi đặt **SLO P95 ≤ 5s**:
ngay cả ở 10 user P95 đã là 21s, nên ở cả hai mức tải, **gần như 0% request đáp ứng SLO**
— goodput thực tế ≈ 0. Hệ thống này không thiếu throughput danh nghĩa, nó thiếu khả năng
phục vụ *trong hạn*.

**Tôi sẽ đổi gì đầu tiên, và vì sao knob đó.** Không phải tăng `--parallel`. Thêm slot
chỉ khiến nhiều request cùng decode, mà decode trên máy này đã nằm trên iGPU Vega 8 yếu,
chia sẻ bus DDR4 (xem `01-tuning-tg128.md`: thêm CPU thread không giúp gì vì decode ở
GPU) — 8 slot sẽ làm *mỗi* token chậm hơn, P95 còn tệ hơn. Hai đòn bẩy thật sự:

1. **Giảm công việc mỗi request** — hạ `LAB_MAX_TOKENS` và/hoặc dùng cấu hình decode
   nhanh hơn. Mỗi request xong nhanh → slot giải phóng sớm → hàng đợi ngắn lại → P95 giảm.
   Đây là cách rẻ nhất để kéo goodput@SLO lên mà không đổi phần cứng.
2. **Admission control / right-size `--parallel`** — chặn bớt tải nhận vào để các request
   được nhận đáp ứng SLO, thay vì để tất cả cùng xếp hàng và cùng vi phạm SLO.

Thay đổi phần cứng (GPU, băng thông cao hơn) sẽ nâng trần thật sự, nhưng nằm ngoài phạm
vi base track — nên trong giới hạn hiện tại, giảm per-request work là knob đầu tiên tôi
chọn.
