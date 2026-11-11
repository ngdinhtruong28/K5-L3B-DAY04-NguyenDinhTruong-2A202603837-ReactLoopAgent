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
| Điểm tổng (22 case) | 93.95/100 | 95.27/100 |
| Điểm case H2 | 50/100 | 100/100 |
| Số model calls của H2 | 2 | 5 |
| Token của H2 (agent + judge) | 2,426 | 7,035 |


- Lỗi đã giải quyết: H2 trước đây chỉ tra `get_incident_details`, thiếu `get_status` để xác nhận trạng thái hiện tại. Sau thay đổi, trace có `validation_missing_status`, agent gọi `get_status` ở model step 3 và trả lời dựa trên cả incident đã resolved lẫn trạng thái Actions operational. Điểm H2 tăng từ 50 lên 100.
- Lỗi còn lại hoặc mới xuất hiện: H7 vẫn kết luận Cloudflare Spectrum operational dù tool trả `TimeoutError`; điểm H7 giảm từ 75 xuống 55. Đây là lỗi cần cải tiến riêng về việc không suy đoán khi tool lỗi. H1 vẫn bị giới hạn ở 60 vì vi phạm yêu cầu gọi tools trong ít nhất hai model steps khác nhau.
- Điểm tổng tăng 1.32 điểm. Các case khác cũng dao động nhẹ giữa hai lần chấm; không thể quy toàn bộ dao động đó cho thay đổi H2.

## 5. Giới hạn

- LLM judge chấm theo expected, câu trả lời và trace nên có thể không nhất quán giữa các lần chạy; điểm là tham khảo, cần đọc lý do chấm và kiểm tra bằng chứng. Trong file, agent và judge cùng dùng `gpt-4.1`, nên có thể có thiên lệch của cùng model.
- Snapshot cố định phản ánh dữ liệu tại thời điểm thu thập, không đại diện cho trạng thái dịch vụ hiện tại. Tools cũng chỉ tìm trong phạm vi dữ liệu mà chúng trả về (ví dụ tối đa 50 incident gần nhất); không tìm thấy không chứng minh incident chưa từng xảy ra.
- Mỗi phiên bản chỉ chạy eval một lần, vì vậy chưa đo được độ dao động giữa nhiều lần chạy. Thay đổi câu trả lời có thể làm điểm judge thay đổi dù cấu hình và dataset giống nhau.
- H2 dùng 2 model calls và 1,112 token agent trước cải tiến; sau cải tiến dùng 5 model calls và 5,192 token agent. Judge dùng lần lượt 1,314 và 1,843 token. Tổng token H2 được ghi trong bảng là agent cộng judge. File eval không có trường chi phí nên không thể kết luận chi phí tiền tệ.
