# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Hoàng Duy Anh
**MSSV:** 2A202602558
**Cohort:** K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** Intel Core i5-12450HX (12th Gen)
- **Cores:** 8 physical / 12 logical
- **CPU extensions:** AVX2 (llama.cpp tự nhận; không ghi trong hardware.json)
- **RAM:** 19.7 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 6GB Laptop GPU (CUDA/Vulkan), nhưng mọi số đo chạy CPU-only (`ngl=0`)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip (+ cudart-llama-bin-win-cuda-12.4-x64.zip)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

`lab.ps1` lỗi cú pháp PowerShell và console cp1252 lỗi Unicode, nên mình sửa `lab.ps1`,
`detect-hardware.py`, `lib/labkit.py`. Tải model bằng `huggingface_hub` bị treo ở 0 MB/s,
nên mình chuyển sang `curl.exe -C -` (~23 MB/s), rồi chạy `download-model.py
--skip-download` để ghi `models/active.json`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3118 | 227 / 318 | 43.8 / 57.2 | 2980 / 3921 / 3921 | 22.8 |
| UD-Q2_K_XL | 2.24 | 2546 | 374 / 466 | 39.2 / 41.5 | 2843 / 2926 / 2926 | 25.5 |

**Quan sát** (≤ 60 chữ): 2-bit chỉ nhanh hơn 1.12× ở decode (25.5 vs 22.8 tok/s) và nhỏ
hơn 0.73 GB, nhưng TTFT chậm hơn (374 vs 227 ms). Vì vậy **không đáng** trên máy này. Mình
chưa so chất lượng câu trả lời của hai bản.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.52 | 15000 | 32000 | 35000 | 8.9 | 0.0% |
| 50 | 0.79 | 32000 | 53000 | 58000 | 23.5 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.50×
- **P95 tăng:** 1.66×
- **Effective concurrency ở 50 users:** 23.5 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.93 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hoà ở hoặc dưới 10 user. Tăng tải 5× chỉ tăng throughput 1.50× (30% so với
tuyến tính), trong khi P95 tăng 1.66×. Số thuyết phục nhất là effective concurrency theo
Little's Law: 23.5 request đang in-flight so với 4 slot. Gauge của server khớp:
`n_busy_slots_per_decode` = 3.93/4 và `requests_deferred` ≈ 45. Phần latency tăng thêm là
queue time, không phải compute time: 4 request đang decode còn ~45 request chờ slot. Knob
đầu tiên mình đổi là giảm `max_tokens`/độ dài output để mỗi request ngắn hơn, vì CPU-only
bị chặn bởi memory bandwidth nên tăng `--parallel` không tăng tổng tok/s.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng, chạy trên laptop | stub |
| N17 Data pipeline | list `TOY_DOCS` có sẵn trong `pipeline.py` | stub |
| N18 Lakehouse | không có | stub |
| N19 Vector + features | keyword-overlap retrieval (không có embedding server) | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 4589.2 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck là LLM (100%), đúng như kỳ vọng vì retrieval chỉ là stub nên gần như miễn phí.
Để giảm 2× mình tấn công stage llm: offload layer lên RTX 3050 bằng `-ngl` và gửi ít
context hơn (1-2 chunk thay vì 3), vì prefill chiếm khoảng một nửa thời gian LLM.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** đổi số thread của llama.cpp từ `-t 8` (số core vật lý, mặc định) lên `-t 12`

```
before:  17.2 tok/s (-t 8)
after:   18.1 tok/s (-t 12)
speedup: 1.05×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Cải thiện thật sự nhỏ (1.05×, gần mức nhiễu đo). Đường cong cho thấy knob thật sự quan
trọng ở hai đầu: từ 1 lên 4 thread, tok/s tăng từ 8.1 lên 17.3 (2.1×), nhưng từ 4 đến 12
thread gần như phẳng (17.3, 17.2, 18.1), rồi rơi còn 12.5 tok/s ở 24 thread.

Cơ chế: decode đọc toàn bộ trọng số từ RAM cho mỗi token, nên bị chặn bởi memory
bandwidth chứ không phải FLOPs. Một thread không đủ để làm đầy memory controller, nhưng
khoảng 4 thread thì đủ; thêm thread sau đó không có thêm bandwidth để dùng. Ở 24 thread
(2× số thread logic) các thread bị OS chia time-slice. Llama.cpp đồng bộ các thread sau
mỗi layer, nên một thread bị descheduled sẽ làm cả nhóm đứng chờ, và chúng còn tranh cache
với nhau.

Kết quả này một phần khác kỳ vọng "đỉnh ở số core vật lý": 12 thread nhỉnh hơn 8, nhưng
chỉ 5%, nên mình coi đó là phẳng chứ không phải đỉnh thật. Bài học: trên máy này số thread
chỉ quan trọng khi quá ít hoặc quá nhiều, còn từ 4 đến 12 thì không đổi nhiều.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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

Dùng GitHub Copilot (AI assistant trong VS Code) để sửa lỗi môi trường Windows, chạy các
lệnh của lab và soạn phần nhận xét từ số liệu thật. Mình đã đọc lại các số liệu trong
báo cáo.
