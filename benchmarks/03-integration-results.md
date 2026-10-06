# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 1051.6 | 1051.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 713.8 | 713.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 735.4 | 735.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **833.6** · total **833.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

Trong lần chạy này tôi **không thay STUB nào** trong `labs/03-integrate/pipeline.py`, nên chỉ có phần serving là thật:

| Day | Piece | Real hay stub? |
|:--|:--|:--|
| N16 Cloud/IaC | không dùng trong pipeline này | stub (không có) |
| N17 Data pipeline | tài liệu lấy từ `TOY_DOCS` có sẵn trong script (STUB 1) | stub |
| N18 Lakehouse | không có bảng/lakehouse thật, corpus nằm trong code | stub |
| N19 Vector + features | `retrieve()` dùng keyword overlap, không có embedding server (STUB 2) | stub |
| N20 Serving | `llama-server` (Gemma 4 E2B Q4_K_XL, port 8080) | **real** |

**Stage nào chiếm nhiều nhất?** `llm`: trung bình 833.6 ms trên tổng 833.6 ms (**100%**); `embed` và `retrieve` đều 0.0 ms. Điều này hiển nhiên đúng với cấu hình hiện tại và **không phải một kết luận về RAG nói chung**: không có embedding server nên stage embed bằng 0 theo thiết kế, và retrieval là so khớp từ khoá trên vài tài liệu đồ chơi nên gần như không tốn thời gian. Một embedder thật hoặc một index lớn có thể làm hai stage này khác hẳn, nên tôi không rút ra kết luận nào về chúng từ lần chạy này.

Bên trong stage `llm`, theo dòng `server` của 3 câu hỏi: prefill trung bình khoảng 300 ms (421, 240, 240 ms cho 149, 114, 113 token prompt) và decode khoảng 511 ms (595, 458, 480 ms cho 30, 23, 24 token). Decode chiếm khoảng 61% và prefill khoảng 36% của stage này. Câu đầu tiên có prefill 421 ms cao hơn hai câu sau (240 ms) dù prompt dài hơn chỉ khoảng 30%; tôi chưa kiểm chứng nguyên nhân, có thể là server chưa có gì trong prompt cache.

**Nếu phải giảm latency 2x**, tôi sẽ tấn công stage `llm` vì nó là toàn bộ thời gian. Trong đó, decode chiếm phần lớn nhất nên đòn bẩy tốt nhất là giảm số token sinh ra (câu trả lời ngắn hơn, hạ `max_tokens`), vì mỗi token tốn khoảng 20 ms; chuyển sang bản 2-bit chỉ cho khoảng 1.18x ở bước baseline nên không đủ 2x một mình. Với prefill, giữ system prompt giống nhau từng byte để server dùng lại prefix đã cache sẽ cắt phần prompt lặp lại. Tôi chưa thử các thay đổi này.
