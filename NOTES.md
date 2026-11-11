# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Thành Tùng
MSSV: 2A202603957

## 1. Case được chọn

- **ID case:** E6 — "GitHub Actions hiện có hoạt động bình thường không?" (Scenario: fail = True, giả lập lỗi timeout công cụ).
- **Câu trả lời của agent ở lần 1:** Agent trả lời cụt lủn `"No response requested."` ngay ở step 1 mà không thực hiện bất kỳ lệnh gọi công cụ nào.
- **Điểm ở lần chạy 1:** 15/100 (Factuality: 10, Completeness: 0, Evidence: 0, Boundaries: 5).

## 2. Bằng chứng trong trace

Trace ở lần chạy 1 thể hiện model dừng ngay lập tức mà không gọi tool:

```text
Trace lần chạy 1 (eval_before.json):
[
  {
    "event": "model",
    "step": 1,
    "finish": "stop",
    "has_calls": false,
    "content_preview": "No response requested."
  }
]
```

**Nguyên nhân xảy ra lỗi:**
- Ở lần chạy 1, prompt chưa đủ tính chỉ thị ép buộc agent phải luôn ưu tiên tra cứu trạng thái qua công cụ trước khi trả lời. Model bị hallucinate hoặc hiểu nhầm ngữ cảnh nên kết thúc ngay với nội dung vô nghĩa `"No response requested."`.
- Hậu quả: Không có dữ liệu, không phát hiện được lỗi timeout giả lập của kịch bản kiểm thử, vi phạm yêu cầu "Dùng tools trước khi khẳng định về trạng thái".

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  1. Cập nhật `SYSTEM_PROMPT` trong `agent.py`: Nhấn mạnh rõ quy tắc bắt buộc dùng công cụ trước, phân biệt rõ dữ liệu snapshot/lịch sử/hiện tại, và luôn ghi nhận bằng chứng hoặc nêu rõ giới hạn khi không lấy được dữ liệu.
  2. Bổ sung cơ chế xử lý lỗi và retry tự động (exponential backoff) trong `providers.py` và hoàn thiện vòng lặp xử lý quan sát lỗi từ công cụ trong ReAct loop của `agent.py`.
- **Vì sao chọn thay đổi này:**
  - Định hướng lại hành vi của model: khi gặp câu hỏi về trạng thái, bắt buộc phải phát sinh `tool_calls`.
  - Khi công cụ trả về lỗi (như `TimeoutError`), agent nhận được kết quả đó dưới dạng quan sát (observation) ở step tiếp theo và biết cách xử lý: báo cho người dùng biết không có dữ liệu do lỗi mạng/timeout thay vì đoán mò hoặc dừng bất thường.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 91.68 / 100 | **94.18 / 100** |
| Điểm case đã chọn (E6) | **15 / 100** | **96 / 100** |
| Số model calls (E6) | 1 step (0 tool calls) | **3 steps (2 tool calls)** |
| Token tổng (E6) | 366 tokens | **1,869 tokens** |

- **Lỗi đã giải quyết:**
  - Ở lần chạy 2, agent đã gọi đúng công cụ `get_status` 2 lần. Cả hai lần đều nhận phản hồi lỗi giả lập `"Không thể lấy dữ liệu status (timeout giả lập)."`.
  - Agent không bịa đặt trạng thái, giải thích trung thực rằng công cụ gặp timeout nên chưa thể khẳng định GitHub Actions bình thường hay có sự cố, đồng thời hướng dẫn người dùng kiểm tra trực tiếp qua githubstatus.com.
  - Điểm của case E6 tăng ngoạn mục từ **15 lên 96 điểm** (+81 điểm).
  - Điểm trung bình toàn bộ 22 case tăng từ 91.68 lên 94.18.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Ở case E6, judge nhận xét trừ 4 điểm nhỏ vì câu trả lời có phỏng đoán tên tham số bộ lọc cụ thể (như `"Actions"` và `"GitHub Actions"`) dù trong trace ngắn gọn không hiển thị rõ chuỗi tham số này.
  - Một số case nhiều bước suy luận (như A5, A6) có sự dao động điểm nhẹ do tính ngẫu nhiên của LLM judge khi ngữ cảnh dài.

## 5. Giới hạn

- **Giới hạn của điểm judge:** LLM judge dùng cùng model với agent (`baidu/ernie-5.5-flash-preview:free`), dễ bị thiên lệch tự đánh giá tốt. Điểm chỉ phản ánh xấp xỉ chất lượng; hai lần chạy với cùng prompt có thể cho điểm khác nhau do tính ngẫu nhiên.
- **Giới hạn của snapshot:** Dữ liệu status page là snapshot tĩnh tại một thời điểm (`captured_at = 2026-11-09T03:50:47Z`), không phản ánh thực tế hiện tại. Chỉ tìm trong 50 incident gần nhất; "không thấy" không chứng minh không có sự cố.
- **Giới hạn của một lần chạy mỗi phiên bản:** Một lần chạy không đủ để xác nhận độ ổn định; điểm có thể dao động ±3–5 điểm giữa các lần chạy kể cả khi agent không thay đổi. Kết luận cải tiến chỉ mang tính tham khảo.
