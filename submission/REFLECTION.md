# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Đỗ Thái Dương
**Cohort:** Cohort 3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** 12th Gen Intel(R) Core(TM) i5-12450HX
- **Cores:** 8 physical / 12 logical
- **CPU extensions:** not reported by the probe on this Windows build (flag
  detection reads `/proc/cpuinfo`, unavailable here) — the i5-12450HX (Alder Lake)
  supports AVX2 by spec
- **RAM:** 19.7 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 6GB Laptop GPU (CUDA backend), Vulkan also present
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` (+ `cudart-llama-bin-win-cuda-12.4-x64.zip`)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (Windows 11, RTX 3050)

**Setup story** (≤ 80 chữ): Chạy đúng theo GUIDE.md, không fail bước nào. `make setup`
tự nhận diện CUDA và tải bản `bin-win-cuda-12.4-x64` thay vì bản CPU, kèm CUDA
runtime DLLs riêng (~640 MB tổng cho runtime). Toàn bộ base track chạy với GPU
offload (`ngl=99`) chứ không phải CPU-only như ví dụ mẫu trong GUIDE.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 10007 | 74 / 264 | 14.0 / 14.5 | 963 / 1176 / 1176 | 71.2 |
| UD-Q2_K_XL | 2.24 | 4647 | 73 / 207 | 12.2 / 12.6 | 839 / 974 / 974 | 82.1 |

**Quan sát** (≤ 60 chữ): Q2_K_XL nhanh hơn 1.15× (82.1 vs 71.2 tok/s), nhẹ hơn 0.73 GB,
load nhanh hơn 2.2×. Test cùng câu hỏi kỹ thuật trên cả hai (`make serve` vs
`serve.py --compare --port 8090`): cả hai trả lời mạch lạc, đúng cơ chế, không khác
biệt rõ rệt. Đáng dùng khi VRAM là ràng buộc (6 GB card); không đáng cho reasoning
đa bước.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.70 | 2600 | 4300 | 5900 | 7.5 | 0.0% |
| 50 | 2.78 | 15000 | 19000 | 19000 | 39.6 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.03×
- **P95 tăng:** 4.42×
- **Effective concurrency ở 50 users:** 39.6 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.97 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà ở/trước 50 users. Bằng chứng:
throughput chỉ tăng 1.03× cho 5× tải, `n_busy_slots_per_decode` đạt 3.97/4 (99%)
liên tục, `requests_deferred` ~40-45 suốt 60s. P95 tăng 4.42× trong khi RPS gần
như đứng yên → phần trễ thêm là **queue time**, không phải compute (TPOT không đổi
~12-14ms/token). Muốn nâng goodput@SLO, tôi sẽ tăng `--parallel` (4→8) trước —
bottleneck là số slot, không phải tốc độ mỗi token — miễn VRAM (6 GB) còn đủ cho
KV-cache thêm.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | none wired | stub |
| N17 Data pipeline | fixed 3-doc corpus | stub |
| N18 Lakehouse | in-memory list | stub |
| N19 Vector + features | keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.7 ms
- llm: 2803.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Đúng như kỳ vọng — embed/retrieve là stub keyword-matching
trên 3 tài liệu nên gần như 0ms, chỉ llm là việc thật. Muốn giảm latency pipeline 2×,
tôi sẽ tấn công stage llm: giảm `max_tokens` hoặc rút ngắn context đưa vào, vì
TPOT đã sát ngưỡng memory-bandwidth của GPU, không còn nhiều dư địa tune server.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** GPU offload — `-ngl 0` (CPU-only) → `-ngl 99` (full GPU offload trên RTX 3050)

```
before:  22.83 tok/s  (llama-bench tg128, -ngl 0, -t 8)
after:   75.8 tok/s   (llama-bench tg128, -ngl 99, -t 8, from make tune)
speedup: 3.3x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

`make tune` (thread sweep, 1→24 threads) cho kết quả **hoàn toàn phẳng**: 75.0 →
75.8 tok/s, spread chỉ 1.01×. Điều này khác kỳ vọng trong deck (peak ở physical
core count, tụt khi vượt do P/E-core barrier sync) — lý do là `ngl=99` đã offload
toàn bộ layer lên GPU, nên `-t` chỉ còn điều khiển một phần việc rất nhỏ trên CPU
(sampling, KV-cache bookkeeping) không bao giờ là bottleneck. Vì vậy tôi đo thêm
một knob khác để tìm ra thay đổi thật sự lớn: chạy cùng model, cùng `-t 8`, nhưng
`-ngl 0` (ép CPU-only). Kết quả 22.83 tok/s — chậm hơn bản GPU 3.3×.

Cơ chế: TPOT bị chặn bởi memory bandwidth ở cả hai trường hợp — mỗi token sinh ra
đòi hỏi đọc lại toàn bộ ~3 GB trọng số. Với `-ngl 0`, việc đọc đó đi qua RAM hệ
thống (dual-channel DDR, băng thông thấp hơn); với `-ngl 99`, cùng việc đọc đó xảy
ra trên VRAM GDDR6 của RTX 3050, băng thông cao hơn nhiều lần. Đây không phải
compute nhanh hơn (matmul đơn giản vẫn vậy) mà là **tier bộ nhớ nhanh hơn** —
đúng bản chất TPOT = memory-bandwidth-bound, chỉ khác là quantization (đổi kích
thước trọng số cần đọc) và GPU offload (đổi băng thông đọc) đều tác động vào cùng
một biến số, còn thread count thì không, một khi GPU offload đã bật.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 — `make sweep-ctx` (context-length prefill sweep, 256→8192 tokens)

**Numbers:**

```
before:  240.1 ms TTFT contribution  @ 256 prompt tokens
after:   3181.3 ms TTFT contribution @ 8192 prompt tokens
growth:  13.2x latency for 32x more tokens (0.41x of linear — sub-linear, not quadratic)
```

**Điều này nói lên gì mà deck chưa nói:**

Deck dự đoán prefill sẽ bùng nổ O(N²) khi context dài. Trên máy này (RTX 3050,
`ngl=99`, model 2B-class), điều đó **không xảy ra** trong dải 256-8192 token —
prefill throughput còn *tăng* theo context (1066 → 2575 tok/s) vì GPU tận dụng
batch matmul lớn hơn hiệu quả hơn, và ở N≤8192 trên model nhỏ, phần linear
projection/MLP (O(N)) vẫn lấn át attention (O(N²)). Nhưng TTFT tuyệt đối vẫn tăng
13.2× — nghĩa là dù không quadratic, 3.2 giây chờ token đầu tiên ở 8192 token vẫn
là trải nghiệm tệ. Bài học thực tế cho RAG: ngân sách context không nên đặt theo
"chỗ đường cong bẻ cong" (chưa thấy trên máy này) mà nên đặt theo SLO TTFT chấp
nhận được — với SLO ~1s, ngân sách hợp lý trên máy này là ~1-2K token retrieved
context, thấp hơn nhiều so với giới hạn 8192 mà context window cho phép.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Thread sweep hoàn toàn phẳng (1.01× spread) từng khiến tôi nghĩ đo sai — hoá ra đó
chính là bằng chứng gián tiếp cho thấy GPU offload đã loại CPU threading khỏi
đường găng hoàn toàn, chứ không phải lỗi benchmark.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
