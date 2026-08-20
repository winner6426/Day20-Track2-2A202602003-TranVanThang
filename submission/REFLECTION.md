# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trần Văn Thắng
**Cohort:** 4
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** Intel(R) Core(TM) i5-10210U CPU @ 1.60GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** not recorded by the hardware probe
- **RAM:** 7.8 GB
- **Accelerator:** Vulkan device present
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` + `UD-Q2_K_XL` (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (local Windows)
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Máy có 7.8 GB RAM nên tôi chọn Qwen3.5 0.8B thay vì Gemma 4 E2B. Lần setup đầu dùng Python MSYS2 và gặp lỗi chứng chỉ SSL khi pip tải dependency. Tôi chuyển sang CPython 3.12, tạo lại `.venv`, rồi setup hoàn tất. Trên Windows tôi dùng `./lab.ps1 <target>` thay cho `make <target>`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 19246 | 4100 / 4608 | 135.6 / 190.4 | 12605 / 15703 / 15703 | 7.4 |
| UD-Q2_K_XL | 0.39 | 22897 | 4302 / 11434 | 1559.9 / 1842.7 | 102594 / 120926 / 120926 | 0.6 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

`UD-Q2_K_XL` saves only 0.11 GB (22%) but decodes about 11.5x slower by TPOT and raises median E2E latency from 12.6 s to 102.6 s. It is not worthwhile for interactive use on this machine. The result indicates compute/dequantization overhead dominates the small model; I would use `Q4_K_M` at 7.4 tok/s.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.02 | 57000 | 57000 | 57000 | 1.0 | 0.0% |
| 50 | 0.02 | 94000 | 94000 | 94000 | 2.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.20×
- **P95 tăng:** 1.65×
- **Effective concurrency ở 50 users:** 2.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.40 / 4 slots (85%)

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Kết quả là provisional vì chỉ 1 và 2 request hoàn tất, nhưng saturation signal vẫn rõ: tăng offered load 5× chỉ tăng throughput 1.20× (24% tuyến tính), còn P95 tăng 1.65× từ 57 s lên 94 s. `requests_deferred=6` và peak busy decode slots 3.40/4 cho thấy queueing. Với P95 SLO 60 s, 50 users không đạt; tôi sẽ giảm output-token budget trước, không tăng `--parallel`, vì slot đã gần đầy và decode chi phối latency.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost-only | stub |
| N17 Data pipeline | in-memory toy documents | stub |
| N18 Lakehouse | toy document dictionary | stub |
| N19 Vector + features | keyword-overlap retrieval; no embeddings/vector DB | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 19206.2 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM generation là bottleneck như dự kiến: 19,206.2 ms trên tổng 19,206.5 ms. Retrieval keyword chỉ mất 0.2 ms và embedding không được gọi. Để giảm latency 2×, tôi sẽ giảm decode/generation time bằng output-token budget nhỏ hơn hoặc cấu hình serving nhanh hơn; tối ưu retrieval sẽ gần như không ảnh hưởng tổng latency.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** giảm số decode threads từ `-t 4` xuống `-t 1` cho Q4_K_M.

```
before:  8.1 tok/s at -t 4
after:   8.7 tok/s at -t 1
speedup: 1.07x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Kết quả này ngược với kỳ vọng đơn giản rằng nhiều core hơn luôn nhanh hơn. Với model 0.8B và workload decode ngắn, lượng compute có thể song song hóa không đủ lớn để bù chi phí quản lý thread, đồng bộ, tranh chấp cache/memory và overhead của đường offload. Vì vậy `-t 1` đạt 8.7 tok/s, còn `-t 4` chỉ đạt 8.1 tok/s; tăng tới 8 hoặc 16 threads còn chậm hơn.

Thread count không phải bottleneck lớn nhất vì toàn bộ sweep chỉ trải rộng 1.09×. Tuy vậy đây là thay đổi tốt nhất đã đo được với cùng model và cùng metric: nó cải thiện throughput 7% mà không thay model hay giảm chất lượng output. Khi cần máy vẫn responsive, `-t 4` vẫn là lựa chọn hợp lý với mức giảm nhỏ.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
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
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
