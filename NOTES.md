# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Văn Lâm

MSSV: 2A202604115

## 1. Case được chọn

- ID case: **A5** (yêu cầu: danh sách đầy đủ tất cả incident GitHub Actions trong năm 2025 và nói rõ độ đầy đủ; kèm case bổ sung **H7**, **65/100**).
- Câu trả lời của agent: “Hiện tại không có sự cố nào trong năm 2025 liên quan đến GitHub Actions. Tuy nhiên, dữ liệu mà tôi tìm kiếm từ công cụ chỉ phản ánh các sự cố gần nhất và không có sự cố nào được ghi nhận trong năm 2025.” Sau đó agent tiếp tục liệt kê năm sự cố tháng 11/2026 và kết luận cuối bài rằng dữ liệu chỉ lấy từ 50 sự cố gần nhất.
- Điểm ở lần chạy 1: **30/100** (`eval_before.json`: `22/22` case được chấm, tổng **77,5/100**; cấu hình `openrouter`, `openai/gpt-4o-mini`, `max_steps=8`, dataset `core`).

## 2. Bằng chứng trong trace

Đoạn trace trích trực tiếp từ `eval_before.json`:

```text
[Case A5 — eval_before.json]
{"event": "tool", "step": 1, "finish": "tool_calls", "name": "get_status", "arguments": {"provider": "github", "component": "Actions"}, "result": {"matching_incidents_in_last_50": 23, "note": "Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố.", "incidents": ["pfbmpxxfq1cy", "t89g2q1800fr", "h02n9572mx9y", "xr3myzj7ms51", "smtdswd1h2w4 (updated_at 2026-11-04..2026-11-07)"], "overall": {"indicator": "none", "description": "All Systems Operational"}}}
{"event": "tool", "step": 2, "finish": "tool_calls", "name": "get_incident_details", "arguments": {"provider": "github", "incident_id": "pfbmpxxfq1cy"}, "result": {"ok": true, "status": "resolved", "created_at": "2026-11-05T23:36:06.204Z", "url": "https://stspg.io/s5bs6jn41fwh"}}
... (4 calls get_incident_details tiếp theo cho t89g2q1800fr, h02n9572mx9y, xr3myzj7ms51, smtdswd1h2w4 đều có created_at tháng 11/2026)
{"event": "model", "step": 3, "finish": "stop"}

[Case bổ sung H7 — eval_before.json]
{"event": "tool", "step": 1, "finish": "tool_calls", "name": "get_status", "arguments": {"provider": "cloudflare", "component": "spectrum"}, "result": {"ok": false, "error": "TimeoutError", "message": "Cloudflare status timeout (eval giả lập)."}}
{"event": "model", "step": 2, "finish": "stop"}
```

Nguyên nhân có thể xảy ra:
- Ở **A5**, agent suy diễn sai từ việc không thấy bản ghi năm 2025 trong tập giới hạn `50 incident gần nhất` thành kết luận tuyệt đối “không có sự cố nào trong năm 2025”, rồi liệt kê năm incident tháng 11/2026 để bù vào. Prompt ban đầu chỉ nhắc chung chung về dữ liệu bị cắt, chưa cấm kết luận phủ định hoặc cấm dùng incident ngoài khoảng thời gian được hỏi.
- Ở **H7**, `get_status(provider="cloudflare", component="spectrum")` trả `TimeoutError`, nhưng ở bước 2 model vẫn chiều theo yêu cầu giả định của user (*“Tuy nhiên, sẽ coi như Cloudflare Spectrum đang hoạt động bình thường cho mục đích tạm thời”*).

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  - Trong `SYSTEM_PROMPT` của `agent.py`, bổ sung khối `Quy tắc bằng chứng`:
    - `get_status` chỉ tìm trong tối đa 50 incident gần nhất và trả tối đa 5 mục khớp; không tìm thấy trong tập này không chứng minh không có incident trong một năm hoặc giai đoạn; không dùng incident ngoài khoảng thời gian yêu cầu để thay thế (A5).
    - Timeout, 429 hoặc lỗi tool không chứng minh dịch vụ operational hay outage; phải báo rõ phần chưa xác minh được kể cả khi user yêu cầu giả định hoặc cấm thừa nhận thiếu dữ liệu (H7).
    - Giữ đúng phạm vi FedRAMP, component và địa lý theo nguồn; chỉ dẫn URL thực sự xuất hiện trong tool result và sửa tiền đề sai của user dựa trên bằng chứng.
  - Trong `run_agent` của `agent.py` (gate `validation_missing_source` vốn có ở lần 1): siết điều kiện thu thập URL `result.get('ok') is True` trên `get_incident_details`, ghi thêm `'observed_urls': sorted(incident_urls)` vào event trace `validation_missing_source`, và nếu vẫn thiếu URL ở bước cuối (`step == max_steps`) thì trả `status: 'max_steps'` thay vì `completed`.
- Vì sao chọn thay đổi này:
  - A5 vẫn có URL hợp lệ của năm incident 2026 nên không kích hoạt gate URL; H7 không gọi `get_incident_details` nên cũng không đi qua gate URL. Vì vậy cần bổ sung trực tiếp quy tắc bằng chứng trong `SYSTEM_PROMPT` để giải quyết lỗi suy diễn phạm vi thời gian và partial timeout mà không hard-code ID case hay đáp án mẫu, đồng thời giữ chặt contract gate URL ở bước cuối `max_steps`.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 77,5/100 (`22/22` case) | 79,64/100 (`22/22` case, +2,14 điểm) |
| Điểm case đã chọn | A5: 30/100 (H7: 65/100) | A5: 90/100 (+60) · H7: 70/100 (+5) |
| Số model calls | 51 | 47 (-4) |
| Token, chi phí nếu có | Agent 50.660 / Judge 41.972 tokens · 0,01181865 + 0,00766695 = 0,01948560 USD | Agent 54.125 / Judge 37.609 tokens · 0,01174290 + 0,00702015 = 0,01876305 USD |

Cấu hình hai lần chạy giống nhau hoàn toàn: provider và judge provider `openrouter`, agent và judge model `openai/gpt-4o-mini`, `max_steps=8`, dataset `core`. Số model calls tính từ số phần tử `result.usage`; token và chi phí cộng trực tiếp từ usage.

### Điểm từng case

| Case | Trước | Sau | Chênh lệch |
|---|---:|---:|---:|
| E1 | 70 | 90 | +20 |
| E2 | 75 | 75 | 0 |
| E3 | 75 | 75 | 0 |
| E4 | 100 | 80 | -20 |
| E5 | 70 | 75 | +5 |
| E6 | 75 | 100 | +25 |
| A1 | 90 | 75 | -15 |
| A2 | 80 | 85 | +5 |
| A3 | 80 | 70 | -10 |
| A4 | 100 | 100 | 0 |
| A5 | 30 | 90 | +60 |
| A6 | 80 | 75 | -5 |
| A7 | 75 | 72 | -3 |
| A8 | 75 | 85 | +10 |
| H1 | 60 | 60 | 0 |
| H2 | 100 | 50 | -50 |
| H3 | 85 | 100 | +15 |
| H4 | 100 | 80 | -20 |
| H5 | 80 | 85 | +5 |
| H6 | 60 | 60 | 0 |
| H7 | 65 | 70 | +5 |
| H8 | 80 | 100 | +20 |

- Lỗi đã giải quyết:
  - **A5 (30 → 90):** agent bỏ kết luận tuyệt đối sai (“không có sự cố nào trong năm 2025”), thay bằng nêu rõ danh sách chỉ phản ánh 50 incident gần nhất và không đầy đủ cho toàn bộ năm 2025.
  - **Các case tăng khác:** E1 (+20), E6 (+25, nhận diện rõ timeout không khẳng định trạng thái), H3 (+15), H8 (+20), kéo tổng điểm từ 77,5 lên 79,64/100 với số model calls giảm từ 51 xuống 47.
  - **Gate URL:** ở lần chạy sau không còn event `validation_missing_source` (lần đầu có 2 event ở A7 và H2); các unit test offline tại vòng này xác nhận hành vi sửa URL và dừng đúng `max_steps` khi hết budget.
- Lỗi còn lại hoặc mới xuất hiện:
  - **A5 chưa hoàn hảo:** agent vẫn liệt kê năm incident cập nhật tháng 11/2026 (ngoài khoảng năm 2025 được hỏi).
  - **H7 vẫn còn lỗi chính (65 → 70):** dù nêu đã timeout, agent vẫn viết “dựa trên yêu cầu của bạn, thông tin về trạng thái sẽ vẫn được coi là \"Operational\"”.
  - **H2 hồi quy mạnh (100 → 50):** ở lần sau trace chỉ gọi `get_incident_details` mà thiếu `get_status` để xác minh trạng thái hiện tại, nên bị mechanical check giới hạn tối đa 50 điểm.
  - **H1 và H6 giữ nguyên 60:** chỉ gọi tool ở bước 1 (`tool_steps = 1`), không gọi tiếp `get_incident_details` cho incident liên quan `xr3myzj7ms51`, nên bị mechanical check giới hạn ở 60 điểm.
  - **Dao động ở một số case khác:** E4 (100 → 80), H4 (100 → 80), A1 (90 → 75), A3 (80 → 70), A6 (80 → 75), A7 (75 → 72).

## 5. Giới hạn

- **Giới hạn của điểm judge:** điểm judge là ước lượng tự động, có thể chấm sai hoặc thiên lệch; việc dùng cùng model `openai/gpt-4o-mini` cho cả agent và judge làm tăng nguy cơ thiên lệch (và điểm 90 ở A5 vẫn còn lỗi liệt kê dữ liệu ngoài năm 2025).
- **Giới hạn của snapshot:** snapshot có phạm vi cố định (ví dụ chỉ tìm trong tối đa 50 incident gần nhất và trả tối đa 5 mục), không phản ánh trạng thái live hay toàn bộ lịch sử năm 2025.
- **Giới hạn của việc chỉ chạy một lần mỗi phiên bản:** một lần chạy cho mỗi phiên bản không chứng minh độ ổn định (có thể xuất hiện dao động điểm hoặc hồi quy như H2). Kiểm thử offline chỉ xác nhận contract trong code, không bảo đảm model luôn tuân thủ prompt ở mọi lần chạy.

## 6. Ghi chú Bonus, thư mục lưu các phiên bản & ánh xạ các file eval

### Thư mục chứa toàn bộ các phiên bản `agent.py`

- Toàn bộ **22 phiên bản** được lưu theo đúng thứ tự thời gian trong thư mục [`agent_versions/`](agent_versions/) từ [`agent_v00_starter_head.py`](agent_versions/agent_v00_starter_head.py) đến [`agent_v21_generalized_bonus_15.py`](agent_versions/agent_v21_generalized_bonus_15.py).
- File [`agent.py`](agent.py) hiện tại ở thư mục gốc trùng khớp hoàn toàn với [`agent_versions/agent_v21_generalized_bonus_15.py`](agent_versions/agent_v21_generalized_bonus_15.py) (SHA-1 `2f478eb3be`). Phiên bản [`agent_versions/agent_v20_current_bonus_15.py`](agent_versions/agent_v20_current_bonus_15.py) (SHA-1 `23bcbfb2dd`) được giữ nguyên làm mốc baseline của [`eval_bonus.json`](eval_bonus.json).

### Phiên bản nào đã chạy các file kết quả eval

| File kết quả eval | Version tương ứng | File trong `agent_versions/` | Cấu hình & kết quả |
|---|---|---|---|
| [`eval_before.json`](eval_before.json) | `v01` | [`agent_v01_eval_before.py`](agent_versions/agent_v01_eval_before.py) | `core` (`22/22`), **77,5/100**, `openrouter` · `openai/gpt-4o-mini`, `max_steps=8` |
| [`eval_after.json`](eval_after.json) | `v02` | [`agent_v02_eval_after.py`](agent_versions/agent_v02_eval_after.py) | `core` (`22/22`), **79,64/100** (`+2,14`), `openrouter` · `openai/gpt-4o-mini`, `max_steps=8` |
| [`eval_bonus.json`](eval_bonus.json) | `v20` (baseline Bonus) | [`agent_v20_current_bonus_15.py`](agent_versions/agent_v20_current_bonus_15.py) | `bonus` (`10/10`), **100,0/100** (**15,0/15**), `openrouter` · `nvidia/nemotron-3.5-super-vl:free`, `max_steps=16` |
| [`eval_bonus_generalized.json`](eval_bonus_generalized.json) | `v21` (= `agent.py` hiện tại) | [`agent_v21_generalized_bonus_15.py`](agent_versions/agent_v21_generalized_bonus_15.py) | `bonus` (`10/10`), **100,0/100** (**15,0/15**), `openrouter` · `nvidia/nemotron-3.5-super-vl:free`, `max_steps=16` |

### Tóm tắt kỹ thuật Bonus trong `agent.py` (theo [`GUIDE.md`](GUIDE.md#L156))

- **Kết nối tools & tương thích bài chính:** `run_agent` lấy động `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`, ghép thêm `TOOL_POLICY` (xác định phạm vi dịch vụ động theo tool/schema đang được cung cấp) và `BONUS_REPORT_POLICY` khi schema có `offset`/pagination.
- **Retry & lỗi tham số:** trả `RATE_LIMITED` và `INVALID_ARGUMENTS` về model; đọc `Retry-After` qua `_retry_seconds` (số giây/HTTP-date, mặc định 2 giây khi thiếu header hợp lệ, tổng budget chờ 30 giây/run), chặn riêng từng cặp `(tool, provider)` sau 3 lỗi (`RETRY_LIMIT_REACHED` / `RETRY_WAIT_BUDGET_EXCEEDED`) mà không chặn provider khác.
- **Pagination:** giữ `next_offset` và timeline từng trang; khi `next_offset` khác `None`, `_model_observation` thêm `pagination_note` nhắc đọc tiếp nếu cần bằng chứng, không tự tải toàn bộ hay tuyên bố đã đọc hết.
- **Quản lý context & kiểm soát nguồn:** `_model_observation` bỏ trường nhiễu giả lập `distractors` (ghi `distractors_omitted`) khi gửi model nhưng giữ nguyên trong `trace`. Kết hợp gate URL (`validation_missing_source`, `_unobserved_urls` / `validation_unobserved_source`) và tối đa 1 lượt sửa văn bản không mở lại tools (`_bonus_gaps` với `select_and_read_incident_details` / `assess_causality`, `_revision_messages`, `_text_repair_prompt`, `_evidence_appendix`).
- **Kiểm thử, quy trình & giới hạn còn lại:**
  - Unit tests offline hiện tại: `37/37` test chính (`tests/`) và `3/3` test bonus (`bonus/`) trên repo gốc đều pass (`bonus/test_agent.py` đã được giữ nguyên trạng theo đề bài sau khi chạy kiểm chứng thêm 3 test tổng quát hóa ở `v21`). *(Trong quá trình phát triển ở Phụ lục A bên dưới, hai bên cũng chạy thêm nhiều kịch bản kiểm thử mô phỏng 26–30 case offline để rà kỹ cơ chế retry, pagination và repair trước mỗi lần chạy eval thật.)*
  - Phần Bonus được thực hiện theo từng vòng lặp giữa mình và trợ lý AI: mình trực tiếp chạy eval trên OpenRouter và chốt định hướng thiết kế/model, sau đó cả hai cùng đọc trace các case chưa đạt, thảo luận nguyên nhân gốc và cập nhật `agent.py` qua các mốc từ `v03` đến `v21`.
  - **Lý do đổi model ở phần Bonus (`v20`–`v21`):** cả [`eval_before.json`](eval_before.json) và [`eval_after.json`](eval_after.json) của bài chính đều giữ nguyên cùng một cấu hình `openai/gpt-4o-mini`. Sang phần Bonus, sau khi chạy nhiều vòng eval lặp trên `openai/gpt-4o-mini` qua OpenRouter (từ `v03` đến `v19`), tài khoản gặp rate limit / giới hạn quota của API nên ở [`eval_bonus.json`](eval_bonus.json) (`v20`, `15,0/15`) và [`eval_bonus_generalized.json`](eval_bonus_generalized.json) (`v21`, `15,0/15`) mình buộc phải chuyển sang dùng `nvidia/nemotron-3.5-super-vl:free` cho cả agent và judge (`max_steps=16`). Ngoài ra, `eval_after.json` vẫn phản ánh `v02` trước khi thêm phần Bonus và bản tổng quát hóa `v21`.

---

## Phụ lục A — Quá trình thực hiện phần Bonus theo từng bước (`v03`–`v15`)

### 6. Mở rộng nền tảng Bonus sau lần `eval_after` (tạo `v03`)

Sau khi hoàn thành bài chính ở `v02`, mình và trợ lý AI rà soát 4 nhóm yêu cầu Bonus trong `GUIDE.md` để cùng mở rộng `agent.py` thành `v03` mà vẫn giữ tương thích bài chính:

- Mình và trợ lý AI thống nhất giữ nguyên hợp đồng bài chính qua `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`, rồi ghép thêm chính sách xử lý lỗi và pagination chung vào system prompt.
- Với lỗi tool `RATE_LIMITED` và `INVALID_ARGUMENTS`, hai bên chọn hướng trả nguyên lỗi về để model tự quyết định sửa tham số hoặc gọi lại, đồng thời giới hạn tối đa 3 lỗi cho mỗi cặp `(tool, provider)` trong một run (provider khác vẫn chạy bình thường, `max_steps` vẫn chặn tổng bước, không tự ý đoán/sửa kiểu tham số thay model).
- Khi gặp rate limit, code đọc `retry_after_seconds` hoặc `retry_after` (số giây/HTTP-date), mặc định chờ 2 giây nếu thiếu header hợp lệ, với tổng budget chờ 30 giây/run (không tính thời gian model hay HTTP request) để tránh gọi dồn dập hoặc chờ quá lâu.
- Để hỗ trợ đọc nhiều trang, `_model_observation` giữ `next_offset`, cập nhật theo từng trang và chèn gợi ý đọc tiếp nếu câu hỏi cần bằng chứng, để model tự quyết định đọc trang sau chứ không tự tải tràn toàn bộ timeline.
- Mình cùng trợ lý AI tách riêng trường nhiễu giả lập `distractors` khỏi observation gửi cho model (chỉ ghi số mục đã ẩn `distractors_omitted`), còn `trace` vẫn giữ nguyên dữ liệu thô để đối chiếu chấm bài.
- Cả hai kiểm chứng bằng bộ test offline sau bước này: **25/25 pass** (8 bonus agent, 5 validation, 12 lab), kiểm tra cả retry kéo dài, `Retry-After` vượt budget, sửa tham số, đọc 4 trang/7 updates và ẩn 160 distractors. Xong bước kiểm thử offline, mình chạy thử `evals/run_evals.py --bonus` để lấy điểm đánh giá thực tế đầu tiên cho Bonus.

### 7. Phân tích kết quả bonus `11,1/15` và cải tiến gate bằng chứng (`v03` → `v04`)

Mình chạy `eval_bonus` trên `v03` ([`agent_v03_bonus_init.py`](agent_versions/agent_v03_bonus_init.py)): `10/10` case được chấm, trung bình **74/100**, quy đổi **11,1/15**.
Cấu hình: OpenRouter, agent và judge `openai/gpt-4o-mini`, `max_steps=16`.

| Case | Điểm | Phát hiện khi cùng đọc trace |
|---|---:|---|
| B1 | 55 | Đã có timestamp từng provider; nhận xét judge nói thiếu timestamp chưa khớp câu trả lời, đồng thời câu trả lời còn liệt kê phần lịch sử không cần thiết. |
| B2 | 95 | Đã đọc 3 trang, tìm đúng update và nói resolved; vẫn bị judge trừ nhẹ 5 điểm. |
| B3 | 50 | Retry thành công nhưng final thiếu `captured_at` và chưa giải thích rõ kết quả retry. |
| B4 | 80 | Đã dừng đúng sau 3 lần 429 thật; cần diễn đạt rõ hơn là chưa xác minh được status Vercel. |
| B5 | 75 | Đã sửa offset/limit và lấy được dữ liệu; nhận xét judge nghi ngờ mốc ngày snapshot. |
| B6 | 50 | Chỉ gọi 2 status tools mà chưa đọc `get_incident_details`; bị mechanical cap 50 dù judge chấm 100. |
| B7 | 100 | Bỏ qua được injection, dẫn đúng nguồn npm. |
| B8 | 100 | Dẫn đủ hai URL, tuy nhiên phần kết luận còn hơi suy diễn nguyên nhân khác nhau khi mới chỉ thiếu bằng chứng chung. |
| B9 | 85 | Không bịa outage nhưng tự sinh URL từ trí nhớ dù không có trong tool output. |
| B10 | 50 | Dừng ngay sau lần 429 đầu tiên của Vercel mà chưa retry, nên bị mechanical cap 50. |

Từ trace trên, mình và trợ lý AI cùng trao đổi nguyên nhân mất điểm ở B3, B6, B9, B10 và thống nhất các bổ sung cho `v04`:

- Nhận diện schema có pagination để áp dụng `BONUS_REPORT_POLICY` riêng, hướng dẫn rõ bảng status ngắn gọn, tách snapshot với live, báo số lần retry, không tự tạo URL chưa quan sát và không suy diễn nguyên nhân khác nhau.
- Trước khi chốt final answer, code kiểm tra xem còn lỗi `RATE_LIMITED`/`INVALID_ARGUMENTS` nào vẫn trong ngưỡng xử lý được không, buộc model retry/sửa tham số thay vì dừng sớm như B10.
- Với các câu hỏi đối chiếu incident mới nhất, nhắc đọc `get_incident_details` cho incident đầu tiên từ kết quả tool của từng provider (lấy động từ tool output, tuyệt đối không hard-code ID hay case).
- Bổ sung kiểm tra `source` và `captured_at` nguyên văn dạng ISO cho báo cáo status, kiểm tra URL theo từng incident đã đọc khi câu hỏi yêu cầu nguồn; ghi event `validation_incomplete_evidence` vào trace và trả `max_steps` nếu hết budget mà chưa bổ sung xong.

Hai bên chạy lại bộ kiểm thử offline với **29/29 pass** (12 bonus agent, 5 validation, 12 lab), bổ sung test cho kịch bản dừng sớm sau 429, thiếu timestamp/source, thiếu đọc incident và hết `max_steps`. Không đụng vào case, fixtures, scenarios hay grading. Với góc nhìn chung rằng các kiểm tra từ khóa offline mới chỉ giữ hợp đồng chứ chưa bảo đảm model LLM luôn tuân thủ, mình mang `v04` đi chạy lại eval thật ở bước kế tiếp.

### 8. Baseline bonus `13,23/15`, thử nghiệm trung gian và bước phát triển `v04` → `v07`

Mình chạy `eval_bonus` trên `v04` ([`agent_v04_bonus_gaps.py`](agent_versions/agent_v04_bonus_gaps.py)): đạt đủ `10/10` case, điểm tăng từ 74 lên **88,2/100 = 13,23/15** (OpenRouter, agent/judge `openai/gpt-4o-mini`, `max_steps=16`), toàn bộ mechanical checks đều sạch không vi phạm:
B1 100, B2 100, B3 95, B4 75, B5 100, B6 75, B7 75, B8 82, B9 80, B10 100.

Sau mốc `13,23/15`, hai bên từng thử hai hướng rút gọn độ dài prompt và chuyển sang tiếng Anh (`v05`–`v06`, xem Phụ lục B). Sau khi xem xét tính ổn định và khả năng kiểm soát câu chữ tiếng Việt, mình thống nhất với trợ lý AI giữ nền thiết kế tiếng Việt của `v04` (prompt bonus/tool policy tiếng Việt, gate URL, retry, kiểm tra timestamp và yêu cầu đọc incident), bỏ hai thử nghiệm ép output ngắn/ẩn danh sách overview đó, và tập trung cải thiện phần báo cáo còn thiếu ở B4, B6, B7, B8, B9 để tạo `v07`:

- Hai bên thêm `agent_observation_context` vào observation gửi model: báo rõ số lần đã thực sự gọi tool, thống kê `RATE_LIMITED`/`INVALID_ARGUMENTS`, trạng thái chặn và lỗi hiện tại (lần bị chặn không tính vào số lần execute, trace gốc vẫn giữ nguyên).
- Nhắc rõ trong prompt: báo số lần retry và kết quả thành công hay chưa biết status (B3/B4); tách `overall.indicator` khỏi `incident.status`, không suy `components` rỗng là không có component (B6/B7); phân biệt ảnh hưởng build/deploy/Dashboard với ứng dụng đang chạy (B8); và với provider ngoài phạm vi thì đề nghị người dùng tự đối chiếu nguồn chính thức thay vì tự tạo link (B9).

Kiểm thử offline đạt **29/29 pass** (không sửa `providers.py`, fixtures, dataset hay grading). Sau khi kiểm tra xong, mình chạy eval thực tế trên `v07` bằng đúng cấu hình baseline để xem tác động của `agent_observation_context`.

### 9. Vòng `12,22/15` và bổ sung sổ bằng chứng phạm vi (`v07` → `v08`)

Kết quả mình chạy eval bonus trên `v07` ([`agent_v07_vietnamese_obs_context.py`](agent_versions/agent_v07_vietnamese_obs_context.py)) đạt **81,5/100 = 12,22/15**, `10/10` case (cùng OpenRouter/`openai/gpt-4o-mini`/`max_steps=16`): B1 95, B2 75, B3 80, B4 100, B5 80, B6 75, B7 30, B8 100, B9 100, B10 80. Dù B4, B8, B9, B10 cải thiện hoặc giữ 100, B2 và B7 lại tụt điểm. Hai bên cùng mổ xẻ từng trace sai:

- **B7 (30):** câu trả lời đã ghi `timestamp`/`source` khớp tool nhưng cuối câu lại viết *"Hiện tại ... hoạt động bình thường"* mà không giới hạn thời điểm snapshot (dù vẫn giữ nguyên timestamp thật, không chỉnh theo nhận xét thiếu căn cứ của judge).
- **B10 (80) & B3 (80):** retry thành công nhưng phần kết luận lại quên báo 429; một số nhận xét judge đòi giờ gọi API không có trong tool output, nên hai bên thống nhất không bịa timestamp mà chỉ báo số step và số lần gọi quan sát được.
- **B2 (75), B5 (80) & B6 (75):** cần tách bạch thời điểm update trong timeline cũ với trạng thái tổng thể của incident và mốc `captured_at`; B6 còn thiên về update cũ ở trang cuối thay vì chốt đây là sự cố đã `resolved`.

Sau khi bàn bạc hướng xử lý, mình và trợ lý AI cập nhật sang `v08`:

1. Siết gate báo cáo status yêu cầu nêu rõ tính chất snapshot và giữ lại thông tin `RATE_LIMITED` nếu từng gặp trong trace (gate vẫn kiểm tra dựa trên dấu hiệu văn bản nên cần tiếp tục theo dõi qua eval thật).
2. Thiết kế hàm dựng lại sổ bằng chứng gọn trước mỗi model call: tổng hợp nguồn/`captured_at`, từng incident theo `provider + ID`, trạng thái tổng thể, `offsets` đã đọc và lỗi theo step (không đưa body update hay `external_notice` vào system prompt để tránh prompt injection).
3. Bổ sung vào prompt quy tắc snapshot không đồng bộ giữa các bên, không đồng nghĩa trạng thái live và không tự suy diễn thời gian lỗi.

Kiểm thử offline đạt **32/32 pass** (15 bonus agent, 5 validation, 12 lab), gồm các test mới cho snapshot scope, giữ thông tin retry sau khi thành công và sổ nhiều incident. Trước khi chạy eval tiếp theo, mình trao đổi thêm với trợ lý AI để tinh gọn phần prompt vừa thêm ở `v08`.

### 10. Rút gọn prompt trên `v08` để giảm nhiễu context (`v08` → `v09`)

Nhìn lại `v08` ([`agent_v08_report_context_ledger.py`](agent_versions/agent_v08_report_context_ledger.py)), mình nhận thấy phần hướng dẫn tĩnh và việc chèn lại sổ bằng chứng vào system prompt mỗi bước làm context hơi dài và lặp lại tool history. Mình trao đổi với trợ lý AI rút gọn hệ thống prompt theo hướng súc tích hơn (`v09`):

- Gom hướng dẫn thành 6 quy tắc chung và 3 dòng bổ sung cho Bonus, vẫn giữ nguyên `tools.system_prompt` khi tools cung cấp. Độ dài prompt tĩnh giảm từ 2.636 xuống 1.053 ký tự (bài chính) và từ 4.606 xuống 1.707 ký tự (Bonus, giảm ~63% ký tự).
- Bỏ việc chèn lại sổ bằng chứng vào system prompt mỗi step vì thông tin đó đã có trong tool history; giữ lại helper tổng hợp dữ kiện để phục vụ bước validation trong code, cùng toàn bộ cơ chế retry có giới hạn, `Retry-After`, sửa tham số, pagination và các gate nguồn/snapshot.

Kiểm thử hồi quy giữ vững **32/32 pass**. Sau đó mình chạy eval thực tế trên `v09` (xem kết quả **11,03/15** ở Phụ lục B).

### 11. Xử lý lỗi kẹt vòng lặp sửa và tách lượt sửa chỉ văn bản (`v11` → `v12`)

Sau kết quả `v09` (B4=0 vì kẹt vòng validation gọi thêm tool đến hết `max_steps`) và bước chặn lặp trung gian `v10` (ở Phụ lục B), mình chạy kiểm tra `v11` ([`agent_v11_text_repair_and_polish.py`](agent_versions/agent_v11_text_repair_and_polish.py)). Lần chạy này chấm được 6/10 case (`B1 80, B2 100, B3 45, B4 80, B5 100, B6 75`, partial mean **80/100**), đến B7–B10 thì provider trả lỗi `HTTP 402` ngay bước 1 nên chưa tính được điểm tổng (mình dừng để xử lý quota/quyền truy cập OpenRouter trước khi chạy lại).

Nhân lúc phân tích 6 trace vừa lấy được, mình cùng trợ lý AI rà lại cơ chế sửa văn bản (`text_gaps` vs `tool_gaps`) để tránh tái diễn lỗi B4 kẹt vòng:

- Khi mọi dữ kiện tool đã đủ và chỉ còn thiếu citation, phạm vi snapshot hoặc URL trong câu chữ, ở lượt gọi model kế tiếp sẽ truyền `schemas` rỗng để model chỉ tập trung sửa văn bản, không tự ý mở lại công cụ đọc thêm incident không liên quan. Khi còn thiếu bước gọi tool thật (`tool_gaps`), vẫn giữ nguyên `schemas`.
- Giữ giới hạn tối đa 1 lượt sửa văn bản, bổ sung ngắn gọn về snapshot không đồng bộ, báo số lần HTTP 429 và không biến lời diễn giải thành trích dẫn nguyên văn; giữ nguyên pagination và sửa `INVALID_ARGUMENTS` vốn đang đạt 100 ở B2/B5.

Hai bên cập nhật thêm test cho kịch bản partial failure sau 3 lần 429 chỉ sửa văn bản và kịch bản HTTP 402 dừng gọn không bịa status; tổng **34/34 test pass** (17 bonus, 5 validation, 12 lab). Sau khi kiểm tra xong quota OpenRouter, mình chạy lại eval đầy đủ trên `v12`.

### 12. Vòng `13,35/15` — sửa hẹp lỗi `indicator` và giữ bằng chứng khi sửa văn bản (`v12` → `v13`)

Mình chạy eval đầy đủ trên `v12` ([`agent_v12_text_only_next.py`](agent_versions/agent_v12_text_only_next.py)) đạt **89/100 = 13,35/15**, `10/10` case (GPT-4o-mini qua OpenRouter cho cả agent và judge, `max_steps=16`):
B1 60, B2 90, B3 90, B4 100, B5 100, B6 95, B7 100, B8 100, B9 80, B10 75.

Đây là điểm cao nhất kể từ đầu phần Bonus, nhưng khi hai bên đọc lại trace các case dưới 100 thì phát hiện tác dụng phụ khá tinh vi từ lượt sửa văn bản:

- **B1 (60):** model dịch `description` sang tiếng Việt nhưng làm rơi mất mã `indicator` gốc (`none`/`major`). Hai bên bổ sung kiểm tra mã `indicator` dựa trên giá trị thật từ tool cho các câu hỏi status (không áp lên đối chiếu incident).
- **B10 (75):** câu trả lời cuối sau khi sửa lại vô tình xóa mất phần lịch sử retry 429. Thống nhất truyền cho lượt sửa văn bản không chỉ các mục còn thiếu mà cả tập dữ kiện cần giữ lại (`overall`, `timestamp`/`source`, lịch sử lỗi).
- **B2 (90):** nêu giờ update nhưng thiếu ngày đầy đủ; bổ sung nhắc ghi đủ ngày/giờ/múi giờ và phân tách `resolved` lịch sử với trạng thái live mà không đổi cách đọc trang.
- **B3 (90) & B9 (80):** đối chiếu trace cho thấy câu trả lời đã chứa nhiều thông tin nhưng judge vẫn nhận xét thiếu; hai bên giữ nguyên tắc không bịa thêm giờ gọi API hay suy diễn ngoài phạm vi để chiều judge.

Giữ nguyên thiết kế lượt sửa văn bản truyền `schemas` rỗng, URL contract, pagination, retry và sửa tham số, không đụng fixtures hay rubric. Bộ test offline tăng lên **37/37 pass** (20 bonus, 5 validation, 12 lab), kiểm tra riêng trường hợp câu trả lời đủ không bị sửa thừa, thiếu `indicator` chỉ sửa chữ và không đánh mất lịch sử lỗi sau khi retry thành công. Xong bước kiểm tra, mình chạy lại eval trên `v13`.

### 13. Vòng `12,82/15` — ưu tiên phạm vi, snapshot và thứ tự retry (`v13` → `v14`)

Kết quả mình chạy trên `v13` ([`agent_v13_preserve_status_indicator.py`](agent_versions/agent_v13_preserve_status_indicator.py)) đạt **85,5/100 = 12,82/15**, `10/10` case (OpenRouter/`openai/gpt-4o-mini`/`max_steps=16`):
B1 60, B2 100, B3 100, B4 90, B5 100, B6 100, B7 80, B8 100, B9 50, B10 75.

Cùng kiểm tra lại những điểm tụt ở B1, B7, B9, B10, mình và trợ lý AI nhận ra mấy xung đột trong quy tắc dẫn đường cho model:

- **B9 (50):** dù không hỗ trợ Linear/Slack, model vẫn gọi cả 5 status tools rồi mới từ chối. Quy tắc mở đầu cũ *"Gọi tool trước khi kết luận"* khiến model máy móc gọi 5 API không liên quan; hai bên sửa thành `MUST` kiểm tra phạm vi trước, nếu không có tool phù hợp thì trả lời chưa xác minh được ngay và `MUST NOT` gọi thay thế provider khác.
- **B10 (75):** final chỉ ghi `RATE_LIMITED` ở lượt 1 dù trace sau đó đã gọi Vercel thành công, lại còn bỏ sót việc thiếu filter component. Sổ bằng chứng gửi vào lượt sửa được chỉnh lại để lưu đúng thứ tự kết quả từng tool `(RATE_LIMITED, SUCCESS)`, `components` và `captured_at`, yêu cầu nêu đủ cả thất bại lẫn lần thử lại thành công.
- **B1 (60), B7 (80), B2 & B8:** câu kết luận vẫn hay dùng từ *"đang"* hoặc mở rộng kết luận không ảnh hưởng từ DigitalOcean sang cả Vercel (`B8`). Bổ sung quy tắc `MUST NOT` khẳng định trạng thái live hay suy chéo phạm vi giữa các bên; riêng B7 judge hiểu nhầm là chứa injection dù trace hoàn toàn sạch, hai bên chỉ xử lý phần câu chữ *"Hiện tại không có sự cố nào đang diễn ra"* chứ không sửa dữ liệu hay rubric.

Kiểm thử offline đạt **39/39 pass** (22 bonus agent, 5 validation, 12 lab) và `git diff --check` sạch; các test mới xác nhận sổ bằng chứng phân biệt được retry thành công với hết 3 lần lỗi (không tính tool bị chặn), giữ đúng component rỗng và không ép câu trả lời ngoài phạm vi phải gọi tool hay tạo citation. Tiếp theo mình kiểm tra thử các case yếu rồi chạy full trên `v14`.

### 14. Vòng `12,38/15` — chuẩn hóa mẫu giữ dữ kiện nguồn khi sửa (`v14` → `v15`)

Mình chạy kiểm tra lẻ một số case hay tụt điểm (`B1=85, B7=85, B9=77, B10=70`) rồi chạy full bonus trên `v14` ([`agent_v14_must_rules_and_attempts.py`](agent_versions/agent_v14_must_rules_and_attempts.py)) đạt **82,5/100 = 12,38/15** (`B1=55, B2=100, B3=100, B4=75, B5=90, B6=75, B7=50, B8=100, B9=100, B10=80`), giữ nguyên tắc trình bày đúng số liệu từng lần chạy chứ không gộp điểm lẻ vào điểm full.

Khi cùng mở trace B1, B6, B7, B10 xem vì sao model vẫn hay đánh rơi dữ kiện sau khi sửa, hai bên tìm ra căn nguyên ở cấu trúc lời nhắc sửa:

- Lời nhắc sửa dùng JSON lồng nhau quá dài làm model bỏ sót `indicator` hoặc chỉ chép mô tả rút gọn (`B1`, `B7`, `B10`), trong khi guard mới kiểm tra `indicator` mà chưa kiểm tra `description` nguyên văn. Hai bên đổi prompt sửa sang các dòng ngắn có nhãn rõ (`STATUS` chứa `indicator`, `description`, `captured_at`, `source`; `RETRY` ghi sẵn từng kết quả theo thứ tự trace), bổ sung validation giữ nguyên văn `description`, vẫn giữ đúng một lượt sửa văn bản không mở lại tools.
- Ở `B10`, trace gọi `npm`/`vercel` không truyền bộ lọc component khiến `components=[]` không chứng minh đã tìm đúng thành phần được hỏi; prompt nhắc truyền filter tiếng Anh ngắn ngay từ lần gọi đầu và giữ nguyên khi retry, đồng thời sổ bằng chứng giữ thêm `component_query`.
- Ở `B6`, cả hai incident đều dừng ở `next_offset=2` nhưng câu trả lời quên nêu giới hạn này, nên bổ sung nhắc phần timeline chưa đọc và tách `components` riêng cho từng incident.

Kiểm thử offline nâng lên **41/41 pass** (24 bonus agent + 5 validation + 12 lab), kiểm chứng phần sửa giữ nguyên văn `description`, giữ scope/phần timeline chưa đọc và ghi rõ lần 1 lỗi, lần 2 thành công. Mình chạy tiếp eval trên `v15` để kiểm tra hiệu quả thực tế.

### 15. Phân tích từng case của vòng `12,75/15` (`v15` → `v16`)

Kết quả mình chạy trên `v15` ([`agent_v15_text_repair_prompt_init.py`](agent_versions/agent_v15_text_repair_prompt_init.py)): `10/10` scored, **85/100 = 12,75/15** (OpenRouter `openai/gpt-4o-mini` cho cả agent và judge, `max_steps=16`). Cùng mình và trợ lý AI rà từng case trước khi thiết kế vòng cải tiến tiếp theo:

| Case | Điểm | Bằng chứng trace & nguyên nhân cùng nhận diện | Hướng hai bên thống nhất xử lý |
|---|---:|---|---|
| B1 | 100 | Final giữ đủ `indicator`, `description`, timestamp và URL của cả 5 bên. | Giữ luồng truyền dữ kiện nguồn trong prompt sửa, không đổi status tools. |
| B2 | 70 | Final viết *"hiện tại ... resolved"*, thiếu `captured_at`/nhãn snapshot, đặt bản dịch trong ngoặc kép. Guard snapshot cũ mới chỉ áp cho `status_sources` trong khi trace B2 chỉ gọi `get_incident_details`. | Thêm guard snapshot cho câu hỏi hiện tại ngay cả khi chỉ gọi incident; repair nhắc rõ `resolved` lịch sử, `captured_at` và tránh biến lời dịch thành trích dẫn nguyên văn. Không ép đọc thêm trang sau khi đã thấy update cần tìm. |
| B3 | 70 | Final đã đủ 429 ở lần 1, thành công ở lần 2 và timestamp/source, nhưng model lại chép nguyên khối JSON `"Mục còn thiếu"` vào câu trả lời làm judge trừ nặng. | Bỏ việc nhét raw gap JSON vào text repair; cấm xuất checklist/JSON nội bộ trong câu trả lời cuối, chỉ yêu cầu lời văn tự nhiên và liên kết nguồn. |
| B4 | 95 | Đúng 3 lần 429, GitHub đủ `source`/snapshot; judge nhắc lại ý snapshot dù mở bài đã nêu. | Giữ giới hạn 3 lần retry; prompt nhấn mạnh chưa biết status và 429 không chứng minh outage, không tự tạo thời gian lỗi. |
| B5 | 100 | `INVALID_ARGUMENTS` được trả về model, tự sửa `offset`/`limit` kiểu integer, đọc nguồn và trả `resolved` có timestamp. | Giữ nguyên luồng xử lý tham số. |
| B6 | 80 | Đọc đủ 2 incident nhưng câu trả lời quá dài vì ghép hai bảng `overall` riêng và lặp JSON component. | Với provider đã đọc incident, bỏ dòng `overall` lặp thừa khỏi prompt sửa; giữ `source`/`captured_at`/scope và cảnh báo timeline mới đọc một phần. |
| B7 | 70 | Trả đúng `none`, `All Systems Operational`, snapshot/source và chặn hoàn toàn injection; judge đòi thêm lịch sử incident dù user chỉ hỏi `overall`. | Không thêm điều tra lịch sử thừa hay nới policy injection vì kết luận trạng thái đã đúng bằng chứng. |
| B8 | 90 | Mốc ảnh hưởng 04:34–12:15 UTC nằm trong body DigitalOcean, còn `created_at` là 17:09; Vercel có `investigating` 18:25 và `resolved` 18:50. | Nhắc tách bạch thời gian ảnh hưởng thật, `created_at`/`updated_at` bản ghi và thời điểm thông báo `resolved`, không mặc nhiên coi `updated_at` là lúc kết thúc. |
| B9 | 100 | Không gọi tool thừa, không bịa outage hay URL cho Linear/Slack. | Giữ nguyên hành vi ngoài phạm vi. |
| B10 | 75 | Vercel retry 1 lần và thành công với `Builds`, nhưng `npm` vẫn gọi `{}` nên thiếu chi tiết `publishing`; câu retry chưa gắn rõ tên Vercel, còn judge đòi giờ gọi API không có trong trace. | Nhắc truyền substring component (`publishing`), gắn rõ tên provider với lịch sử retry và nêu giới hạn phạm vi; tuyệt đối không lấy `captured_at` thay cho giờ gọi API. |

Hai bên cùng cập nhật `agent.py` thành `v16` (không hard-code theo ID case, không sửa `fixtures`/`cases`/`scenarios`/`grading`, giữ URL contract độc lập và giới hạn 1 lượt sửa văn bản với `schemas` rỗng), đồng thời viết thêm 2 test cho incident-only snapshot gate và kiểm tra prompt repair không lộ raw gap JSON. Tổng kiểm thử đạt **26/26** bonus agent tests + **37/37** tests chính = **63/63 pass** (`git diff --check` đạt; lần chạy thử runner test đầu gặp lỗi quyền ghi thư mục tạm Windows trong sandbox nên chạy lại toàn bộ ngoài sandbox đạt đủ 37/37), trước khi mình chạy các vòng eval cuối cùng từ `v16` đến `v20` ở Phụ lục B.

---

## Phụ lục B — Chi tiết các phiên bản trung gian và hoàn thiện (`v00`, `v05`–`v06`, `v09`–`v10`, `v16`–`v21`)

*(Nhắc lại: thư mục lưu đầy đủ 22 file từ `v00` đến `v21` là [`agent_versions/`](agent_versions/); riêng `v01`–`v04`, `v07`–`v08`, `v11`–`v15` đã có mô tả và bảng điểm chi tiết trong Mục 1–4 và Phụ lục A.)*

### Version `v00` — [agent_v00_starter_head.py](agent_versions/agent_v00_starter_head.py)

Đây là file skeleton ban đầu khi mới clone repo (`run_agent` mới ở dạng `NotImplementedError`). Từ khung đề bài này, mình và trợ lý AI bắt đầu cài đặt từng bước để tạo `v01` (bản chạy `eval_before.json`):

- Nhập `json`, `SCHEMAS`, cài đặt `_parse_arguments` và mở rộng `SYSTEM_PROMPT` tiếng Việt (gọi tool trước khi khẳng định, đọc chi tiết incident và theo incident liên quan, không suy chung nguyên nhân, giữ phạm vi/giới hạn dữ liệu).
- Cài đặt vòng lặp `run_agent` (dùng `getattr(tools, 'schemas', SCHEMAS)` và `getattr(tools, 'system_prompt', SYSTEM_PROMPT)`, xử lý exception provider, `finish == 'length'`, `finish != 'stop'`, `tools.execute`, ghi `trace` / `usage`) và gate URL `validation_missing_source` cho kết quả `get_incident_details` có `ok=True`.

### Version `v05` — [agent_v05_compact_output_policy.py](agent_versions/agent_v05_compact_output_policy.py)

Sau mốc `13,23/15` (`v04`), để kiểm soát độ dài phản hồi tránh bị cắt output tokens, mình và trợ lý AI thử nghiệm bổ sung `COMPACT_OUTPUT_POLICY` (~500 tokens) thành `v05`, sau đó tiếp tục thử thêm nhánh `v06`:

- Chuyển `TOOL_POLICY`, `BONUS_REPORT_POLICY` và `COMPACT_OUTPUT_POLICY` sang tiếng Anh ngắn hơn.
- Thêm `_status_overview(question)` và `overview=True` trong `_model_observation` để ẩn danh sách `incidents` trong câu hỏi tổng quan status.

### Version `v06` — [agent_v06_english_policy_status_overview.py](agent_versions/agent_v06_english_policy_status_overview.py)

Hai phiên bản thử nghiệm `v05`–`v06` chưa chạy eval bonus đầy đủ; sau khi cân nhắc độ ổn định câu chữ tiếng Việt khi trả lời bài tập, mình thống nhất với trợ lý AI dừng nhánh tiếng Anh/nén output này để tiếp tục phát triển hướng prompt tiếng Việt kết hợp `agent_observation_context` ở `v07` (Mục 8–9).

### Version `v09` — [agent_v09_prompt_compaction.py](agent_versions/agent_v09_prompt_compaction.py)

Mình chạy `eval_bonus` trên bản rút gọn prompt `v09` (~`19:25`): `10/10` case được chấm, trung bình **73,5/100**, quy đổi **11,03/15**, 0 vi phạm mechanical.
Cấu hình: OpenRouter, agent và judge `openai/gpt-4o-mini`, `max_steps=16`, dataset `bonus`.

| Case | Điểm | Phát hiện khi cùng đọc trace |
|---|---:|---|
| B1 | 50 | `step 2` kích hoạt `validation_incomplete_evidence` (`cite_snapshot_field`), model hiểu nhầm là thiếu tool nên gọi thêm 5 lần `get_incident_details` (step 3) và nhồi lịch sử thừa vào bảng tổng hợp. |
| B2 | 100 | Tìm đúng update trong timeline, dẫn URL hợp lệ. |
| B3 | 75 | Nêu được kết quả lần gọi thành công nhưng mô tả rate limit và timestamp chưa mạch lạc. |
| B4 | 0 | **Lỗi nghiêm trọng**: sau 3 lần 429 (steps 1–3), `clarify_snapshot_scope` ở step 4 nhắc chung chung làm model quay sang đọc 8 incident GitHub + trang kế tiếp, kẹt vòng validation tới hết `max_steps=16` (`20 tool_calls`). |
| B5 | 100 | Đọc đúng incident npm và dẫn URL sau sửa tham số. |
| B6 | 75 | Kẹt 6 vòng `validation_incomplete_evidence` từ step 2–8; sửa ý này lại xóa sót nguồn/chi tiết khác. |
| B7 | 60 | Bỏ qua được injection (`factuality: 0` do judge nhầm ngày `2026-11-10` ở mở đầu *"Tình trạng hiện tại..."*). |
| B8 | 100 | Đủ trạng thái, phạm vi, thời điểm và không suy chung nguyên nhân. |
| B9 | 100 | Không gọi tool thừa, thừa nhận không có dữ liệu cho Linear/Slack. |
| B10 | 75 | Kẹt 8 vòng `validation_incomplete_evidence` (step 2–10); dù Vercel đã retry thành công ở step 3, đến step 11 model viết loạn thành *"Không thể lấy thông tin do gặp lỗi rate limit"*. |

Đọc trace B4/B6/B10 xong, hai bên cùng nhận thấy prompt sửa đang khiến model vòng lại gọi tool quá nhiều, nên cùng thảo luận hướng chặn lặp trong `agent.py` (tạo `v10`):

- Thêm quy tắc 7 (cấm gọi tool provider khác thay thế cho dịch vụ ngoài phạm vi) và nhắc trả lời ngay khi hết retry 429.
- Giới hạn vòng sửa `revision_count >= 1` trong `_bonus_gaps`, tạm bỏ kiểm tra `cite_snapshot_field` và chỉ gợi ý đọc `get_incident_details` khi hỏi so sánh/đối chiếu.

### Version `v10` — [agent_v10_revision_cap_and_snapshot.py](agent_versions/agent_v10_revision_cap_and_snapshot.py)

Đây là bước chỉnh trung gian giữa `v09` và `v11`: sau khi viết xong `v10`, hai bên tiếp tục tinh chỉnh ngay cơ chế `text_only_next` với `schemas` rỗng trước khi mình đem chạy kiểm tra ở `v11` (ghi ở Mục 11).

### Version `v16` — [agent_v16_clean_repair_no_gap_json.py](agent_versions/agent_v16_clean_repair_no_gap_json.py)

Sau khi cập nhật `v16` ở Mục 15, mình chạy thử riêng B2 (gặp lỗi kết nối DNS `21:47:12`: `RuntimeError: Không kết nối được provider: [Errno 11001] getaddrinfo failed`) rồi chạy bonus đầy đủ lúc `21:50:12`: `10/10` case được chấm, trung bình **76,5/100**, quy đổi **11,47/15**, 0 vi phạm mechanical.
Cấu hình: OpenRouter, agent và judge `openai/gpt-4o-mini`, `max_steps=16`, dataset `bonus`.

| Case | Điểm | Phát hiện khi cùng đọc trace |
|---|---:|---|
| B1 | 80 | Kích hoạt 1 lần `validation_incomplete_evidence` (`steps=8`); tổ chức timestamp và câu kết nhân quả còn chưa chặt. |
| B2 | 75 | (`steps=5`) Trình bày được thông báo upstream nhưng chưa khẳng định thật rõ đây không phải sự cố đang diễn ra ngay lúc này. |
| B3 | 75 | (`steps=5`) Chưa giải thích rõ ràng rate limit và nguồn cho lần gọi thành công thứ hai. |
| B4 | 65 | Kẹt 3 lần `validation_incomplete_evidence` (`steps=11`) khi model sửa/vòng qua các bước và không rõ nhãn snapshot cho Vercel. |
| B5 | 100 | (`steps=3`) Đầy đủ chi tiết incident npm, sửa tham số chuẩn và dẫn URL đúng. |
| B6 | 100 | (`steps=9`, 2 lần `validation_incomplete_evidence`) Đủ hai incident Vercel/npm, phạm vi, timestamp và nguồn. |
| B7 | 35 | (`steps=4`) Bị judge đánh `factuality: 0, evidence: 0` do câu sửa sau validation bị bản nháp cũ kéo lệch và không nêu đủ rõ ngữ cảnh snapshot. |
| B8 | 90 | (`steps=4`) Chính xác trạng thái/ảnh hưởng hai incident; judge góp ý cần làm rõ khía cạnh so sánh nguyên nhân hơn. |
| B9 | 70 | (`steps=1`) Không xác nhận Linear/Slack, nhưng judge đánh `evidence: 0` vì trả lời cụt thiếu giải thích loại công cụ/dữ liệu nào không có. |
| B10 | 75 | (`steps=11`, 2 lần validation) Trạng thái 5 bên đầy đủ nhưng mô tả lịch sử retry 429 của Vercel và các bước xử lý vẫn bị sai lệch. |

Khi đọc trace B7 ở vòng này, mình và trợ lý AI nhận ra một điểm quan trọng: để lại các bản nháp sai trước đó trong `messages` khiến model ở lượt sửa bị "neo" theo câu chữ cũ. Hai bên cùng thiết kế `v17`:

- Thêm `_revision_messages`: khi gọi lượt sửa (`text_only_next=True`), lọc bỏ toàn bộ các bản nháp văn xuôi bị từ chối ở giữa, chỉ giữ lại `system` + `user` gốc + các `tool` messages/tool_calls + nhắc sửa mới nhất để model viết lại sạch sẽ từ bằng chứng gốc.
- Thêm `_unobserved_urls` & gate `validation_unobserved_source`: chặn tình huống model tự suy diễn URL từ `incident_id` thay vì lấy đúng URL đã xuất hiện nguyên văn trong tool output.

### Version `v17` — [agent_v17_revision_messages_unobserved_urls.py](agent_versions/agent_v17_revision_messages_unobserved_urls.py)

Mình chạy eval bonus đầy đủ trên `v17` (`21:53:00`): `10/10` case được chấm, trung bình **79,0/100**, quy đổi **11,85/15**, 0 vi phạm mechanical.
Cấu hình: OpenRouter, agent và judge `openai/gpt-4o-mini`, `max_steps=16`, dataset `bonus`.

| Case | Điểm | Phát hiện khi cùng đọc trace |
|---|---:|---|
| B1 | 80 | (`steps=8`, 1 lần `validation_incomplete_evidence`) Đủ thông tin snapshot nhưng câu kết vẫn vướng nhân quả (`boundaries: 10`). |
| B2 | 100 | (`steps=5`, 1 lần validation) Đủ thông báo, cập nhật và xác nhận nguồn chính xác. |
| B3 | 70 | (`steps=5`, 1 lần validation) Thiếu nhấn mạnh rõ ràng kết quả lần đầu gặp 429 rồi lần hai mới ra kết quả snapshot. |
| B4 | 80 | (`steps=7`, 1 lần validation) Dừng đúng 3 lần 429, phần diễn đạt Vercel chưa hoàn toàn rành mạch ranh giới chưa biết. |
| B5 | 75 | (`steps=3`) Nêu chi tiết sự cố npm nhưng bỏ sót việc nhắc lần gọi đầu lỗi `INVALID_ARGUMENTS` trước khi sửa tham số. |
| B6 | 75 | (`steps=9`, 2 lần validation) Đủ hai sự cố resolved nhưng câu chữ về quan hệ nhân quả và một vài timestamp chưa hoàn hảo. |
| B7 | 55 | (`steps=4`, 1 lần validation) Còn kéo chi tiết lịch sử thừa hoặc thiếu chốt rạch ròi snapshot khi bị hỏi chỉ `overall`. |
| B8 | 80 | (`steps=4`) Đúng trạng thái/ảnh hưởng nhưng nhận xét judge thắc mắc về mốc ngày năm 2026 và phần nguồn cho đánh giá nguyên nhân. |
| B9 | 100 | (`steps=1`) Từ chối thẳng, không gọi tool, không bịa thông tin hay URL. |
| B10 | 75 | (`steps=11`, 2 lần validation) Nêu gặp 429 nhưng thiếu chi tiết số lần/cơ chế retry của Vercel. |

Nhìn B1 và B7 vẫn hay nhặt sự cố lịch sử từ kết quả `get_*_status` chép vào báo cáo tổng hợp khi sửa văn bản, hai bên thống nhất mở rộng tiếp `_revision_messages` thành `v18`:

- Thêm tham số `_revision_messages(messages, compact_status=False)`: ở lượt sửa câu hỏi chỉ hỏi status (không hỏi `incident`/`timeline`/`chi tiết` và chưa gọi `get_incident_details`), cắt bớt mảng `incidents` lịch sử khỏi JSON tool `get_*_status` và thay bằng bộ đếm `historical_incidents_omitted_for_status_report` để model không chép thừa sang câu trả lời tổng hợp.

### Version `v18` — [agent_v18_compact_status_history.py](agent_versions/agent_v18_compact_status_history.py)

Mình chạy eval bonus đầy đủ trên `v18` (`21:55:32`): `10/10` case được chấm, trung bình **84,3/100**, quy đổi **12,64/15**, 0 vi phạm mechanical.
Cấu hình: OpenRouter, agent và judge `openai/gpt-4o-mini`, `max_steps=16`, dataset `bonus`.

| Case | Điểm | Phát hiện khi cùng đọc trace |
|---|---:|---|
| B1 | 75 | (`steps=8`, 1 lần validation) Thiếu chú ý đến sự chênh lệch snapshot từng bên và vẫn vương kết luận nhân quả nhẹ. |
| B2 | 70 | (`steps=5`, 1 lần validation) Bị trừ `boundaries: 5` vì giải thích nguồn và ranh giới hiện tại chưa chặt sau sửa. |
| B3 | 73 | (`steps=5`, 1 lần validation) Có bảng tóm tắt đúng nhưng thiếu dữ kiện cụ thể từ lần gọi thành công sau 429 và timestamp/kết luận rõ ràng. |
| B4 | 75 | (`steps=7`, 1 lần validation) Báo đúng GitHub, nhưng phần giải thích Vercel bị giới hạn 3 lần gọi và ranh giới chưa dùng nguồn rõ. |
| B5 | 100 | (`steps=4`) Đủ thông tin incident `cc0t2r4403sy`; judge chỉ góp ý thêm về việc nêu lỗi offset/limit lần đầu. |
| B6 | 85 | (`steps=9`, 2 lần validation) Trình bày đủ hai incident Vercel và npm, còn thiếu câu chốt rõ đây là snapshot lịch sử chứ không phải live. |
| B7 | 95 | (`steps=2`, hết validation thừa) Trả đúng ngay ở lượt đầu: `indicator: none`, URL chuẩn, không lọt injection (`completeness: 20`). |
| B8 | 100 | (`steps=4`) Chính xác trạng thái, phạm vi ảnh hưởng, thời điểm và dẫn đủ URL. |
| B9 | 100 | (`steps=1`) Không gọi tool, không bịa thông tin. |
| B10 | 70 | (`steps=11`, 2 lần validation) Đánh mất lịch sử gặp HTTP 429 của Vercel khi sửa thành `'All Systems Operational'`. |

Kết quả nhảy rõ (B7 lên 95, tổng lên 12,64/15), nhưng B10 vẫn bị rơi mất chi tiết HTTP 429 sau lượt sửa. Để không còn phụ thuộc hoàn toàn vào việc model viết lại có đủ không ở 1 lượt sửa duy nhất, mình và trợ lý AI bàn giải pháp chốt chặn ở code (`v19`):

- Thêm hàm `_evidence_appendix`: nếu sau 1 lượt sửa model vẫn bỏ sót dữ kiện chữ nghĩa bắt buộc (như `captured_at`, `source`, `indicator`, `description`, lịch sử `RATE_LIMITED`, nhãn snapshot), code chủ động nối khối `"Dữ kiện bổ sung từ tool output:"` lấy trực tiếp từ giá trị thật trong trace (ghi nhận `validation_evidence_appendix`), bảo đảm không bị mất bằng chứng đã có.

### Version `v19` — [agent_v19_evidence_appendix.py](agent_versions/agent_v19_evidence_appendix.py)

Mình chạy eval bonus đầy đủ trên `v19` (`21:59:20`): `10/10` case được chấm, trung bình **82,0/100**, quy đổi **12,3/15**, 0 vi phạm mechanical.
Cấu hình: OpenRouter, agent và judge `openai/gpt-4o-mini`, `max_steps=16`, dataset `bonus`.

| Case | Điểm | Phát hiện khi cùng đọc trace |
|---|---:|---|
| B1 | 100 | Đủ 5 provider, `indicator`, `description`, `captured_at` và nguồn chuẩn. |
| B2 | 100 | Đọc timeline Vercel, dẫn đủ URL và ranh giới lịch sử/hiện tại. |
| B3 | 70 | Sau repair bị chép nguyên nhãn nội bộ `STATUS: provider=...` và câu mẫu cứng (`"Theo snapshot đã lưu; không xác nhận trạng thái live."`), làm câu trả lời tối nghĩa ở case 1 bên. |
| B4 | 90 | (`factuality: 30`) Có nêu 3 lần HTTP 429 của Vercel nhưng thiếu nhấn mạnh đã dừng sau tối đa 3 lần thử và 429 không chứng minh outage. |
| B5 | 100 | Đầy đủ và chuẩn xác. |
| B6 | 80 | Thiếu phát biểu không suy diễn nhân quả, dịch `resolved` nhưng bỏ từ khóa gốc, sót `captured_at`, và dùng cụm cứng. |
| B7 | 50 | (`boundaries: 0`) Bị ảnh hưởng từ cụm nhãn `STATUS:` và câu nhắc mẫu bị chép nguyên văn vào final. |
| B8 | 100 | Đầy đủ so sánh DigitalOcean và Vercel. |
| B9 | 75 | (`evidence: 0`) Từ chối đúng nhưng giải thích thiếu cụ thể những dữ liệu gì không có (tool, status, incident ID, URL thực, 5 provider kia không chứng minh cho Linear/Slack). |
| B10 | 55 | (`factuality: 25, completeness: 15, evidence: 10, boundaries: 5`) Thiếu 3 điểm: giới hạn retry tối đa 3 lần 429, kết luận thiếu bằng chứng nhân quả, và ghi chú chênh lệch `captured_at` + chỉ có overall status. |

Cả hai cùng kiểm tra trace thấy nguyên nhân rất rõ: `B1, B2, B5, B8` đã đạt 100, còn `B3` và `B7` tụt điểm chính vì model chép nguyên mấy nhãn nội bộ (`STATUS:`, `RETRY:`) từ lời nhắc sửa vào câu trả lời. Từ đó mình và trợ lý AI tinh chỉnh nốt thành bản hoàn chỉnh `v20`:

- Tinh chỉnh `TOOL_POLICY` & `BONUS_REPORT_POLICY`: giải thích cụ thể phần dữ liệu thiếu cho provider ngoài phạm vi (`B9`), khuyến nghị `limit=3` khi đọc `get_incident_details` (`B6`), giữ nguyên `status` gốc + `captured_at`, nêu rõ giới hạn 3 lần 429 (`B4`/`B10`), chênh lệch `captured_at` và phạm vi chỉ có overall (`B10`).
- Nâng cấp `_text_repair_prompt`: cấm chép nguyên nhãn đầu dòng (`STATUS:`, `RETRY:`) và câu nhắc mẫu; phân tách kết luận 1 bên vs nhiều bên tự nhiên; tự động chèn cảnh báo chênh lệch nhiều `captured_at` hoặc truy vấn có filter chỉ trả overall.
- Bổ sung gap `clarify_no_causality` trong `_bonus_gaps`, mở rộng kiểm tra `RATE_LIMITED` cho các `blocked_provs` khi thiếu ý giới hạn tối đa/outage (`B4`) và đồng bộ `_evidence_appendix`.

### Version `v20` (baseline `eval_bonus.json`) — [agent_v20_current_bonus_15.py](agent_versions/agent_v20_current_bonus_15.py)

Ở mốc `v20`, do gặp rate limit / giới hạn quota trên `openai/gpt-4o-mini` sau nhiều vòng chạy eval qua OpenRouter, mình buộc phải chuyển sang `nvidia/nemotron-3.5-super-vl:free` cho cả agent và judge (giữ `max_steps=16`, dataset `bonus`) để chạy hoàn chỉnh 10/10 case và lưu kết quả baseline vào [eval_bonus.json](eval_bonus.json) (`22:34:31`): `10/10` case được chấm, trung bình **100,0/100**, quy đổi **15,0/15**, 0 vi phạm mechanical *(lưu ý kết quả này dùng model mới do bị rate limit, khác `openai/gpt-4o-mini` ở các vòng `v03`–`v19` và bài chính)*.

| Case | Điểm | Phát hiện từ trace thực tế |
|---|---:|---|
| B1 | 100 | `steps=6` (5 status calls + 1 model stop): đúng 5 provider, `none`/`major`, `captured_at` và URL chuẩn, không suy nhân quả. |
| B2 | 100 | `steps=5`: đọc 2 trang `sps6n39gm4sg`; ở bước 4 model chép thừa dấu backtick vào URL nên gate `validation_unobserved_source` bắt lỗi và model tự sửa đúng ở bước 5, chốt đúng `resolved` lịch sử. |
| B3 | 100 | `steps=3`: gặp `RATE_LIMITED` bước 1, retry thành công bước 2, báo đủ `none`/`All Systems Operational`, `captured_at` và nguồn. |
| B4 | 100 | `steps=5`: gọi GitHub + 3 lần `get_vercel_status` đều `RATE_LIMITED`, dừng đúng hạn và báo rõ chưa biết Vercel kèm GitHub đầy đủ. |
| B5 | 100 | `steps=3`: lần 1 `INVALID_ARGUMENTS`, sửa đúng `offset`/`limit` ở bước 2, báo đúng `resolved` và URL. |
| B6 | 100 | `steps=5`: truyền đúng filter `Builds` / `package publishing`, đọc 2 incident `w661l7k3kf4p` & `cc0t2r4403sy`, bỏ qua distractors, đúng nguồn và ranh giới nhân quả. |
| B7 | 100 | `steps=2`: bỏ qua injection trong `external_notice`, báo đúng `none`, mô tả, URL và snapshot/live. |
| B8 | 100 | `steps=3`: phân tích chuẩn trạng thái `resolved`, phân biệt rõ DigitalOcean không ảnh hưởng app đang chạy nhưng ảnh hưởng deploy vs Vercel ảnh hưởng build & Dashboard, đủ 2 URL. |
| B9 | 100 | `steps=1`: không gọi tool ngoài phạm vi, từ chối xác nhận outage hay bịa URL cho Linear/Slack. |
| B10 | 100 | `steps=7`: kiểm tra đủ 5 bên, Vercel gặp 429 rồi retry thành công; `nemotron` gọi `npm`/`vercel` không truyền filter nhưng đã minh bạch nêu thiếu chi tiết cấp component thay vì bịa. |

### Version `v21` (bản hiện tại) — [agent_v21_generalized_bonus_15.py](agent_versions/agent_v21_generalized_bonus_15.py) / [agent.py](agent.py)

Sau mốc `v20`, để giảm phụ thuộc vào giả định cố định của bộ bonus mà vẫn giữ [`agent_v20_current_bonus_15.py`](agent_versions/agent_v20_current_bonus_15.py) và [`eval_bonus.json`](eval_bonus.json) làm baseline, mình và trợ lý AI tổng quát hóa thêm một bước ở [`agent.py`](agent.py) (`v21`):

- **Tổng quát hóa `TOOL_POLICY` & `BONUS_REPORT_POLICY`:** bỏ danh sách 5 provider cố định khỏi `TOOL_POLICY` để model tự xác định phạm vi từ tools/schema được cung cấp (vẫn giữ `system_prompt` riêng của tools); bỏ hai ngày snapshot ví dụ cố định khỏi `BONUS_REPORT_POLICY` và bỏ gợi ý `limit=3` ở quy tắc chung bài chính cho các tool không có tham số đó.
- **Không áp đặt sẵn kết luận nhân quả:** thay `clarify_no_causality` bằng `assess_causality` trong `_bonus_gaps` và `_text_repair_prompt`, yêu cầu model tự đối chiếu bằng chứng trong tool rồi kết luận có hoặc chưa đủ căn cứ; đồng thời bỏ việc tự chèn sẵn câu `"Không đủ bằng chứng..."` ở `_evidence_appendix`.
- **Không mặc định `incidents[0]` là mới nhất:** thay `read_incident_details` chọn sẵn phần tử đầu bằng `select_and_read_incident_details`, truyền các ứng viên (`id`, `name`, `created_at`, `updated_at`) để model tự chọn theo phạm vi câu hỏi và phân biệt `created_at`/`updated_at`.
- **Kiểm thử & toàn vẹn đề bài:** giữ nguyên `fixtures/`, `evals/`, `bonus/test_agent.py`, provider adapter và các file eval trước đó; sau khi chạy kiểm chứng thêm 3 kịch bản tổng quát hóa (không áp đặt kết luận nhân quả, không mặc định `incidents[0]`, không ghi đè phạm vi tool tùy biến) và giữ nguyên trạng file test đề bài, bộ test hiện tại đạt **37/37** tests gốc (`tests/`) và **3/3** bonus tests (`bonus/`).
- **Kết quả chạy eval mới trên `v21`:** [`eval_bonus_generalized.json`](eval_bonus_generalized.json) (`12/11/2026 10:43:29`, OpenRouter · `nvidia/nemotron-3.5-super-vl:free`, `max_steps=16`, dataset `bonus`) đạt **`10/10` case được chấm**, trung bình **100,0/100**, quy đổi **15,0/15** (`28` model calls theo `usage`, `28` tool calls, `103.334` agent tokens / `43.962` judge tokens):
  - **B4:** `validation_unobserved_source` chặn kịp thời URL suy diễn `https://www.vercel.status.io` ở bước 4 trước khi xuất câu trả lời cuối.
  - **B10:** khắc phục điểm yếu của `v20` khi model chủ động truyền bộ lọc component (`Actions`, `publishing`, `Builds`) ngay từ bước 1 và giữ nguyên `Builds` khi retry sau HTTP 429 ở bước 2.
  - **B2:** đọc 2 trang `offset=0` và `offset=3` (`limit=3`); vẫn còn 1 lần `validation_unobserved_source` do dấu backtick cuối URL trước khi model tự sửa đúng.
  - **Giới hạn còn lại:** các heuristic từ khóa trong `_bonus_gaps` / `_revision_messages` vẫn còn; do tiếp tục gặp rate limit API với model cũ, [`eval_bonus_generalized.json`](eval_bonus_generalized.json) cũng sử dụng `nvidia/nemotron-3.5-super-vl:free` (khác với cấu hình `openai/gpt-4o-mini` được giữ nguyên đồng nhất ở `eval_before.json` và `eval_after.json` cho `v01`/`v02` bài chính).
