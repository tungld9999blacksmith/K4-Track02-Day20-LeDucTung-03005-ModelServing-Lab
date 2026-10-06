# 01 - Measure: latency baseline

Model `Gemma 4 E2B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 70233 | 1146 / 7126 | 35.8 / 64.6 | 3294 / 9811 / 9811 | 28.0 |
| UD-Q2_K_XL | 2.24 | 18676 | 1054 / 8095 | 41.2 / 46.2 | 3511 / 10864 / 10864 | 24.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.15x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead � few cores, no GPU offload � the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**Tóm tắt số đo.** Trên máy tôi (Windows, CPU-only, GPU offload OFF), `UD-Q2_K_XL`
nhỏ hơn `UD-Q4_K_XL` 0.73 GB (2.24 vs 2.97 GB, ~25% nhỏ hơn) nhưng lại decode **chậm
hơn 1.15×**: 24.3 vs 28.0 tok/s, TPOT P50 41.2 vs 35.8 ms. TTFT và E2E của hai bản xấp
xỉ nhau (TTFT P50 1054 vs 1146 ms; E2E P50 3511 vs 3294 ms), chênh lệch nằm trong nhiễu.

**Vì sao 2-bit lại CHẬM hơn — đây là điểm chính.** Ít bit chỉ cho ra decode nhanh hơn
khi tốc độ bị chặn bởi **memory bandwidth** (ít byte phải đọc mỗi token). Máy tôi không
có GPU, ít nhân, nên đang **compute-limited**: mỗi trọng số Q2 phải được **dequantize**
về float trước khi nhân, format nén càng sâu thì bước giải nén càng tốn phép tính. Ở đây
chi phí dequantize của Q2 lớn hơn phần byte nó tiết kiệm được, nên Q2 nhỏ hơn nhưng chạy
chậm hơn — ngược với trực giác "ít bit = nhanh hơn", và đó là kết quả đúng chứ không
phải lỗi đo.

**Chất lượng câu trả lời (serve cả hai, hỏi cùng câu).** Tôi chạy bản 4-bit ở :8080 và
bản 2-bit ở :8090 rồi hỏi y hệt một câu ("Explain what memory bandwidth is and why it
limits LLM token generation speed, in 4-5 sentences"). Bản **Q4 trả lời gọn, đúng, đủ
4-5 câu và kết thúc hoàn chỉnh**. Bản **Q2 rò rỉ phần "nháp" suy luận** (in ra dàn ý
kiểu "The user wants...", "Drafting...", "Self-Correction...") và **không bao giờ cho ra
câu trả lời sạch**, chạm trần max_tokens giữa chừng. Câu hỏi số học thứ hai (60 km trong
45 phút) lặp lại khác biệt đó: Q4 đi thẳng tới phép tính, Q2 lan man hơn. Với cùng
max_tokens, Q2 tiêu token vào phần nháp nên hữu dụng thực tế thấp hơn hẳn.

**Kết luận: trên máy này Q2_K_XL KHÔNG đáng.** Nó vừa chậm hơn (compute-limited), vừa
kém kiểm soát định dạng đầu ra, trong khi chỉ tiết kiệm 0.73 GB RAM. Q2 chỉ hợp lý khi
RAM là ràng buộc cứng — không nạp nổi bản 4-bit. Khi còn nạp được Q4, Q4 thắng cả về tốc
độ lẫn chất lượng.