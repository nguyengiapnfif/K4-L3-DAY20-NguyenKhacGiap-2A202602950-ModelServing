# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **10 physical · 10 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 38.0 | 85% |
| 5 | 42.8 | 96% |
| 10 | 44.6 | 100% |
| 20 | 38.3 | 86% |

**Best**: `-t 10` at 44.6 tok/s
**Slowest tested**: `-t 1` at 38.0 tok/s (1.17x spread)
**Against the physical-core default** (`-t 10`, 44.6 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=10 make bench
```

## Your explanation

Bảng ở trên do `make tune` sinh ra với `ngl=99`, tức toàn bộ layer chạy trên GPU Metal của M4. Vì kết quả gần như phẳng nên tôi chạy thêm một sweep **CPU-only** (`-ngl 0`) để xem số thread thật sự ảnh hưởng thế nào. Số liệu thô nằm ở `01-tuning-tg128-cpu-raw.md` (2 lần lặp, lưới 1-10 thread) và `01-tuning-tg128-cpu-r5-raw.md` (5 lần lặp). Tôi chạy `llama-bench` trực tiếp chứ không qua `tune.py` vì lưới mặc định của script có điểm `-t 20` chạy quá 1800 giây rồi timeout và làm script crash.

**1. Khi chạy GPU (`ngl=99`): không có knee.** Chỉ 1 thread đã đạt 85% mức tốt nhất (38.0 vs 44.6 tok/s), 5 thread đạt 96%, và thread tốt nhất (10) đúng bằng mặc định nên speedup là 1.00x. Lý do hợp lý: phần decode chạy trên GPU, CPU thread chỉ chuẩn bị và gửi lệnh, nên thêm thread không thêm công việc hữu ích. Đến 20 thread (gấp đôi số core) thì tụt còn 86%, tức các thread thừa chỉ tốn công điều phối.

**2. Khi chạy CPU-only: knee nằm ở 6-8 thread, không phải ở 10.** Với 5 lần lặp: 4 thread 38.8, 6 thread 46.0, 8 thread **47.5**, 10 thread 32.7 ± 1.9 tok/s. Từ 4 lên 6 thread tăng 19%, từ 6 lên 8 chỉ tăng 3% (đã gần phẳng), rồi 10 thread **tụt 31%** so với 8. Đỉnh nằm dưới số core "physical" mà `make probe` báo (10). Máy này là M4 có **4 core Performance + 6 core Efficiency** (`sysctl hw.perflevel*`), nên 10 core không đồng đều: các thread đầu tiên vào core nhanh, phần còn lại vào core chậm hơn.

**Cơ chế đoán được (chưa kiểm chứng riêng từng yếu tố):**
- *Đoạn tăng rồi phẳng (4 -> 8)*: decode bị giới hạn bởi memory bandwidth chứ không phải FLOPs. Khi thêm thread đến một mức nào đó, băng thông bộ nhớ thống nhất (unified memory) đã bão hoà nên thêm thread gần như không thêm tốc độ.
- *Đoạn tụt ở 10 thread*: llama.cpp đồng bộ các thread bằng barrier sau mỗi phép toán, nên cả bước decode chờ thread chậm nhất. Khi dùng hết mọi core, bất kỳ tiến trình nền nào cũng giành mất một core và làm một thread bị trễ. Tôi thực sự quan sát thấy `mediaanalysisd` của macOS chiếm khoảng 100% một core trong lúc đo, và độ lệch chuẩn của điểm 10 thread (±1.9, lần 2 lặp là ±6.2) lớn gấp 10 lần các điểm khác (±0.2). Với 8 thread còn 2 core trống cho hệ điều hành. Đây là giải thích phù hợp với dữ liệu nhưng tôi chưa chạy lại khi tiến trình đó tắt để xác nhận.
- *20 thread*: không ra kết quả sau 240 giây (và lần trước quá 1800 giây) trong khi các điểm khác xong trong vài giây. Một mẫu duy nhất, nhưng cho thấy oversubscribe khi các thread chờ nhau tại barrier có thể rất thảm hoạ, tệ hơn nhiều so với "chỉ giảm nhẹ".

**3. Điều này nói gì về lab.** Cùng một knob (số thread) có tác dụng 1.00x khi chạy GPU và có tác dụng lớn (8 thread nhanh hơn 10 thread 1.45x: 47.5 vs 32.7 tok/s) khi chạy CPU-only. Nên "số thread tốt nhất" chỉ có nghĩa khi biết decode đang chạy ở đâu. Cũng đáng chú ý là bản CPU-only tốt nhất (47.5 tok/s) tương đương bản GPU (44.6 tok/s) ở decode, điều này phù hợp với việc decode bị chặn bởi bandwidth dùng chung một bộ nhớ, chứ không phải bởi lượng tính toán.

**Giới hạn của kết quả:** đo trên một máy, tiến trình nền không kiểm soát được, mẫu `-t 20` chỉ có một lần, và lần chạy CPU-only đầu tiên của `tune.py` (1/5/10 thread: 18.2/30.7/22.0 tok/s) nhiễu hơn đáng kể so với lần 5 lặp, nên tôi chỉ dùng lần 5 lặp để kết luận.
