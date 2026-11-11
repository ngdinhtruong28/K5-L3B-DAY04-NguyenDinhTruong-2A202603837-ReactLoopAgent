# Ghi chú phân tích · Day 04

Họ và tên: Nguyễn Đình Trường
MSSV: 2A202603837

## 1. Case được chọn

- ID case: **H2**
- Câu trả lời của agent: Incident GitHub `t89g2q1800fr` đã `resolved` trong snapshot và không thể dùng incident lịch sử đó để khẳng định GitHub Actions đang outage. Agent dẫn URL `https://stspg.io/lv4q3xm9r0rk`.
- Điểm ở lần chạy 1: **50/100** (điểm judge thô 91/100; điểm chính thức bị giới hạn bởi kiểm tra cơ học).

## 2. Bằng chứng trong trace

Trace chỉ có tool đọc chi tiết incident, sau đó model kết thúc mà không gọi `get_status` để kiểm tra trạng thái hiện tại:

```text
model step=1 finish=tool_calls tool_names=["get_incident_details"]
tool name=get_incident_details
     arguments={"provider":"github","incident_id":"t89g2q1800fr"}
     result.status="resolved"
     result.components=["Actions"]
     result.url="https://stspg.io/lv4q3xm9r0rk"
model step=2 finish=stop tool_names=[]
```

Nguyên nhân có thể xảy ra: agent xem trạng thái `resolved` của incident như đủ để trả lời câu hỏi về tình trạng hiện tại của component. Judge ghi nhận thiếu `get_status`; vì vậy điểm bị chặn ở 50.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong `agent.py`, nhận diện một số cách hỏi về trạng thái hiện tại; theo dõi provider đã tra incident và provider đã gọi status tool. Nếu model định kết thúc khi còn provider chưa được status lookup, ghi event `validation_missing_status` vào trace và yêu cầu model gọi status tool trước khi trả lời.
- Vì sao chọn thay đổi này: Thay đổi xử lý đúng thiếu sót thể hiện trong trace H2, đồng thời không ép gọi status tool khi câu hỏi chỉ hỏi trạng thái của một incident lịch sử.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng |  |  |
| Điểm case đã chọn |  |  |
| Số model calls |  |  |
| Token, chi phí nếu có |  |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
