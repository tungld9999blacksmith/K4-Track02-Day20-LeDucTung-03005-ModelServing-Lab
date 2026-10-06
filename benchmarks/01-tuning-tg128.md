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

## Your explanation (required -- replace this line)

Không có knee rõ rệt — đường cong gần như phẳng. Toàn dải -t 1 → 16 chỉ dao động trong khoảng 31.5–32.4 tok/s (spread 1.03×), nằm trong biên độ nhiễu đo. Peak danh nghĩa ở -t 2 (32.4 tok/s), nhưng -t 4 và -t 16 cũng cùng 32.4, còn bản default theo physical-core (-t 4) đạt tỉ lệ 1.00× so với best → thực tế thay đổi thread count không cải thiện decode trên máy này.

Cơ chế: tg128 đo decode, mà decode bị chặn bởi memory bandwidth, không phải FLOPs. Mỗi token sinh ra phải đọc gần như toàn bộ trọng số (~3 GB bản Q4_K_XL) từ RAM. Trên CPU 4 nhân vật lý này, chỉ cần 1–2 thread đã đủ bão hòa băng thông bộ nhớ; thêm thread sau đó chỉ xếp hàng chờ cùng các memory channel chứ không có thêm bandwidth để dùng → throughput đứng yên. Vì vậy curve phẳng ngay từ -t 2.

Vì sao không tụt ở -t 16 (oversubscribe)? Kỳ vọng từ deck là -t vượt số nhân vật lý sẽ chậm lại do cạnh tranh scheduling. Ở đây không thấy tụt đáng kể, vì công việc đã bị bottleneck ở bộ nhớ chứ không ở compute — các thread thừa phần lớn ngồi chờ RAM (stall) nên overhead context-switch bị che lấp, nằm trong nhiễu. Nếu đo lại nhiều vòng, nhiều khả năng -t 16 sẽ nhỉnh chậm hơn chút, nhưng chênh lệch quá nhỏ để coi là thật.

Kết luận / hành động: trên máy này, thread count không phải đòn bẩy. Bottleneck là memory bandwidth. Muốn tăng decode thật sự phải giảm số byte phải đọc mỗi token (quantization/định dạng khác) hoặc dùng bandwidth cao hơn (GPU/Metal), chứ không phải thêm thread. Tôi giữ LAB_N_THREADS=4 (bằng số nhân vật lý) cho các bước sau vì nó nằm trong nhóm nhanh nhất và tránh oversubscribe.