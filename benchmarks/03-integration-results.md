# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.4 | 4258.4 | 4258.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 3680.2 | 3680.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3689.5 | 3689.6 |

Mean per stage (ms): embed **0.0** � retrieve **0.2** �
llm **3876.0** � total **3876.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

Khai báo trung thực:

- **N16 Cloud/IaC — stub.** Không dựng trong lab này.
- **N17 Data pipeline — stub.** Không có ingestion/ETL thật; dùng `TOY_DOCS` cứng.
- **N18 Lakehouse — stub.** Không có lưu trữ/bảng thật.
- **N19 Vector + features — stub.** `embed()` không gọi embedding server (chạy
  `keyword overlap fallback`), `retrieve()` là so khớp từ khoá chứ không phải vector
  search thật. Có thể làm real bằng bonus C9 (`make serve-embed`).
- **N20 Serving — real.** `llama-server` phục vụ thật qua `/v1/chat/completions`; latency
  llm là số đo thật từ model.

**Stage nào thống trị?** `llm` chiếm **100%** tổng latency (mean llm 3876.0 ms so với
retrieve 0.2 ms và embed 0.0 ms). **Đúng như kỳ vọng:** embed/retrieve đang stub nên gần
như tức thời, trong khi llm là bước thật duy nhất và vốn đắt (prefill + decode). Kể cả
khi N19 là vector search thật, một truy vấn vector trên vài nghìn tài liệu vẫn chỉ tốn
mili-giây, nên llm gần như chắc chắn vẫn là stage nặng nhất.

**Muốn giảm latency pipeline 2× thì tấn công vào đâu?** Vào **stage llm**, vì nó là 100%
chi phí — tối ưu embed/retrieve không đổi được gì đáng kể (định luật Amdahl). Cụ thể:
(1) giảm số token sinh / prompt ngắn hơn; (2) bật **prompt caching** để không prefill lại
phần context lặp giữa các query; (3) decode nhanh hơn bằng accelerator mạnh hơn — vì
như `01-tuning-tg128.md` cho thấy, decode đang chạy trên iGPU Vega 8 yếu (chia sẻ DDR4),
đổi sang GPU rời có bandwidth cao sẽ cắt phần lớn thời gian llm.
