# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **4 physical � 8 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 31.5 | 97% |
| 2 | 32.4 | 100% |
| 4 | 32.4 | 100% |
| 8 | 31.8 | 98% |
| 16 | 32.3 | 100% |

**Best**: `-t 2` at 32.4 tok/s
**Slowest tested**: `-t 1` at 31.5 tok/s (1.03x spread)
**Against the physical-core default** (`-t 4`, 32.4 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Your explanation

Không có knee rõ rệt — đường cong gần như phẳng. Toàn dải -t 1 → 16 chỉ dao động trong khoảng 31.5–32.4 tok/s (spread 1.03×), nằm trong biên độ nhiễu đo. Peak danh nghĩa ở -t 2 (32.4 tok/s), nhưng -t 4 và -t 16 cũng cùng 32.4, còn bản default theo physical-core (-t 4) đạt tỉ lệ 1.00× so với best → thực tế thay đổi thread count không cải thiện decode trên máy này.

Cơ chế — và đây là điểm mấu chốt: benchmark này chạy với `ngl=99`, tức decode được
**offload lên GPU (Vulkan device 0 = AMD Vega 8 iGPU)**, không chạy trên CPU. Cờ `-t`
chỉ điều khiển số **thread CPU**, mà khi decode nằm trên GPU thì CPU gần như chỉ lo
sampling và điều phối — nên thay đổi `-t` gần như không chạm tới phần việc nặng. Đó là lý
do curve phẳng ngay từ `-t 1`: nếu decode chạy trên CPU thì `-t 1` thường chậm hơn `-t 4`
vài lần, chứ không phải chỉ 3% như ở đây.

Vì sao không tụt ở -t 16 (oversubscribe)? Kỳ vọng từ deck (giả định decode trên CPU) là
vượt số nhân vật lý sẽ chậm lại do cạnh tranh scheduling. Ở đây không thấy tụt vì công
việc nặng không nằm trên CPU threads — các thread thừa gần như rảnh, overhead context-
switch không đáng kể so với thời gian GPU decode. Kết quả phẳng này **mâu thuẫn với hình
dạng kỳ vọng trong deck**, và nguyên nhân đúng là: bottleneck không phải CPU thread.

Lưu ý thêm về bản chất Vega 8: đây là iGPU chia sẻ chung bus DDR4 với CPU, không có VRAM
tốc độ cao riêng, nên decode vẫn bị giới hạn bởi băng thông bộ nhớ hệ thống — chỉ là giới
hạn đó không thay đổi theo số CPU thread.

Kết luận / hành động: trên máy này, thread count **không phải** đòn bẩy vì decode đã nằm
trên iGPU. Muốn tăng decode thật sự phải giảm số byte/phép tính mỗi token (quantization
khác) hoặc dùng accelerator có bandwidth cao hơn (ví dụ ép dùng GTX 1050 qua
`GGML_VK_VISIBLE_DEVICES`, hoặc build CUDA). Tôi giữ `LAB_N_THREADS=4` (bằng số nhân vật
lý) cho các bước sau vì nó nằm trong nhóm nhanh nhất và tránh oversubscribe vô ích.