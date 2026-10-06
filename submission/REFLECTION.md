# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Lê Đức Tùng
**MSSV:** 03005
**Cohort:** K4 — Track 02
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** AMD Ryzen 5 3550H with Radeon Vega Mobile Gfx
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2
- **RAM:** 13.9 GB
- **Accelerator:** 2 thiết bị Vulkan — Vulkan0: AMD Radeon Vega 8 (iGPU, ~8.5 GB RAM chia sẻ) · Vulkan1: NVIDIA GTX 1050 (3 GB). Runtime dùng bản Vulkan; **decode thực tế offload lên Vega 8 iGPU** (device 0 mặc định), GTX 1050 không được dùng.
- **llama.cpp asset đã tải:** llama.cpp build `b10488` (prebuilt, backend Vulkan + CPU)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) — từ `models/active.json`

**Chạy ở đâu:** laptop của tôi (local, đủ RAM cho model mặc định).

**Setup story** (≤ 80 chữ): Lab chạy thẳng, không phải workaround gì. Điều đáng chú ý là
probe thấy 2 GPU Vulkan; llama.cpp mặc định chọn device 0 (Vega 8 iGPU) nên `ngl=99`
offload lên iGPU chứ không phải GTX 1050. Model 2.97 GB nằm gọn trong RAM chia sẻ của
iGPU nên offload thành công.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 70233 | 1146 / 7126 | 35.8 / 64.6 | 3294 / 9811 / 9811 | 28.0 |
| UD-Q2_K_XL | 2.24 | 18676 | 1054 / 8095 | 41.2 / 46.2 | 3511 / 10864 / 10864 | 24.3 |

**Quan sát** (≤ 60 chữ): Q2 nhỏ hơn ~25% (0.73 GB) nhưng decode **chậm hơn 1.15×** (24.3
vs 28.0 tok/s) — compute-bound ở khâu dequantize trên iGPU Vega 8 yếu, không phải
bandwidth-bound. Về chất lượng: hỏi cùng câu, Q4 trả lời gọn/đúng/đủ; Q2 rò rỉ phần
"nháp" suy luận, không ra câu trả lời sạch. Kết luận: Q2 **không đáng** trừ khi bộ nhớ là
ràng buộc cứng.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.65 | 12000 | 21000 | 21000 | 8.1 | 0.0% |
| 50 | 0.50 | 37000 | 56000 | 57000 | 16.2 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.78× (thực ra **giảm**)
- **P95 tăng:** 2.67×
- **Effective concurrency ở 50 users:** 16.2 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 4.00 / 4 slots

**Saturation reading** (≤ 80 chữ): Server đã bão hoà ở/dưới 50 user. Bằng chứng thuyết
phục nhất: tải tăng 5× nhưng throughput **giảm** còn 0.78×, còn effective concurrency
16.2 ≈ 4× số slot — tức ~12 request luôn xếp hàng. Phần latency thêm là **queue time**
(biết vì P95 tăng 2.67× trong khi throughput không tăng, và busy-slots đã kịch trần 4/4
với 46 request deferred). Để nâng goodput@SLO tôi sẽ **giảm per-request work** (hạ
max_tokens) hoặc admission control trước — **không** tăng `--parallel`, vì decode đã nằm
trên iGPU yếu, thêm slot chỉ làm mỗi token chậm hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | stub (không dựng) |
| N17 Data pipeline | TOY_DOCS cứng | stub |
| N18 Lakehouse | — | stub |
| N19 Vector + features | keyword overlap | stub (chưa dùng embedding/vector search; real được qua bonus C9) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 3876.0 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck là stage **llm (100%)** — đúng kỳ vọng, vì
embed/retrieve đang stub nên gần như tức thời, còn llm là bước thật đắt (prefill +
decode). Giảm 2× latency thì tấn công vào llm: bật prompt caching cho context lặp, hạ số
token, hoặc dùng accelerator mạnh hơn iGPU Vega 8. Tối ưu retrieval vô nghĩa (Amdahl).

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** sweep thread count `-t` qua `make tune` (1 → 2 → 4 → 8 → 16), tìm cấu hình
decode tốt nhất.

```
before:  -t 1   → 31.5 tok/s  (tg128)
after:   -t 2   → 32.4 tok/s  (best)
speedup: 1.03×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Con số thì nhỏ (1.03×), nhưng **cái nó tiết lộ mới là điều quan trọng nhất**: đường cong
gần như **phẳng** từ `-t 1` tới `-t 16`. Nếu decode chạy trên CPU như deck giả định, thì
`-t 1` phải chậm hơn `-t 4` vài lần (decode CPU scale gần tuyến tính tới số nhân vật lý
rồi mới bão hoà). Việc tăng gấp 4 lần số thread chỉ đổi được 3% chứng tỏ **phần việc nặng
không nằm trên CPU threads**. Khớp với header benchmark (`ngl=99`) và với probe (2 thiết
bị Vulkan): decode thực ra được **offload lên GPU — cụ thể là iGPU AMD Vega 8** (Vulkan
device 0 mặc định). Cờ `-t` chỉ điều khiển thread CPU, mà CPU lúc này gần như chỉ lo
sampling/điều phối, nên không chạm tới thời gian GPU decode.

Đây là một kết quả **mâu thuẫn với hình dạng kỳ vọng trong deck**, và giá trị nằm ở chỗ
giải thích đúng *vì sao*. Hệ quả thực hành: trên máy này, thread count **không phải** đòn
bẩy — chỉnh `-t` là vô ích. Muốn tăng decode thật sự phải hành động ở đúng tầng bottleneck:
hoặc giảm byte/phép tính mỗi token (đổi quantization — nhưng như §2 cho thấy Q2 còn chậm
hơn vì dequant tốn hơn trên iGPU yếu), hoặc ép dùng accelerator mạnh hơn (GTX 1050 qua
`GGML_VK_VISIBLE_DEVICES`, hoặc build CUDA). Bài học: luôn **xác minh tầng nào thực sự
làm việc** trước khi tối ưu — một sweep "không có tác dụng" lại là dữ liệu chẩn đoán có
giá trị nhất.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _(chưa làm bonus)_

**Numbers:**

```
(để trống)
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Decode không chạy trên CPU như tôi tưởng, cũng không chạy trên GPU rời GTX 1050, mà trên
**iGPU Vega 8** tích hợp — và chính một thí nghiệm "không cho kết quả gì" (sweep thread
phẳng) lại là thứ phát hiện ra điều đó.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

- Tôi dùng Claude để hiểu bài lab và các yêu cầu của bài lab, chỉnh sửa lại nhưng phần code để tương thích với máy local
- Tôi dung Claude để giải thích các khái niệm, yêu cầu của từng phần
- Tôi xin cam đoan tôi dùng AI cho mục đích học tập, và nâng cao kiến thức.
