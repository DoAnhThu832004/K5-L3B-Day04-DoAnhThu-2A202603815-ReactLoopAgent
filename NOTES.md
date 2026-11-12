# Ghi chú phân tích · Day 04

Họ và tên: Đỗ Anh Thư
MSSV: 2A202603815

## 1. Case được chọn

- ID case: A5
- Câu hỏi của case: "Tôi cần danh sách đầy đủ tất cả incident GitHub Actions trong năm 2025. Hãy tìm dữ liệu và nói rõ độ đầy đủ của kết quả, đừng bịa."
- Câu trả lời của agent ở lần 1:
  Agent mở đầu bằng: *"Hiện tại không có incident nào liên quan đến GitHub Actions trong năm 2025. Trong danh sách gần đây, có một số incident đã xảy ra, nhưng chúng đều diễn ra vào năm 2026..."* Sau đó liệt kê thông tin 5 sự cố thuộc tháng 11/2026 và kết luận rằng *"kết quả có độ đầy đủ trong phạm vi tìm kiếm gần nhất nhưng không thể đưa ra dữ liệu cho năm 2025, vì không có sự cố được ghi nhận vào thời gian đó"*.
- Điểm lần 1: 20 / 100 (Factuality: 0/40, Completeness: 5/25, Evidence: 0/20, Boundaries: 15/15).
- Nhận xét của Judge: Câu trả lời bị sai sự thật khi khẳng định không có bất kỳ incident nào trong năm 2025, đồng thời lại đưa các sự cố của năm 2026 vào câu trả lời cho năm 2025 mà không có căn cứ.

## 2. Bằng chứng trong trace

Mở file `eval_before.json` xem phần trace của case A5, chuỗi gọi tool diễn ra như sau:
1. `Step 1`: Gọi `get_status(provider='github', component='Actions')`. Kết quả trả về `overall: none`, kèm ghi chú công cụ chỉ tìm trong 50 incident gần nhất và *"không tìm thấy không chứng minh không có sự cố"*.
2. `Step 2`: Agent gọi liên tục 5 tool calls `get_incident_details` cho các incident ID gồm `t89g2q1800fr`, `h02n9572mx9y`, `xr3myzj7ms51`, `smtdswd1h2w4`, `pfbmpxxfq1cy`. Tất cả các sự cố này đều diễn ra vào tháng 11/2026.
3. Cuối cùng, agent tự kết luận là không tìm thấy sự cố 2025 nên coi như năm 2025 không có sự cố nào.

Nguyên nhân gốc rễ:
- Model bị nhầm lẫn giữa việc "công cụ không có dữ liệu" với "thực tế không có sự cố". Snapshot bài lab chỉ lưu 50 incident gần nhất (thời điểm tháng 11/2026), hoàn toàn không có dữ liệu của năm 2025.
- Model chưa kiểm soát tốt phạm vi câu hỏi: Người dùng hỏi năm 2025 nhưng agent lại đi gọi chi tiết các sự cố năm 2026 rồi đem vào câu trả lời, vừa tốn số lượt gọi tool vừa gây sai lệch thông tin.

## 3. Thay đổi đã thực hiện

Mình đã thực hiện hai thay đổi chính trong `agent.py`:
1. Bổ sung hướng dẫn vào `SYSTEM_PROMPT` (ở Quy tắc 2): Nhắc model rằng snapshot chỉ chứa tối đa 50 incident gần nhất tại tháng 11/2026. Khi gặp câu hỏi yêu cầu danh sách đầy đủ của năm 2025 hoặc giai đoạn dài (2020–2026), model phải nói rõ ngay từ đầu là công cụ không có dữ liệu lịch sử này nên không thể cung cấp danh sách đầy đủ, việc không tìm thấy không chứng minh là năm 2025 không xảy ra sự cố, và tuyệt đối không liệt kê các sự cố 2026 ra thay thế.
2. Thêm logic kiểm tra nguồn (Evidence Gate) theo đúng yêu cầu Bài 4: Trích xuất và lưu lại URL của các sự cố đã đọc thành công qua `get_incident_details`. Nếu câu trả lời cuối cùng chưa có URL nguồn, agent sẽ ghi nhận event `validation_missing_source` vào trace và nhắc model bổ sung link nguồn trong giới hạn số bước cho phép.

Lý do chọn cách làm này:
Cách tiếp cận này xử lý trực diện lỗi suy diễn sai ở case A5 (biết từ chối đúng mực khi thiếu dữ liệu thay vì nói bừa), đồng thời đảm bảo thỏa mãn bài test kiểm tra nguồn của Bài 4.

## 4. So sánh hai lần chạy

Dưới đây là các số liệu rút ra từ `eval_before.json` và `eval_after.json` (cả hai lần đều chạy chung cấu hình: model `openai/gpt-4o-mini`, judge `openai/gpt-4o-mini`, max_steps 8, dataset core):

| Chỉ số | Lần 1 (`eval_before.json`) | Lần 2 (`eval_after.json`) | Thay đổi |
|---|---|---|---|
| Điểm trung bình cả bộ (22 case) | 77.82 / 100 | 87.05 / 100 | Tăng 9.23 điểm |
| Điểm case đã chọn (A5) | 20 / 100 | 95 / 100 | Tăng 75 điểm |
| Tổng số lượt gọi model của Agent | 51 calls | 49 calls | Giảm 2 calls |
| Token của Agent (prompt + completion) | 44,477 tokens | 103,399 tokens | Tăng ~58k tokens |
| Tổng chi phí (Agent + Judge) | ~$0.0187 | ~$0.0234 | Tăng ~$0.0047 |

Đánh giá cụ thể:
- Những điểm cải thiện: Case A5 tăng từ 20 lên 95 điểm. Model không còn gọi tool thừa (0 tool calls), nói thẳng giới hạn của snapshot 50 incident và từ chối khẳng định năm 2025 không có lỗi. Nhiều case khác cũng tăng điểm tốt nhờ prompt rõ ràng hơn: H5 (35 -> 75), E1 (70 -> 100), E6 (70 -> 100), A3 (75 -> 100).
- Những điểm giảm hoặc phát sinh: Một vài case bị giảm điểm nhẹ như E2 (100 -> 75), E4 (100 -> 75), H8 (75 -> 55). Nguyên nhân là do khi thêm các quy tắc kiểm soát biên chặt hơn, câu trả lời của model ở một số câu đơn giản trở nên thận trọng và ngắn gọn hơn, khiến LLM Judge chấm điểm độ đầy đủ (completeness) thấp đi một chút. Ngoài ra, prompt dài hơn cũng làm tăng số lượng token đầu vào.

## 5. Giới hạn

Qua hai lần chạy eval, mình nhận thấy một số giới hạn thực tế:
1. Độ lệch của LLM Judge: Điểm số do LLM chấm vẫn có độ dao động nhất định giữa các lần chạy do tính chất sinh từ ngẫu nhiên của mô hình. Đôi khi câu trả lời ngắn gọn, đúng trọng tâm và an toàn lại bị judge chấm khắt khe ở phần completeness.
2. Giới hạn cố hữu của Snapshot: Bộ dữ liệu offline chỉ gói gọn trong 50 sự cố gần nhất tính đến tháng 11/2026. Do đó, agent hoàn toàn không thể giải quyết các câu hỏi về lịch sử xa hoặc cập nhật theo thời gian thực ngoài đời, mà chỉ có thể giải thích rõ ràng phạm vi dữ liệu mình có.
3. Hạn chế của việc đánh giá qua 1 lần chạy: Điểm tăng từ 77.82 lên 87.05 cho thấy các thay đổi đi đúng hướng, nhưng việc chỉ chạy mỗi phiên bản một lần chưa phản ánh hết phương sai (variance) của model trên tập dữ liệu.
