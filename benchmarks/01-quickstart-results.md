# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3091 | 149 / 219 | 27.5 / 32.0 | 1897 / 2158 / 2158 | 36.4 |
| UD-Q2_K_XL | 2.24 | 3064 | 150 / 208 | 23.2 / 29.4 | 1654 / 1824 / 1824 | 43.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.18x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

**Số đo.** Trên Apple M4 (10 core, 16 GB, Metal offload `ngl=99`), UD-Q2_K_XL decode nhanh hơn UD-Q4_K_XL **1.18x** (TPOT P50 27.5 -> 23.2 ms, tức 36.4 -> 43.1 tok/s) và nhỏ hơn **0.73 GB** (2.97 -> 2.24 GB, giảm khoảng 25%). E2E P50 giảm từ 1897 xuống 1654 ms (khoảng 13%). TTFT gần như không đổi (149 vs 150 ms) vì prompt ngắn nên prefill nhỏ, và 2-bit chỉ giúp phần decode. P95 hai bản gần nhau (TPOT 32.0 vs 29.4 ms) và chỉ có 10 mẫu mỗi bản, nên không nên đọc chênh lệch ở đuôi như một kết luận.

**Vì sao chỉ 1.18x chứ không phải 1.33x?** Decode bị giới hạn bởi memory bandwidth: mỗi token phải đọc lại gần như toàn bộ weights. Nếu chỉ tính số byte thì 2.24/2.97 = 0.75 cho speedup lý tưởng khoảng 1.33x. Thực tế thấp hơn vì còn chi phí dequantize, traffic của KV cache và activation không giảm theo bit, và bản "UD" (dynamic) giữ một số tensor ở độ chính xác cao hơn nên 2-bit không nhỏ đúng 2/4. Đây là giải thích dựa trên cơ chế, tôi chưa tách riêng từng yếu tố bằng phép đo. Số tuyệt đối cũng dao động giữa các lần chạy: một lần chạy trước đó trên cùng máy cho 42.6 vs 51.2 tok/s (1.20x), cùng tỉ lệ nhưng nền cao hơn khoảng 15%, nhiều khả năng do page cache và nhiệt. Bảng ở trên là lần chạy sau, và ảnh `02-bench.png` khớp với bảng này.

**Chất lượng.** Tôi bật cả hai bản (Q4 port 8080, Q2 port 8090), hỏi cùng 4 câu với `temperature=0`, `max_tokens=250`: giải thích vì sao decode bị chặn bởi bandwidth, bài toán tiền thối bằng tiếng Việt, hàm Python `is_palindrome`, và thủ đô của Úc. Cả hai đều ra đáp án đúng ở mọi câu có đáp án kiểm chứng được (tiền thối 14.000đ, Canberra, hàm palindrome cho cùng logic). Khác biệt tôi thấy: (1) với câu giải thích bandwidth, Q2 dài dòng và lặp ý hơn, không giữ đúng "3 câu", còn Q4 gọn hơn; cả hai đều giải thích cơ chế chưa chuẩn (nhấn vào "dữ liệu giữa bộ nhớ và core" thay vì việc đọc lại toàn bộ weights cho mỗi token); (2) ở bài tiền thối, Q2 gán nhãn sai một dòng ("Tổng số tiền khách đưa: 86.000đ" thay vì tổng tiền hàng) dù phép tính vẫn đúng; (3) ở bài palindrome, Q4 ghi nhầm trong docstring là "phân biệt chữ hoa/thường" trong khi code đúng là bỏ qua, nên lỗi diễn đạt có ở cả hai bản. Bộ 4 câu này rất nhỏ nên đây là cảm nhận định tính, không phải đánh giá chất lượng có ý nghĩa thống kê.

**Kết luận: với máy này, 2-bit không đáng dùng.** Máy có 16 GB và Metal nên bản 4-bit nằm gọn trong bộ nhớ; 0.73 GB tiết kiệm được không giải quyết ràng buộc nào. Đổi lại chỉ được thêm khoảng 18% decode speed, trong khi Q2 cho thấy dấu hiệu chất lượng kém hơn một chút ở câu giải thích và một lỗi nhãn ở bài số học. Tôi sẽ chọn Q4 làm mặc định. 2-bit hợp lý khi RAM là ràng buộc cứng (máy 4-8 GB, hoặc cần nhồi thêm nhiều slot/KV cache vào cùng một lượng bộ nhớ), và chỉ sau khi tự kiểm tra chất lượng trên đúng tác vụ của mình.
