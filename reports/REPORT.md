# Lab 25B — Report

**Họ tên / nhóm:** Hoàng Anh Minh - 2A202602566  **Commit:** `c0e729dc7f807ef372e8c4daaefb68203fc56f04`  **Điểm tự động (`make grade`):** 84.0 / 88

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.217 | 1.044 | 0.005 | 2.129 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.742 | 1.064 | 0.034 | 2.150 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.958 | 1.053 | 0.019 | 2.203 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Trong G6, `alpha-eu` bị treo hoàn toàn (hang). Nếu để 30s (bằng deadline turn), request sẽ bị kẹt 30s và hết sạch deadline (`give_up: deadline = 107`), availability sụt xuống 0.9488. Đặt 8.0s giúp phát hiện treo sớm, kịp failover sang `beta` trong khi vẫn đủ lớn so với median thông thường (~1.2s - 1.8s). |
| max_attempts_per_provider | 2 | Cho phép tối đa 1 lần retry (tổng 2 attempts) trên cùng provider. Trong G2 và G5, `naive` retry vô tội vạ dẫn đến amplification vọt lên 9.904 và 17.03. Giới hạn 2 attempts giúp dập tắt khuếch đại ngay tại nguồn. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Áp dụng Full Jitter Exponential Backoff với base 0.25s và cap 4.0s. Tránh hiện tượng thundering herd đồng loạt dội ngược vào provider khi vừa hồi phục, đồng thời tôn trọng `retry-after` header trong G1 (giảm vi phạm retry-after từ 36,027 xuống 0). |
| retry_budget_ratio | 0.1 | Đảm bảo lượng retry không bao giờ vượt quá 10% tổng request mới. Trong G3 (brownout 30% lỗi), `naive` có retry_ratio = 0.164 (> 0.10 max), còn client của ta giới hạn chuẩn ở 0.034. |
| breaker.window_size / min_calls | 20 / 10 | Đánh giá tỷ lệ lỗi trên cửa sổ trượt 20 kết quả gần nhất, chỉ kích hoạt khi đã có ít nhất 10 kết quả. Tránh tình trạng ngắt mạch oan do vài lỗi ngẫu nhiên ban đầu khi lưu lượng còn thấp. |
| breaker.failure_rate_threshold | 0.5 | Ngắt mạch khi tỷ lệ lỗi $\ge 50\%$. Giúp client nhanh chóng phát hiện `alpha` bị quá tải trong G2 hoặc sập hoàn toàn trong G6 để failover sang `alpha-eu`/`beta`. |
| breaker.slow_call_s | 5.0 | Bắt các cuộc gọi thành công nhưng chậm $\ge 5.0$s. Trong G4 (`alpha` bị chậm thêm 5s, latency lên ~6.2s nhưng không có lỗi HTTP), nếu không có tham số này, p95 sẽ vọt lên 13.764s như `naive`. Nhờ có `slow_call_s = 5.0`, breaker ngắt mạch và kéo p95 xuống còn 5.684s. |
| breaker.open_duration_s | 10.0 | Thời gian giữ trạng thái OPEN là 10s trước khi sang HALF_OPEN để thăm dò. Đủ để provider xả nghẽn hàng đợi (queue wait tối đa của provider là 10s). |
| breaker.half_open_max_probes | 2 | Chỉ cho phép tối đa 2 request probe đồng thời trong HALF_OPEN. Ngăn chặn làn sóng request ồ ạt tràn vào làm sập lại provider đang trong giai đoạn hồi phục. |

Sơ đồ quyết định của `on_error` (loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```text
                                  +-------------------+
                                  |   ProviderError   |
                                  +---------+---------+
                                            |
                                      classify(err)
                                            |
         +--------------------+-------------+---------------+----------------------+
         |                    |                             |                      |
   CLIENT_ERROR        QUOTA_EXHAUSTED                 RATE_LIMITED          OVERLOADED /
  (400, 401, 422)   (429 insufficient_quota)       (429 rate_limit_exceeded) SERVER_ERROR / TIMEOUT
         |                    |                             |                      |
  release(permit)      release(permit)               release(permit)        on_failure(permit)
         |                    |                             |                      |
  GiveUp("client_error") cooldown 300s               cooldown retry_after    Breaker OPEN?
  (Không retry,       Failover ngay                  Failover ngay           Hoặc tries >= max_attempts?
   Không failover)    (idx+1, delay=0)               (idx+1, delay=0)        Hoặc hết budget/deadline?
                                                                                   |
                                                                        +----------+----------+
                                                                        | NO                  | YES
                                                                        v                     v
                                                                   Retry CÙNG             Failover
                                                                   provider sau           sang provider
                                                                   backoff.delay(tries)   tiếp theo (idx+1)
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.**  
   Khi giả lập breaker cấu hình sai (`breaker.min_calls: 1000000000`) và đặt `max_attempts_per_provider: 5`:
   - Khi **bật** retry budget (`ratio=0.1`): G2 đạt availability 0.9987, amplification 1.380, retry_ratio 0.058, recovery 0.0s; G3 đạt availability 1.0, amplification 1.136, retry_ratio 0.068.
   - Khi **tắt** retry budget (`ratio=100.0`): G2 bị khuếch đại tải nặng với amplification vọt lên 1.673 (vượt trần 1.6), retry_ratio lên tới 0.494 (gần 50% traffic là retry), recovery mất 8.0s; G3 có retry_ratio vọt lên 0.159 (vi phạm trần 0.10).
   - Ngay cả khi breaker hoạt động đúng, retry budget vẫn tối quan trọng vì đóng vai trò là "chốt chặn phân tán": Breaker chỉ ngắt khi tỷ lệ lỗi đã vượt ngưỡng, trong khi Retry Budget chủ động khống chế tổng thông lượng retry của toàn cụm dưới mức 10% tại mọi thời điểm, ngăn chặn nguy cơ retry storm âm ỉ tích tụ.

2. **`attempt_timeout_s: 30`.**  
   Khi nâng `attempt_timeout_s` lên 30.0s:
   - Ở G4 (slow region), availability vẫn đạt 1.0 và p95 đạt 5.608s do breaker phát hiện slow call và chuyển mạch.
   - Tuy nhiên ở G6 (provider outage, `alpha-eu` bị hang), availability sụt nghiêm trọng xuống **0.9488** (thất bại so với ngưỡng 0.95), ghi nhận **107 lượt give_up do deadline** (`give_up_reasons: {'deadline': 107, 'no_provider_available': 15}`).
   - Nguyên nhân: Khi một provider bị treo, request tốn trọn 30s của turn deadline để chờ đợi; khi timeout xảy ra thì turn đã cạn kiệt thời gian, không còn cơ hội failover sang `beta`. Do đó, timeout "đúng" phải được tính toán dựa trên độ trễ p99 bình thường của provider (~1.5s) cộng thêm biên độ an toàn (~8s) để chừa đủ thời gian cho các bước fallback tiếp theo trước khi hết deadline người dùng (30s).

3. **Bỏ lọc tool calling khỏi `ladder()`.**  
   Khi cho phép tool call rơi vào cả model không hỗ trợ tool (`gamma-mini`):
   - Ở băng G6, hệ thống ghi nhận **235 lượt định tuyến không tương thích** (`incompatible_routes = 235`), availability tổng tụt dốc từ 1.0000 xuống **0.9015**.
   - Dưới góc nhìn Product Manager: Có đúng 235 khách hàng đang yêu cầu xử lý hoàn tiền nhưng hệ thống lại chuyển tiếp yêu cầu đến một mô hình AI giá rẻ không có khả năng thực thi công cụ thanh toán. Kết quả là giao dịch hoàn tiền bị tê liệt, khách hàng không nhận được tiền bồi hoàn và rời bỏ dịch vụ trong bức xúc, dù chatbot vẫn phản hồi tin nhắn văn bản thông thường.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Trực quan lý thuyết:
- **Vì sao chỉ idempotency key là chưa đủ?** Nếu chỉ có idempotency key mà không có checkpoint bền vững, khi worker bị crash ở bước sau (ví dụ bước gửi email hoặc tóm tắt), worker mới khởi động lại sẽ phải chạy lại từ đầu tất cả các bước LLM phía trước, gây lãng phí chi phí LLM (`wasted_llm_per_crash` cao) và gia tăng latency.
- **Vì sao chỉ checkpoint cũng chưa đủ?** Nếu chỉ lưu checkpoint mà không truyền idempotency key xuống payment gateway, trường hợp bước thanh toán đã trừ tiền thành công nhưng kết nối mạng bị timeout trước khi nhận response, lệnh retry tiếp theo sẽ gửi yêu cầu thanh toán mới và gây ra trừ tiền trùng lặp (duplicate refund).
- **Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?** Nếu ghi checkpoint hoàn tiền thành công trước khi thực sự gọi API payment, khi worker bị crash đúng thời điểm đang gọi payment (tiền chưa đi), worker mới phục hồi sẽ đọc checkpoint, tưởng rằng tiền đã hoàn và chuyển sang bước gửi email $\rightarrow$ dẫn đến sự cố nghiêm trọng: **hoàn tiền bị mất (`lost_refunds > 0`)**, khách nhận được email xác nhận hoàn tiền nhưng thực tế tài khoản không nhận được xu nào!

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (phút 3481) | page (phút 3507) | 27 min |
| provider_outage | page | page (phút 8760) | page (phút 8764) | 4 min |
| deploy_blip | none | page (phút 15240) - **FAIL** | none (không page) | Không có |
| slow_leak | ticket | none - **FAIL** | ticket (phút 22037) | 1337 min |
| degraded_evening | page | page (phút 30214) | page (phút 30291) | 291 min |
| night_spike | none | page (phút 34740) - **FAIL** | none (không page) | Không có |
| small_bump | none | page (phút 39542) - **FAIL** | none (không page) | Không có |

- **Error budget 30 ngày:** Đã tiêu **42.95%** (`budget_consumed = 0.4295`), còn lại **57.05%** (`budget_remaining = 0.5705`). Do ngân sách còn trên 50%, hệ thống **KHÔNG phải đóng băng release** (`freeze_releases = False`).
- **Vì sao `small_bump` (20 phút ở 12% lỗi) KHÔNG nên đánh thức ai lúc 11 giờ trưa?**  
  Với lưu lượng thông thường, một đợt tăng lỗi 12% kéo dài 20 phút chỉ tiêu tốn chưa đầy $0.12 \times 20 / (0.005 \times 43200) \approx 1.1\%$ tổng ngân sách lỗi của cả tháng. Nó không gây đe dọa nghiêm trọng tới SLO 99.5% của tháng. Nếu dùng ngưỡng đơn lẻ 5 phút như `naive alerter`, hệ thống sẽ kích hoạt chuông báo động giả (false page), làm gián đoạn công việc của kỹ sư on-call và gây ra hội chứng mệt mỏi vì cảnh báo (alert fatigue).

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình ảnh đính kèm: `reports/gameday/G2_overload_storm.png` và dữ liệu chuỗi thời gian từ `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**  
Vào lúc $t=120\text{s}$, provider chính `alpha` bị suy giảm 75% năng lực xử lý (mất GPU), kích hoạt hàng đợi nghẽn và mã lỗi quá tải 529. Phiên bản client naive thiếu cơ chế Circuit Breaker, không có jitter và không kiểm soát ngân sách retry, dẫn đến việc liên tục thử lại mù quáng và khuếch đại lưu lượng lên gấp 9.9 lần. Hậu quả là hệ thống rơi vào trạng thái lỗi siêu ổn định (metastable failure), khiến độ sẵn sàng sụt giảm nghiêm trọng còn 45.6% và mất tới 155 giây sau khi hạ tầng đã hồi phục mới có thể tự thoát khỏi sự cố.

**Ảnh hưởng:**  
- **1,285 lượt yêu cầu của khách hàng bị thất bại hoàn toàn** (chỉ có 1,077 lượt thành công trên tổng số 2,362 turns đến, availability đạt mức thảm họa 45.6%).
- Sự cố kéo dài liên tục **335 giây** (từ $t=120\text{s}$ đến $t=455\text{s}$).
- Vượt ngân sách lỗi: Với SLO 99.5%, số lỗi được phép cho 2,362 turns là $\approx 11.8$ turns. Lượng thất bại 1,285 turns đã đốt **10,880% (gấp gần 109 lần)** ngân sách lỗi được phép của toàn bộ chu kỳ.

**Timeline** (giây ảo — trích xuất từ `naive_G2_overload_storm_timeseries.csv`):

| t (s) | Sự kiện | Bằng chứng (số liệu thực tế) |
|---:|---|---|
| 120 | `alpha` mất 75% GPU, hàng đợi bắt đầu dồn ứ | $t=120$: calls_ok = 7, attempts = 5. Ngay sau đó tại $t=122$: calls_ok sụt về 0. |
| 136 | Bắt đầu bùng nổ bão retry (retry storm) | Attempts dồn vào alpha tăng vọt: $t=136$: 53 att/s $\rightarrow$ $t=138$: 122 att/s $\rightarrow$ $t=140$: 216 att/s. |
| 144 | Đỉnh điểm khuếch đại tải, hệ thống tê liệt hoàn toàn | $t=144$: 254 attempts/s đổ vào alpha (gấp 12.7 lần capacity bình thường là 20 rps). calls_ok = 0. |
| 300 | Nhà cung cấp `alpha` khôi phục hoàn toàn 100% capacity | Capacity của alpha hồi phục về mức bình thường, nhưng attempts vẫn duy trì ở mức 115 att/s; calls_ok = 0. |
| 370 | Làn sóng retry đợt 2 tự duy trì do client timeout dồn dập | Attempts tiếp tục bùng phát trở lại: $t=370$: 173 att/s, $t=372$: 176 att/s; calls_ok vẫn bằng 0. |
| 455 | Bão retry tan dần, hệ thống bắt đầu có response thành công đầu tiên | $t=453$: calls_ok = 4; $t=455$: calls_ok = 4; $t=459$: calls_ok = 10, attempts giảm về 16 att/s. Recovery mất **155 giây** kể từ $t=300$. |

**Trigger vs root cause:**  
- **Trigger:** Sự cố sụt giảm tạm thời 75% năng lực GPU của nhà cung cấp `alpha` trong khoảng thời gian từ $t=120\text{s}$ đến $t=300\text{s}$.
- **Root cause:** 
  1. Client thiếu Circuit Breaker để ngắt tải khi provider suy thoái.
  2. Thiếu Fallback Ladder: 100% attempts (34,475 lượt) dồn cứng vào `alpha` mà không điều hướng sang `alpha-eu` hay `beta`.
  3. Cơ chế retry nguyên thủy không có jitter và không có Retry Budget: Khiến mỗi request thất bại được nhân bản liên tục, gây ra hệ số khuếch đại tải lên đến 9.904 lần.
- **Vì sao provider đã hồi phục ở $t=300\text{s}$ mà hệ thống vẫn sập (Metastable Failure)?**  
  Server xử lý request theo cơ chế hàng đợi FIFO với thời gian chờ tối đa 10s. Khi client timeout ở giây thứ 8, server backend **vẫn tiếp tục xử lý request vô hồn đó trong hàng đợi**. Đồng thời, client lại gửi ngay request retry mới vào hàng đợi. Vòng lặp phản hồi dương này (Positive Feedback Loop) khiến hàng đợi của server luôn trong tình trạng đầy ắp các "zombie request" mà client đã bỏ cuộc. Tải công việc do retry tự sinh ra duy trì ở mức 115–176 att/s (vượt xa giới hạn 20 rps của server), tự duy trì trạng thái sập thêm 155s sau khi nguyên nhân gốc đã biến mất.

**Điều gì giúp / điều gì làm tệ hơn:**  
- *Làm tệ hơn:* Client timeout độc lập với server khiến server lãng phí năng lực tính toán xử lý các request đã quá hạn; việc retry tức thì không giãn cách thời gian làm nghẽn hoàn toàn băng thông.
- *Giúp giải quyết:* Triển khai Circuit Breaker phát hiện lỗi 529 và cắt mạch dứt khoát sang `alpha-eu`/`beta`; áp dụng Retry Budget chặn đứng tỷ lệ retry ở mức 10%; bổ sung Full Jitter Backoff phá vỡ tính đồng bộ của các đợt gọi lại.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai Sliding Window Circuit Breaker ngắt mạch sang `alpha-eu` và `beta` khi tỷ lệ lỗi $\ge 50\%$ hoặc độ trễ $\ge 5$s | Mitigate | Resilience Team | Chạy kiểm thử G2 trong CI: `recovery_s <= 5.0s`, `availability >= 0.95`. |
| 2 | Tích hợp Token-Bucket Retry Budget giới hạn retry tối đa 10% tổng traffic | Prevent | Core Platform | Kiểm tra kiểm thử tải G2/G3: `amplification <= 1.2` và `retry_ratio <= 0.10`. |
| 3 | Cấu hình Burn-Rate Alerting đa cửa sổ giám sát tốc độ đốt ngân sách lỗi của ShopAgent | Detect | SRE Team | Tích hợp Alertmanager: Cảnh báo page kích hoạt trong vòng $\le 5$ phút khi xảy ra sự cố sập tương tự. |

## 7. Một điểm yếu còn lại

**Tình huống điểm yếu:**  
Trong kịch bản suy thoái diện rộng hoặc đứt cáp quang quốc tế khiến **cả 3 provider lớn (`alpha`, `alpha-eu`, `beta`) đồng thời bị chậm hoặc quá tải**:
- `ResilientClient` hiện tại sẽ duyệt tuần tự qua ladder. Với `attempt_timeout_s = 8.0s`, nếu cả `alpha` và `alpha-eu` đều chạm trần timeout trước khi failover sang `beta`, tổng thời gian chờ của 1 turn đã mất $8 + 8 = 16$s.
- Nếu `beta` cũng phản hồi chậm (~5s), tổng thời gian của 2 bước trong turn (tool + chat) sẽ chạm trần 30s (`turn_deadline_s`), khiến tỷ lệ trễ p95/p99 tăng vọt và nguy cơ chạm deadline người dùng tăng cao.
- Ngoài ra, việc dồn toàn bộ lưu lượng sang `beta` (chi phí 1.4/call) làm chi phí vận hành tăng vọt 40%.

**Giải pháp khắc phục:**  
1. **Dynamic Hedging / Speculative Execution:** Nếu provider chính chưa phản hồi sau mốc p90 latency thông thường (~2.5s), client chủ động gửi thêm một request dự phòng (hedged request) sang provider thứ hai thay vì chờ hết trần timeout 8s; request nào về trước sẽ lấy kết quả và hủy (cancel) request còn lại.
2. **Cost-Aware Adaptive Load Balancing:** Phân bổ tải theo trọng số (weighted round-robin) dựa trên latency thực tế và chi phí thay vì cố định một ladder cứng nhắc, giúp san sẻ áp lực và tối ưu hóa chi phí.

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9117 | 0.9640 | 0.634 | 173 | 0 | 6 | 0.927 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ở giá trị starter 0.80, hệ thống bị **138 câu trả lời sai** (`wrong_answers_by_reason: {'other_question': 138}`) do nhận nhầm câu hỏi khác nghĩa. Tăng lên 0.92 loại bỏ triệt để lỗi này (`wrong_answers = 0`) trong khi vẫn đảm bảo `cache_hit_ratio = 63.4%` (nếu đặt quá cao 0.97, hit ratio tụt xuống 55.2%). |
| ttl_s | 45 | Ở mức starter 600s hoặc 300s, khi bảng giá ship đổi ngầm ở $t=480\text{s}$, hệ thống trả về tới 95 câu lỗi thời (`outdated_answers = 95` vi phạm trần 15). Giảm xuống 45s giúp câu trả lời tự hết hạn và cập nhật giá mới nhanh chóng, ép `outdated_answers` xuống chỉ còn **6** (đạt yêu cầu $\le 15$). |
| max_stale_s | 240 | Đợt sự cố sập toàn bộ provider kéo dài từ $t=240\text{s}$ đến $t=360\text{s}$ (120 giây). Giá trị starter `max_stale_s: 0` khiến `faq_availability` sụt xuống 86.34% và phục vụ 0 stale answer. Đặt 240s giúp cơ chế stale-if-error kích hoạt phục vụ 173 câu trả lời cũ, nâng `faq_availability` lên **96.4%** (vượt chuẩn 93%). |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng similarity:**  
   Chạy C1 với các ngưỡng khác nhau:
   - `similarity_threshold = 0.80`: hit ratio = 0.647, **138 câu trả lời sai** (`wrong_answers_by_reason: {'other_question': 138}`), cost/turn = 0.972.
   - `similarity_threshold = 0.92`: hit ratio = 0.634, **0 câu trả lời sai**, cost/turn = 0.927.
   - `similarity_threshold = 0.97`: hit ratio = 0.552, **0 câu trả lời sai**, cost/turn = 0.984.  
   Ở ngưỡng 0.80, các câu hỏi có ngữ cảnh hoàn toàn khác biệt nhưng embedding có độ tương đồng hình thức (như "Chính sách hoàn tiền mất bao lâu?" vs "Quy định đổi trả hàng bị lỗi") bị gộp nhầm, khiến khách hàng nhận thông tin sai lệch nghiêm trọng. Ngưỡng 0.92 là điểm cân bằng lý tưởng giữa độ chính xác và tỷ lệ tận dụng cache.

2. **TTL:**  
   Chạy C1 với các mức TTL:
   - `ttl_s = 30`: outdated_answers = 2, hit ratio = 0.592, cost/turn = 0.956.
   - `ttl_s = 60`: outdated_answers = 3, hit ratio = 0.652, cost/turn = 0.916.
   - `ttl_s = 300`: outdated_answers = **95** (vi phạm tiêu chí $\le 15$), hit ratio = 0.731, cost/turn = 0.863.  
   Webhook chỉ phát huy tác dụng khi tài liệu thay đổi có sự kiện chủ động từ hệ thống quản lý nội dung (như đổi chính sách hoàn tiền ở $t=150\text{s}$). Khi đối tác thứ ba âm thầm thay đổi biểu phí vận chuyển ở $t=480\text{s}$ (silent update) mà không bắn webhook, cache hoàn toàn không hay biết. Do đó, một TTL đủ ngắn (ở đây là 45s) là chốt chặn phòng thủ duy nhất để bắt buộc cache phải revalidate với nguồn dữ liệu gốc, khống chế số câu trả lời cũ ở mức an toàn ($\le 15$).

3. **Stale-if-error:**  
   Khi chạy với `max_stale_s: 0` (tắt cơ chế stale-if-error):
   - `stale_served` tụt về 0 và `faq_availability` giảm mạnh từ 96.40% xuống còn **86.34%** (thất bại so với yêu cầu tối thiểu 93%).
   - Trong suốt 120 giây toàn bộ provider ngừng hoạt động ($t=240\text{s}$ - $360\text{s}$), người dùng đặt câu hỏi FAQ hoàn toàn không nhận được câu trả lời nào dù nội dung câu hỏi không hề thay đổi. Cơ chế stale-if-error cho phép phục vụ nội dung cũ trong tình huống khẩn cấp, bảo vệ trải nghiệm người dùng mà không làm sai lệch tính đúng đắn.

Trả lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") KHÔNG được cache, kể cả khi cache theo từng user?**  
  1. *Rủi ro rò rỉ dữ liệu riêng tư (Data Leakage & PII):* Embedding của câu hỏi cá nhân có thể tương đồng với câu hỏi của người khác; nếu có sai sót về logic scope, người dùng B có thể đọc được thông tin đơn hàng nhạy cảm (địa chỉ, số điện thoại, giá trị món hàng) của người dùng A.  
  2. *Dữ liệu thay đổi liên tục theo thời gian thực (High Volatility):* Trạng thái vận chuyển đơn hàng là thông tin biến động động (dynamic state). Trả về dữ liệu từ cache có thể cung cấp trạng thái cũ (ví dụ: hàng vẫn hiển thị "đang xử lý" trong khi thực tế đã "giao thành công"), gây hiểu lầm và mất niềm tin của khách hàng.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**  
  Không thể chỉ dựa vào SLO Availability tổng thể ở tầng gateway người dùng. Cần thiết lập alert dựa trên các chỉ số nội bộ sau:
  1. **`provider_call_error_rate` hoặc `upstream_5xx_rate`:** Tỷ lệ lỗi trực tiếp từ các cuộc gọi ra provider bên dưới (backend error rate).
  2. **`circuit_breaker_state`:** Cảnh báo ngay khi một hoặc nhiều Circuit Breaker chuyển sang trạng thái `OPEN`.
  3. **`stale_response_ratio` / `cache_stale_served_count`:** Tỷ lệ câu trả lời phục vụ người dùng bằng dữ liệu cũ (stale responses). Nếu tỷ lệ này tăng đột biến vượt ngưỡng (ví dụ $> 5\%$ trong 5 phút), hệ thống phải kích hoạt Page cảnh báo đội ngũ on-call ngay lập tức rằng backend đang tê liệt và hệ thống chỉ đang "cầm cự" bằng bộ nhớ đệm.
