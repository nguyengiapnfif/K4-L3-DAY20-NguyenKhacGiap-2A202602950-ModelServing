# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Khắc Giáp
**MSSV:** 2A202602950
**Cohort:** K4 (L3)
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS (Darwin 25.2.0, arm64)
- **CPU:** Apple M4
- **Cores:** 10 physical / 10 logical (4 Performance + 6 Efficiency)
- **CPU extensions:** NEON
- **RAM:** 16 GB
- **Accelerator:** Apple Metal (Apple M4, offload `ngl=99`)
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** Mac mini của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

`make setup` lần đầu lỗi `CERTIFICATE_VERIFY_FAILED` khi tải llama.cpp, vì Python 3.13 cài từ python.org
trên macOS thiếu bộ chứng chỉ CA. Workaround: đặt `SSL_CERT_FILE` trỏ vào `certifi` (đã có trong
venv) rồi chạy lại `make setup`, lần sau thành công.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3091 | 149 / 219 | 27.5 / 32.0 | 1897 / 2158 / 2158 | 36.4 |
| UD-Q2_K_XL | 2.24 | 3064 | 150 / 208 | 23.2 / 29.4 | 1654 / 1824 / 1824 | 43.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit decode nhanh hơn 1.18x và nhỏ hơn 0.73 GB (~25%), TTFT gần như không đổi. Tôi hỏi cùng 4 câu
trên cả hai server: đáp án đúng ở cả hai, nhưng Q2 dài dòng hơn và có một lỗi gán nhãn ở bài số học.
Với máy 16 GB này tôi không thấy đáng dùng 2-bit (mẫu 4 câu nên chỉ là cảm nhận định tính).

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.90 | 8900 | 15000 | 15000 | 8.4 | 0 |
| 50 | 0.95 | 31000 | 47000 | 48000 | 27.2 | 0 |

- **Offered load tăng 5×, throughput thực tăng:** 1.06×
- **P95 tăng:** 3.13×
- **Effective concurrency ở 50 users:** 27.2 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.97 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hoà ngay từ 10 user: tải tăng 5× nhưng RPS chỉ tăng 1.06×, còn P95 tăng 3.13× (15 s -> 47 s),
và gauge cho thấy 4/4 slot kín cùng ~45 request bị defer. Phần latency tăng thêm là queue time (một
request đơn lẻ xong trong ~1.9 s, ở 50 user P50 là 31 s mà RPS gần như không đổi). Batching chỉ cho ~1.47× throughput
gộp (64.5 vs 44 tok/s), nên tôi sẽ giới hạn số request đồng thời trước (chưa thử), chứ không tăng `--parallel`.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng trong pipeline này | stub |
| N17 Data pipeline | `TOY_DOCS` có sẵn trong script (STUB 1) | stub |
| N18 Lakehouse | corpus nằm trong code, không có bảng thật | stub |
| N19 Vector + features | keyword overlap, không có embedding server (STUB 2) | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 833.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

`llm` chiếm 100% là hiển nhiên với cấu hình này: embed bằng 0 theo thiết kế (không có embedding server) và
retrieve chỉ là so khớp từ khoá trên vài tài liệu đồ chơi, nên đây không phải kết luận về RAG nói chung.
Trong `llm`, decode chiếm ~61% (~511 ms) và prefill ~36% (~300 ms). Để giảm 2× tôi sẽ cắt số token sinh ra
trước (chưa thử); chuyển sang 2-bit chỉ cho ~1.18× nên không đủ một mình.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** (chế độ CPU-only, `-ngl 0`) hạ số thread từ mặc định `-t 10` (= số core physical) xuống `-t 8`

```
before:  32.7 ± 1.9 tok/s   (-t 10, tg128, CPU-only, 5 reps)
after:   47.5 ± 0.2 tok/s   (-t 8,  tg128, CPU-only, 5 reps)
speedup: 1.45×
```

Lưu ý: ở chế độ GPU mặc định (`ngl=99`, `make tune`) knob này gần như không có tác dụng (best -t 10 = 44.6 tok/s,
speedup vs mặc định 1.00×, và 1 thread đã đạt 85%). Số liệu thô ở `benchmarks/01-tuning-tg128-cpu-r5-raw.md`.

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả khác kỳ vọng ở hai điểm. Thứ nhất, với `ngl=99` (GPU Metal) số thread gần như không quan trọng vì
decode chạy trên GPU, CPU thread chỉ điều phối lệnh. Thứ hai, khi ép CPU-only, đỉnh không nằm ở 10 core
"physical" mà ở 6-8 thread: 4 thread 38.8, 6 thread 46.0, 8 thread 47.5, 10 thread 32.7 tok/s. M4 có
4 core Performance + 6 core Efficiency, nên 10 core không đồng đều.

Đoạn tăng rồi phẳng (4 -> 8 thread, chỉ +3% từ 6 lên 8) phù hợp với việc decode bị chặn bởi memory bandwidth
dùng chung chứ không phải FLOPs. Đoạn tụt ở 10 thread (-31% so với 8) tôi giải thích bằng barrier đồng bộ
của ggml: mỗi phép toán chờ thread chậm nhất, và khi dùng hết mọi core thì bất kỳ tiến trình nền nào cũng
làm trễ một thread (tôi thấy `mediaanalysisd` chiếm ~100% một core lúc đo; độ lệch chuẩn ở 10 thread
±1.9, lớn gấp ~10 lần các điểm khác). Với 8 thread còn hai core trống cho hệ điều hành. Điểm 20 thread không ra
kết quả sau 240 s (lần trước >1800 s), một mẫu duy nhất nhưng cho thấy oversubscribe có thể rất tệ.

Đây là giải thích hợp lý với dữ liệu, nhưng tôi chưa chạy lại khi tiến trình nền đó tắt, và chưa tách riêng
ảnh hưởng của core P so với core E bằng cách ghim thread, nên cơ chế barrier/nền vẫn là giả thuyết.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

Không làm bonus.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Cùng một knob (số thread) cho 1.00× khi chạy GPU nhưng 1.45× khi chạy CPU-only, và một bản CPU-only
đã chỉnh đúng thread (47.5 tok/s) ngang với bản GPU ở decode (44.6 tok/s).

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Claude Code (Claude Sonnet 5.5) được dùng để: chạy các lệnh
`make` của lab, chạy thêm sweep `llama-bench` CPU-only, đọc log/lỗi (lỗi SSL khi tải runtime), và soạn nháp các
đoạn giải thích trong `benchmarks/*.md` và REFLECTION từ số liệu thật của máy. Số liệu do script/`llama-bench`
sinh ra trên máy của tôi, không sửa tay. Các screenshot do tôi tự chụp từ terminal của mình.
