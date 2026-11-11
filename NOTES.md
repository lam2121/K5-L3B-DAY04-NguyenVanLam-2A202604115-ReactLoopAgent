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
