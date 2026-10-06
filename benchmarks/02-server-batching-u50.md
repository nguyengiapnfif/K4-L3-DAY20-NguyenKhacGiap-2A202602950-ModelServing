# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.97 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 7816 |

Highest sampled value was **3.97 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak `n_busy_slots_per_decode` là **3.97 / 4** slot (99%), `requests_processing` = 4 và `requests_deferred` lên tới 46. Gauge được đọc trong lúc `make load-50` đang chạy (30 mẫu, mỗi 2 giây), nên đây là bằng chứng scheduler thật sự gộp 4 request vào chung một bước decode, chứ không phục vụ lần lượt (khi đó gauge sẽ quanh 1; ở smoke test một request lẻ gauge đúng bằng 1.00).

**Batching có giúp throughput, nhưng ít hơn kỳ vọng.** Từ `02-server-metrics-u50.csv`, trong cửa sổ có tải `tokens_predicted_total` tăng 3528 token trong 54.7 giây, tức khoảng **64.5 tok/s gộp**, so với 44.0 tok/s của một request đơn lẻ ở smoke test. Bốn slot chỉ cho khoảng 1.47x throughput decode, không gần 4x. Tôi chưa đo riêng nguyên nhân; những ứng viên hợp lý là prefill của các prompt RAG dài chen vào giữa các bước decode, và phần decode vẫn tốn bandwidth/compute cho mỗi slot, nên batching không miễn phí như lý thuyết "đọc weights một lần cho cả batch" gợi ý.

**Đối chiếu với effective concurrency.** Báo cáo batching này được đo trong một lần chạy `make load-50` riêng, chạy trước lần chạy cho ra `02-server-results.md` (tôi chạy lại load test sau đó để chụp screenshot, nên hai file đến từ hai lần chạy 50 user khác nhau trên cùng cấu hình server). `02-server-results.md` báo effective concurrency 27.2 ở 50 user (RPS x avg latency), trong khi gauge của server cho thấy 4 request đang xử lý cộng khoảng 45 request bị defer, tức khoảng 49 request nằm trong hệ thống, gần đúng với 50 user (think time chỉ 0.2-1.5 giây trong khi latency khoảng 30 giây nên gần như ai cũng đang chờ). Con số locust thấp hơn vì chỉ có 56 request hoàn thành trong 60 giây và những request còn dở khi cửa sổ kết thúc không được tính, nên nó là cận dưới. Để biết có bao nhiêu request đang nằm trong hệ thống thì tôi tin gauge của server hơn; effective concurrency vẫn đủ để kết luận rằng tải vượt xa số slot.
