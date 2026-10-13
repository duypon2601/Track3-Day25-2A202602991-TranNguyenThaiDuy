# Lab 25B — Report

**Họ tên / nhóm:** Trần Nguyễn Thái Duy — MSV 2A202602991  **Commit:** `bcc77a2`  **Điểm tự động (`make grade`):** 84 / 88 (4 điểm còn lại là dữ liệu SLO ẩn, giảng viên chấm)

> Số liệu lấy từ `reports/gameday/*.json`, `reports/cache/*.json`, `reports/workflows.json`, `reports/slo.json`
> (seed 25). Các thí nghiệm ở mục 3 và 8 chạy lại cùng băng, chỉ đổi đúng tham số được nêu.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0,7194 | 7,157 | 7,738 | 6,738 | 2,163 | 4 | FAIL |
| G1 | yours | 1,0 | 4,817 | 1,060 | 0,001 | 2,094 | 0 | PASS |
| G2 | naive | 0,4560 | 15,034 | 9,904 | 8,904 | 4,627 | 155 | FAIL |
| G2 | yours | 1,0 | 10,781 | 1,038 | 0,005 | 2,116 | 0 | PASS |
| G3 | naive | 1,0 | 4,768 | 1,164 | 0,164 | 2,328 | 0 | FAIL |
| G3 | yours | 1,0 | 4,659 | 1,060 | 0,023 | 2,142 | 0 | PASS |
| G4 | naive | 1,0 | 13,764 | 1,004 | 0,004 | 2,008 | 0 | FAIL |
| G4 | yours | 1,0 | 5,684 | 1,013 | 0,002 | 2,069 | 0 | PASS |
| G5 | naive | 0,3699 | 5,359 | 17,030 | 16,030 | 2,124 | – | FAIL |
| G5 | yours | 0,8179 | 5,482 | 1,003 | 0,002 | 2,504 | – | PASS |
| G6 | naive | 0,5170 | 4,836 | 12,344 | 11,344 | 2,009 | 0 | FAIL |
| G6 | yours | 1,0 | 6,225 | 1,067 | 0,027 | 2,164 | 0 | PASS |

Các check riêng của từng băng (naive → yours): `retry_after_violations` G1 36.027 → 0, G2 28.381 → 0;
`client_error_retries` G5 9.149 → 0; `unneeded_failovers` G5 = 0; `incompatible_routes` G6 = 0;
`primary_share_after_restore` G2 = 1,0, G4 = 1,0, G6 = 1,0.

Ngoài seed 25, tôi chạy thêm 8 seed (25–32) × 5 biến thể băng (lệch −30 s / +20 s / +45 s, cường độ ×0,8 / ×1,25)
= 40 lần mỗi băng: 0 lần FAIL. Biên hẹp nhất là G5 availability 0,7875–0,837 (ngưỡng 0,78) — con số này do tỷ lệ
request malformed quyết định, không phải do client (0 `give_up` ngoài `client_error`).

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8,0 | G6: `alpha-eu` treo, mỗi attempt mất đúng một timeout. Với 8 s chỉ có 78 attempt treo và availability 1,0; với 30 s availability tụt còn 0,9488 (104 lượt `deadline`). 8 s ≈ 2× p99 lúc bình thường của provider chậm nhất (`beta` ~4,1 s) và còn đủ chỗ cho 3 attempt trong deadline 30 s. |
| max_attempts_per_provider | 2 | Tối đa 1 retry cùng provider rồi fail over. G3: retry cùng `alpha` chỉ thành công 70%, fail over sang `alpha-eu` gần 100%. Đặt 5 (thí nghiệm 1) làm amplification G2 từ 1,038 lên 1,355. |
| backoff_base_s / cap_s | 0,25 / 4,0 | Full jitter để retry không dồn về cùng một thời điểm; cap 4 s vì retry dài hơn nữa ăn hết deadline 30 s. G3 p95 4,659 s — backoff không làm chậm lượt thành công. |
| retry_budget_ratio | 0,05 (+ 0,25 token/s, tối đa 20) | G3 chấm `retry_ratio ≤ 0,10`. Giá trị starter 0,1 + 0,5/s cho trần lý thuyết 0,1 + 0,5/8 = 0,16 ở 8 call/s; trên 40 lần chạy đo được tối đa 0,068. Với 0,05 + 0,25/s trần là ~0,08, đo được tối đa 0,037 (seed 25: 0,035 → 0,023), availability vẫn 1,0. |
| breaker.window_size / min_calls | 20 / 10 | Ở 8 call/s, 10 kết quả đến trong ~1,25 s: đủ nhanh để G6 chỉ tốn tổng cộng 51 lỗi 503 trong 300 s `alpha` sập (kể cả probe), đủ lớn để nhiễu nền 0,2% không làm trip (G5: 413 lỗi 400 nhưng `unneeded_failovers` = 0, tức breaker `alpha` không mở). |
| breaker.failure_rate_threshold | 0,5 | G3 (30% lỗi) cố ý **không** trip: `alpha` vẫn nhận 2.983/4.089 attempt, phần lỗi được retry/fail over. G6 (100% lỗi) trip ngay. |
| breaker.slow_call_s | 5,0 (tỷ lệ 0,6) | G4 không có lỗi nào, chỉ có độ trễ +5 s → latency ~6,2 s. p99 lúc khỏe của `alpha` ~2,7 s nên 5 s tách rõ hai trạng thái. Kết quả: p95 13,764 s (naive) → 5,684 s. |
| breaker.open_duration_s | 10,0 | Quyết định tốc độ quay lại primary: G2/G4/G6 `recovery_s` = 0, `primary_share_after_restore` = 1,0. Ngắn hơn thì tốn thêm probe vào provider đang hỏng (G6: mỗi probe treo 8 s). |
| breaker.half_open_max_probes | 2 (cần 2 probe tốt) | Giới hạn "học phí" khi thăm dò: suốt 300 s sự cố G6, `alpha-eu` chỉ nhận 78 attempt (so với ~2.400 nếu không chặn). |

Sơ đồ quyết định của `on_error`:

```
lỗi -> classify()
 |
 |-- CLIENT_ERROR (400/401/403/404/422)
 |     breaker.release (không nói gì về sức khỏe provider)
 |     -> GiveUp("client_error")            không retry, không fail over, không cooldown
 |
 |-- QUOTA_EXHAUSTED (429 insufficient_quota)
 |     breaker.release
 |     cooldown[provider] = now + quota_cooldown_s (300 s); đánh dấu "không bao giờ chờ"
 |     -> fail over ngay (delay 0)
 |
 |-- RATE_LIMITED (429 + retry-after)
 |     breaker.release
 |     cooldown[provider] = now + retry_after   (dùng chung cho mọi call)
 |     -> fail over ngay (delay 0)
 |
 `-- OVERLOADED / SERVER_ERROR / TIMEOUT
       breaker.on_failure
       tries < max_attempts_per_provider  VÀ breaker còn CLOSED
         VÀ backoff vừa deadline  VÀ retry budget còn token ?
           có    -> retry CÙNG provider sau full-jitter backoff
           không -> fail over ngay

fail over = đi tiếp xuống ladder (đã lọc theo tool calling), bỏ qua provider đang cooldown
            hoặc bị breaker từ chối.
hết ladder -> nếu có cooldown rate-limit <= 5 s, breaker không OPEN và còn kịp deadline:
              chờ đúng retry-after + jitter rồi gửi; ngược lại GiveUp("no_provider_available").
```

## 3. Ba thí nghiệm

**1. Retry budget là lớp phòng thủ cuối.** Breaker bị vô hiệu (`min_calls: 1e9`), `max_attempts_per_provider: 5`.

| Băng | Budget | availability | amplification | retry_ratio | recovery (s) | attempts |
|---|---|---:|---:|---:|---:|---:|
| G2 | giữ | 1,0 | 1,355 | 0,032 | 0 | 6.402 |
| G2 | tắt | 0,967 | 1,597 | 0,400 | 32 | 7.528 |
| G3 | giữ | 1,0 | 1,126 | 0,037 | 0 | 4.346 |
| G3 | tắt | 1,0 | 1,162 | 0,158 | 0 | 4.483 |

Khi breaker hỏng, budget là thứ duy nhất chặn retry: ở G2 nó giữ retry_ratio ở 0,032 thay vì 0,400, và hệ thống
không rơi vào trạng thái nghẽn kéo dài (recovery 0 s so với 32 s — vượt ngưỡng 30 s, availability mất 3,3 điểm).
Ở G3, tắt budget đưa retry_ratio lên 0,158 > 0,10 (FAIL). Khi breaker hoạt động đúng thì nó làm phần lớn công việc
(G2 amplification 1,038, tốt hơn cả 1,355 của budget đơn độc), vì breaker cắt hẳn traffic sang provider hỏng, còn
budget chỉ giới hạn *tỷ lệ*. Vẫn cần cả hai vì chúng hỏng theo cách khác nhau: breaker là trạng thái theo từng
provider và phụ thuộc cấu hình đúng (ngưỡng, `min_calls`); budget là giới hạn toàn cục, không phụ thuộc việc nhận
diện đúng provider nào đang lỗi — lỗi 30% rải rác như G3 không bao giờ làm breaker trip, chỉ budget giữ được nó.

**2. `attempt_timeout_s: 30`.**

| Băng | timeout | availability | p95 (s) | give_up_reasons |
|---|---:|---:|---:|---|
| G6 | 8 | 1,0 | 6,225 | – |
| G6 | 30 | 0,9488 | 5,480 | `deadline`: 104, `no_provider_available`: 18 |
| G4 | 8 | 1,0 | 5,684 | – |
| G4 | 30 | 1,0 | 5,608 | – |

Ở G6, availability tụt 5,1 điểm (FAIL ngưỡng 0,95) trong khi p95 lại *giảm* từ 6,225 xuống 5,480 s: mỗi call vào
`alpha-eu` treo đủ 30 s rồi hết deadline, và những lượt đó biến mất khỏi p95 vì p95 chỉ tính lượt thành công
(survivorship bias). G4 gần như không đổi vì `alpha` vẫn *trả lời* sau ~6,2 s — timeout không bao giờ được chạm
tới, thứ cứu G4 là slow-call breaker chứ không phải timeout. Timeout nên đặt từ hai phía: sàn là p99 lúc bình thường
của provider (ở đây ≤ ~4,1 s) để không cắt nhầm request khỏe, trần là `(deadline của user − thời gian cho một lần
fail over) / số attempt cần giữ chỗ`; không bao giờ bằng chính deadline của user.

**3. Bỏ lọc tool calling khỏi `ladder()`.** G6: `incompatible_routes` 0 → 235, availability 1,0 → 0,9015 (FAIL),
235 lượt kết thúc với `incompatible_model_output`. Nói với product manager: "trong 5 phút provider chính sập,
235 khách yêu cầu hoàn tiền được chuyển sang một model rẻ không biết gọi tool. Model vẫn trả lời, hệ thống ghi nhận
là *thành công* (HTTP 200, không có lỗi nào để alert), nhưng yêu cầu hoàn tiền không hề được thực hiện. Đây là kiểu
hỏng tệ nhất: dashboard xanh, khách không được phục vụ, và ta còn trả tiền cho 235 call vô ích." Fallback chỉ có
giá trị khi model thay thế làm được đúng việc đó.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1,0 | 81 | 128 | 81 | 0 | 1.496.700 cent | 1,123 |
| yours | 1,0 | 66 | 0 | 0 | 0 | 0 | 0,303 |

**Chỉ idempotency key là chưa đủ:** key ngăn được trả tiền hai lần, nhưng worker mới không có trí nhớ nên phải chạy
lại toàn bộ workflow từ `classify_request` — tốn LLM call (naive lãng phí 1,123 call/crash, có checkpoint còn 0,303)
và kết quả các bước không có key có thể khác lần trước. **Chỉ checkpoint là chưa đủ:** giữa "payment đã chạy" và
"checkpoint đã ghi" luôn có một khe hở. Payment chạy xong rồi worker chết (`WorkerCrash`) hoặc mất response
(`ServiceTimeout`) thì chưa có checkpoint, lần chạy lại sẽ hoàn tiền lần hai nếu không có key — đó chính là 128
hoàn tiền trùng (14.967 USD trả thừa) của bản naive. Key đóng khe hở đó; checkpoint làm việc resume rẻ.

**Ghi checkpoint trước khi gọi payment:** nếu worker chết sau khi ghi nhưng trước khi tiền đi, worker sau thấy
checkpoint, coi bước đó đã xong và bỏ qua → **hoàn tiền bị mất**, khách nhận email "đã hoàn tiền" mà không có tiền.
Lỗi này còn âm thầm hơn hoàn tiền trùng vì không có bản ghi nào trong ledger để đối soát.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (1 phút) | page | 27 phút (hạn 35) |
| provider_outage | page | page (0 phút) | page | 4 phút (hạn 8) |
| deploy_blip | none | **page sai** | không | – |
| slow_leak | ticket | **bỏ sót** | ticket | 1.337 phút (hạn 1.800) |
| degraded_evening | page | page (214 phút) | page | 291 phút (hạn 330) |
| night_spike | none | **page sai** | không | – |
| small_bump | none | **page sai** | không | – |

Naive: 7 page, 3 page sai, bỏ sót `slow_leak` → FAIL. Burn-rate: 3 page, 1 ticket, 0 page sai → PASS. Kiểm tra thêm
trên 5 seed × 3 lịch sự cố lệch giờ (dữ liệu sinh bằng `slo/data.py`): 15/15 PASS.

**Error budget 30 ngày:** 25.901.755 request, cho phép 129.508,8 request lỗi, thực tế 55.618 → availability 99,7853%.
Đã tiêu **42,95%**, còn **57,05%** → **không** đóng băng release (chỉ đóng băng khi budget còn lại ≤ 0).

**Vì sao `small_bump` không nên đánh thức ai:** 20 phút ở 12% lỗi là ~1.800 request lỗi ≈ 1,4% budget tháng. Trên
cửa sổ 1 giờ, tỷ lệ lỗi chỉ là 20 × 12% / 60 = 4% → burn rate 8×, dưới ngưỡng page 14,4× (ngưỡng tương ứng với việc
đốt 2% budget trong 1 giờ); trên cửa sổ 6 giờ là 0,67% → 1,3×, dưới 6×. Sự cố tự hết, không đe dọa SLO, và không có
hành động nào on-call làm được mà thay đổi kết quả. Page cho những việc như vậy làm on-call mất niềm tin vào page —
đến lúc có sự cố thật thì phản ứng chậm đi. Nó vẫn được tính vào budget và hiện ra ở báo cáo tuần.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình: `reports/gameday/G2_overload_storm.png`. Số: `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt:** Lúc t=120 s `alpha` mất 75% năng lực GPU trong 3 phút. Client v1 phản ứng bằng 3 tầng retry chồng nhau
(tối đa 36 attempt/call, delay cố định 0,2 s, không fail over), đẩy lưu lượng vào `alpha` từ ~8 lên trung bình
149 attempt/s (đỉnh 254), khiến tỷ lệ thành công về 0. Provider hồi phục lúc t=300 nhưng dịch vụ chỉ trở lại
lúc t≈455–470: sự cố 3 phút của provider thành ~5,4 phút sập hoàn toàn phía người dùng.

**Ảnh hưởng:** 1.285 / 2.362 lượt user thất bại (availability 45,6%); không có call nào thành công trong khoảng
t=142–450 (~308 s), trong đó 150 s nằm *sau* khi provider đã hồi phục (`recovery_s` = 155). 34.475 attempt cho
3.481 call (amplification 9,9), chi phí 4,627 / lượt tốt (bình thường ~2,0). Với 4 lượt/s, một tháng có ~10,37 triệu
lượt, budget 0,5% = 51.840 lượt lỗi → riêng sự cố này tiêu **~2,5% error budget tháng** (1.285 / 51.840).

**Timeline** (giây ảo):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 60–120 | Bình thường | 6–8 attempt/s vào `alpha`, success 1,00, amplification ~1,0 |
| 120 | `alpha` mất 75% năng lực (trigger) | năng lực ~15 → ~3,75 req/s, tải vào vẫn ~8 call/s |
| 120–135 | Hàng đợi dài dần, chưa có lỗi | attempt 6–13/s; call hoàn tất giảm còn 3,1/s (trước đó 8,1/s); success vẫn 1,00 |
| 135–140 | Timeout 8 s và 529 đầu tiên → retry bắt đầu | 90,8 attempt/s; 5 call hoàn tất, cả 5 thành công |
| 140–145 | Retry storm | **237 attempt/s** (đỉnh 254 trong 1 s), 0/5 call thành công; lỗi đầu tiên với user lúc t=142 |
| 150–300 | Sập hoàn toàn | trung bình 149 attempt/s (24.575 attempt), success 0,00 suốt 150 s |
| 300 | `alpha` hồi phục hoàn toàn | năng lực về ~15 req/s |
| 300–360 | **Vẫn sập** dù provider đã khỏe | attempt giảm còn 15–16/s (≥ năng lực 15/s), success 0,00 |
| 360–420 | Retry storm lần hai | attempt tăng lại 77–136/s, success 0,00 |
| 420–450 | Tải lắng xuống | 12–14 attempt/s, success vẫn 0,00 |
| 450–470 | Hồi phục | success 8/25 (t=450–455) → 37/45 → 62/63 (t=465–470); lỗi cuối cùng lúc t=468 |

Trong toàn băng: 26.298 lỗi `rate_limit_exceeded`, 3.194 lỗi `overloaded_error` (529), 2.737 `client_timeout`;
28.381 lần retry trước hạn `retry-after`.

**Trigger vs root cause.** *Trigger* là `alpha` mất 75% GPU — một sự cố của provider, nằm ngoài tầm kiểm soát và
chắc chắn sẽ còn xảy ra. *Root cause* là thiết kế retry của client: ba tầng (app × gateway × SDK = 36 attempt)
nhân với nhau, delay cố định không jitter, không tôn trọng `retry-after`, không có retry budget, không có breaker,
không có đường fail over. Với thiết kế đó, bất kỳ sự suy giảm năng lực nào cũng bị khuếch đại ~10–30 lần đúng vào
lúc provider yếu nhất. Bằng chứng: cùng băng, cùng trigger, `ResilientClient` đạt availability 1,0 với đỉnh chỉ
18 attempt/s vào `alpha`.

**Vì sao không tự hồi phục sau t=300 (metastable failure).** Vòng lặp tự duy trì:

1. Hàng đợi của `alpha` đầy request mà client **đã bỏ** (timeout 8 s) — server không biết và vẫn xử lý chúng.
   Toàn bộ năng lực vừa hồi phục bị tiêu vào công việc không ai còn chờ.
2. Request mới phải chờ sau đống đó, lâu hơn 8 s → client lại timeout → lại retry → thêm một request bị bỏ vào hàng đợi.
3. Tải retry (15–16 attempt/s ở t=300–360, rồi 77–136/s ở t=360–420) luôn ≥ năng lực 15 req/s, nên hàng đợi không
   bao giờ rút xuống dưới ngưỡng 8 s. Phần vượt quá bị trả 429/529 trong 0,05–0,1 s và được retry sau 0,2 s —
   càng bị từ chối nhanh thì retry càng dồn dập.

Trigger đã hết, nhưng tải do chính client tạo ra đủ để giữ hệ thống ở trạng thái xấu. Hệ thống chỉ thoát ra khi
các call cũ lần lượt chạm trần 36 attempt hoặc hết deadline 30 s (655 `retries_exhausted`, 585 `deadline`) và tải
rơi xuống dưới năng lực đủ lâu để hàng đợi rút hết (t≈450).

**Điều gì giúp / điều gì làm tệ hơn.**
- Giúp: deadline 30 s của user và trần 36 attempt — hai thứ duy nhất khiến storm tự tắt. Rate limit 20 rps của
  tenant chặn được phần lớn attempt trước khi chúng vào hàng đợi.
- Làm tệ hơn: retry nhân theo tầng; delay cố định 0,2 s (mọi client quay lại cùng lúc); bỏ qua `retry-after`
  (28.381 vi phạm — chính 429 lại là nguồn retry nhanh nhất); timeout 8 s ngắn hơn thời gian chờ hàng đợi tối đa 10 s
  nên mọi request được nhận đều thành công việc bỏ đi; không dùng `alpha-eu`/`beta` dù cả hai đang khỏe.
- Phát hiện: không có alert theo amplification; nhìn tỷ lệ lỗi của provider thì sau t=300 "provider đã ổn".

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Gom retry về **một** tầng (tắt retry của SDK và gateway), full jitter, tôn trọng `retry-after` | prevent | Team Agent Platform | CI chạy băng G2: `amplification ≤ 1,2`, `retry_after_violations = 0` (hiện tại: 1,038 và 0) |
| 2 | Retry budget toàn cục 5% + circuit breaker sliding-window có slow-call cho từng provider | prevent | Team Agent Platform | CI: G2 `recovery_s ≤ 30`, G3 `retry_ratio ≤ 0,10`; test tắt breaker (thí nghiệm 1) vẫn giữ `retry_ratio ≤ 0,05` |
| 3 | Fallback ladder `alpha → alpha-eu → beta` có lọc tool calling | mitigate | Team Agent Platform | CI: G2 availability ≥ 0,95 (hiện 1,0), G6 `incompatible_routes = 0` |
| 4 | Alert theo `amplification` (attempt / call > 2 trong 1 phút) và theo burn rate đa cửa sổ của availability lượt user | detect | SRE on-call | Phát lại băng G2 naive trên staging: alert phải nổ trước t=150 (storm bắt đầu t=135) |
| 5 | Game day hàng quý: phát lại 6 băng + băng ẩn trên bản release ứng viên | detect | SRE + Agent Platform | Biên bản game day; mọi băng PASS là điều kiện release |
| 6 | Gửi deadline còn lại kèm request để provider/gateway bỏ request đã hết hạn thay vì xử lý | mitigate | Team Gateway | Đo trên staging: số request đã hết hạn nhưng vẫn được xử lý = 0 |

## 7. Một điểm yếu còn lại

**G2 p95 = 10,781 s** (trên 40 lần chạy: 4,98–11,91 s) — qua được vì G2 không chấm p95, nhưng đó là chậm thật với
user. Breaker chỉ trip sau khi có đủ 10 kết quả mà ≥ 60% chậm hơn 5 s; khi `alpha` vừa mất năng lực, các call đã
vào hàng đợi phải chờ tới 8 s rồi mới timeout và fail over (118 `client_timeout`), tức là mỗi lượt đó mất 8 s trước
khi được phục vụ. Sau đó mỗi 10 s lại có 2 probe HALF-OPEN chịu chung số phận.

Cách sửa: (1) **hedged request** — sau khoảng p95 lúc khỏe (~2,1 s với `alpha`) mà chưa có trả lời thì gửi song song sang
`alpha-eu`, lấy kết quả về trước, giới hạn hedge ở ~5% call để không tự gây quá tải; (2) hạ `slow_call_s` theo p99
đo được của từng provider thay vì một hằng số 5 s cho cả bốn; (3) dùng probe tổng hợp (request rẻ, không gắn với
user) cho HALF-OPEN để không user nào phải trả "học phí" 8 s.

Điểm yếu thứ hai: chi phí khi `beta` gánh tải — G5 cost/turn 2,504 so với ~2,07 bình thường, vì `beta` đắt hơn 40%
và client hiện không có ngân sách chi phí nào.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0,8373 | 0,8424 | 0 | 0 | 0 | 0 | 1,386 | FAIL |
| cached (của bạn) | 0,9124 | 0,9645 | 0,592 | 192 | 0 | 2 | 0,955 | PASS |

Khối `cache:` của starter (0,80 / 600 / 0) cho: `wrong_answers` 181 (`other_question`), `outdated_answers` 80,
`faq_availability` 0,8587 → FAIL ba check.

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0,90 | Đo trên embedding của catalog: các cách hỏi của *cùng* một ý có cosine ≥ 0,951; hai ý *khác nhau* gần nhất là 0,862. 0,90 nằm giữa khoảng trống đó. 0,80 → 181 câu trả lời sai; 0,97 → hit ratio 0,499 (FAIL). |
| ttl_s | 30 | Thứ duy nhất giới hạn `outdated_answers`. TTL 300 → 95 câu lỗi thời (ngưỡng 15). TTL 30: 2 trên seed 25, tối đa 9 trên 24 lần chạy (8 seed × 3 băng lệch giờ), hit ratio 0,563–0,594. TTL 45 đã có lần 16–18 (FAIL); TTL 20 hit ratio tối thiểu chỉ còn 0,512 (ngưỡng 0,5). |
| max_stale_s | 300 | Phải dài hơn sự cố (120 s trên C1) cộng khoảng cách giữa hai lần hỏi của ý ít gặp. 0 → `faq_availability` 0,8529; 60 → 0,918 (vẫn FAIL 0,93); 300 → 0,9645. |

**Thí nghiệm:**

1. **Ngưỡng.**

   | threshold | hit ratio | wrong_answers | faq_availability | cost/turn |
   |---:|---:|---:|---:|---:|
   | 0,80 | 0,610 | 181 (`other_question`) | 0,8593 | 1,016 |
   | 0,90 | 0,592 | 0 | 0,9645 | 0,955 |
   | 0,97 | 0,499 | 0 | 0,9640 | 1,018 |

   Ở 0,80 các cặp "gần mà khác" (cosine ~0,84–0,86) bị trả nhầm cho nhau: chính sách hoàn tiền ↔ thời gian hoàn
   tiền, phí ship nội thành ↔ đi tỉnh, đổi size ↔ đổi màu, hình thức thanh toán ↔ COD. Hit ratio chỉ nhỉnh hơn
   0,018 nhưng đổi lấy 181 câu sai — mỗi câu sai làm hỏng một lượt, nên availability còn *thấp* hơn và cost/turn
   cao hơn. Ở 0,97 cache không còn nhận ra các cách hỏi khác nhau của cùng một ý (cosine 0,951–0,984): hit ratio
   mất 9,3 điểm, không an toàn hơn chút nào.

2. **TTL.**

   | ttl_s | hit ratio | outdated_answers | stale_served | cost/turn |
   |---:|---:|---:|---:|---:|
   | 30 | 0,592 | 2 | 192 | 0,955 |
   | 60 | 0,652 | 3 | 157 | 0,915 |
   | 300 | 0,731 | 95 | 68 | 0,863 |

   Webhook chỉ giúp khi *có ai đó gửi nó*. Bảng phí ship đổi lúc t=480 mà không có sự kiện nào tới client, nên
   cache không có cách nào biết entry đã sai; chỉ còn tuổi của entry là tín hiệu, và TTL là cận trên của thời gian
   một câu lỗi thời còn được phục vụ. TTL 60 qua được seed 25 (3 câu) nhưng là do pha của entry so với thời điểm
   thay đổi — kỳ vọng số câu lỗi thời tỷ lệ với TTL, nên tôi chọn 30 để có biên an toàn trên băng ẩn, chấp nhận
   mất 6 điểm hit ratio và 0,04 cost/turn.

3. **Stale-if-error.** `max_stale_s: 0` → `faq_availability` 0,9645 → 0,8529 (−11,2 điểm, FAIL), `stale_served`
   192 → 0, hit ratio 0,592 → 0,481. Trong 120 s mọi provider sập, không còn gì để trả lời câu hỏi FAQ chung; với
   cửa sổ stale, 192 lượt đó vẫn nhận được câu trả lời đúng (0 `wrong_answers`, vì invalidation và scope
   `prompt_version` vẫn được áp dụng cho cả entry stale).

**Trả lời:**

- **Vì sao không cache câu hỏi cá nhân, kể cả cache theo từng user?** Có hai lý do độc lập. (1) *An toàn:* câu hỏi
  của hai user chỉ khác số đơn, embedding giống nhau ~0,99 — cao hơn mọi ngưỡng hợp lý — nên chỉ cần một lỗi nhỏ ở
  khâu scope là đơn hàng của người này lộ cho người khác (`other_user`). Rủi ro là lộ dữ liệu cá nhân, lợi ích gần
  như bằng không. (2) *Đúng đắn:* dù tách theo user, câu trả lời phụ thuộc trạng thái đổi theo phút (đơn đang ở đâu,
  tiền đã về chưa) và không có webhook nào báo cho cache; trả lời "đơn đang giao" khi đơn đã giao xong là sai
  (`personal_cached`). Thêm nữa, mỗi user hỏi câu của riêng mình một hai lần nên hit ratio theo user gần 0 —
  không có gì để đổi lấy rủi ro đó. Vì vậy cache dùng allowlist: chỉ câu hỏi chung, dựa trên tài liệu chính sách.

- **Dashboard xanh nhờ cache — alert bằng số nào?** Alert trên những số đo *provider*, không phải số đo lượt user:
  (1) `stale_served` / giây > 0 kéo dài — trên C1 nó bằng 0 lúc bình thường và ~1,6/s trong sự cố (192 lượt/120 s),
  là tín hiệu trực tiếp "ta đang sống nhờ cache"; (2) tỷ lệ thành công *ở origin* — call thực sự gửi tới provider
  — và số breaker đang OPEN (cả 4 cùng mở = page); (3) availability của lượt không cache được (câu hỏi cá nhân, lượt
  cần tool): trên C1 availability tổng chỉ 0,9124 dù FAQ đạt 0,9645, tức là nhóm này đang thất bại; (4) tuổi lớn
  nhất của câu trả lời đang được phục vụ so với `ttl_s + max_stale_s` — khi nó tiến gần 330 s thì cache sắp hết
  khả năng che, và sự cố sẽ lộ ra toàn bộ cùng một lúc.
