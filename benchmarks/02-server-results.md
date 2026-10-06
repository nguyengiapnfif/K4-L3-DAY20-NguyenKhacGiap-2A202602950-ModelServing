# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 53 | 0.90 | 8900 | 15000 | 15000 | 8.4 | 0.0% |
| 50 | 56 | 0.95 | 31000 | 47000 | 48000 | 27.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.06x** (21% of linear) |
| P95 latency | **3.13x** |
| Effective concurrency at 50 users | 27.2 vs `--parallel 4` slots (occupancy/slot ratio 6.80) |

**Saturated.** Throughput delivered only 1.06x for 5x the offered load, and effective concurrency (27.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.06x while P95 moved 3.13x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server đã bão hoà ngay từ 10 user, không phải chỉ ở 50.** Con số thuyết phục nhất là throughput: tăng tải 5x chỉ cho **1.06x** RPS (0.90 -> 0.95), tức 21% so với tuyến tính, trong khi P95 tăng **3.13x** (15 s -> 47 s). Effective concurrency đã là 8.4 ở 10 user, gấp đôi 4 slot, và ở 50 user gauge của server cho thấy 3.97/4 slot bận cùng khoảng 45 request bị defer (xem `02-server-batching-u50.md`). Nghĩa là cả 4 slot luôn kín và mọi request thêm vào chỉ đứng chờ; phần latency tăng thêm là **queue time**, không phải compute time. Bằng chứng gián tiếp: một request đơn lẻ xong trong khoảng 1.9 giây (bench ở bước baseline, 64 token), trong khi ở 50 user P50 là 31 giây, gấp khoảng 16 lần, mà RPS gần như không đổi.

**Goodput@SLO.** Nếu chọn SLO là P95 <= 10 giây thì ngay ở 10 user server đã vượt (P95 = 15 s), và ở 50 user thì vượt rất xa (P95 = 47 s, P50 = 31 s nên quá nửa số request nằm ngoài SLO). Nghĩa là gần như toàn bộ phần throughput mua thêm được (1.06x) đều không nằm trong SLO. Tôi chưa tính chính xác tỉ lệ request dưới 10 giây vì các percentile locust chỉ là xấp xỉ.

**Knob đổi trước (chưa thử, đây là suy luận từ số liệu).** Tôi sẽ không tăng `--parallel` trước, vì 4 slot đã kín mà batching chỉ cho khoảng 1.47x throughput gộp (64.5 vs 44 tok/s, xem `02-server-batching-u50.md`), tức tài nguyên tính toán dùng chung mới là giới hạn: thêm slot chỉ chia mỏng cùng một lượng compute và làm mỗi request chậm hơn. Thay vào đó tôi sẽ giới hạn số request đồng thời hoặc độ sâu hàng đợi (admission control) quanh mức bão hoà để giữ P95 trong SLO, rồi mới giảm công việc mỗi request (hạ `max_tokens`, hoặc dùng bản 2-bit vốn nhanh hơn 1.18x ở bước baseline). Thí nghiệm đề xuất trong hướng dẫn, so `LAB_PARALLEL=1` với 4 slot, tôi chưa chạy nên chưa biết slot ảnh hưởng RPS hay P95 nhiều hơn.

**Giới hạn:** chỉ 53-56 request mỗi lần chạy (dưới ngưỡng tin cậy tốt), một lần chạy mỗi mức. Số tuyệt đối dao động giữa các lần chạy (tôi đã chạy cùng cấu hình hơn một lần và RPS lệch cỡ 20%; máy còn có tiến trình nền không kiểm soát được, đã thấy `mediaanalysisd` chiếm một core ở bước tune), nên chỉ nên tin vào kết luận định tính về bão hoà, không tin vào từng con số tuyệt đối.
