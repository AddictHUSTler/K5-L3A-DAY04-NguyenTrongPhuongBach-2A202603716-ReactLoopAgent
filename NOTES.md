# Ghi chú phân tích · Day 04

Nguyễn Trọng Phương Bách - MSSV: 2A202603716

**CP1–CP3:** [hai tool trả JSON](logs/cp1.log); [agent gọi LLM thật](logs/cp2.json) trả `completed` sau 2 model calls; [tests CP3](logs/cp3.log) đạt 40/42, còn 2 lỗi validation nguồn.

## 1. Case được chọn

**E2 đã đọc incident nhưng không dẫn URL.** Agent nêu đúng phạm vi FedRAMP của lỗi cache purge, nhưng phần nguồn chỉ có timestamp, ID, tên và nội dung update. Điểm trước sửa là **95/100**; lỗi thiếu URL được xác định bằng cách đọc answer và trace, không phải lý do judge trừ điểm.

## 2. Bằng chứng trong trace

Trong [eval_before.json](eval_before.json), tìm case `E2`:

- Step 1: `get_status` tìm Cloudflare `cache purge`.
- Step 2: `get_incident_details` cho `wp49bp8bqo6c` trả `ok: true` và URL `https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c`.
- Step 3: `finish: stop`, run trả `completed`, answer thiếu URL trên.

**Nguyên nhân:** loop chưa kiểm tra URL trước khi trả `completed`. Model có thể coi ID và trích update là đủ cho yêu cầu “Dẫn nguồn từ tool”; đây là giả thuyết, không biết được suy luận bên trong model.

## 3. Thay đổi đã thực hiện

[agent.py](agent.py) lưu URL từ `get_incident_details` có `ok: true`. Trước khi hoàn thành:

- Answer chứa ít nhất một URL đã lưu: cho hoàn thành.
- Thiếu tất cả URL đã lưu: ghi `validation_missing_source`, yêu cầu model sửa trong `max_steps`.
- Chưa đọc được URL nào: không bắt thêm nguồn. Hết lượt mà chưa đạt: trả `max_steps`.

[Tests CP5](logs/cp5.log): **42/42 pass**, gồm sửa thiếu URL, hết lượt và tool thất bại. Khi rà soát, chạy lại `python3 -m unittest discover -s tests -v`: **42/42 pass**.

## 4. So sánh hai lần chạy

[Trước — CP4](eval_before.json) và [sau — CP6](eval_after.json) có `config` giống hệt: agent và judge đều dùng `openai` / `gpt-5.4-mini`, `max_steps = 8`, dataset `core`. Cả hai đủ E1–E6, A1–A8, H1–H8; trace ghi cùng timestamp snapshot ngày **09/11/2026**.

| Chỉ số | Trước | Sau |
|---|---:|---:|
| Điểm tổng /100 | 91.0 | 92.55 |
| Case được chấm | 22/22 | 22/22 |
| Run `completed` | 22/22 | 22/22 |
| E2 /100 | 95 | 92 |
| Answer thiếu tất cả URL incident đã đọc | 10 | 0 |
| Agent model calls | 57 | 75 |
| Judge model calls | 22 | 22 |
| Agent tokens | 61,044 | 107,789 |
| Judge tokens | 48,775 | 47,859 |
| Tổng tokens | 109,819 | 155,648 |

- **E2 đã sửa:** step 3 ghi validation, step 4 trả URL đúng. Calls tăng 3 → 4, điểm giảm 95 → 92. Toàn bộ lần sau có 11 event validation.
- **Cải thiện khác:** H3 50 → 100, gọi details cho ID giả và nhận `INCIDENT_NOT_FOUND`; H6 60 → 90, đọc tiếp incident liên quan ở step 2. Những lựa chọn tool này xảy ra trước validation, chưa thể quy cho cải tiến CP5.
- **Lỗi còn lại:** H7 50 → 54, vẫn kết luận Spectrum `operational` dù step 1 trả `TimeoutError`. H1 92 → 86, chưa đọc incident liên quan `xr3myzj7ms51`.
- **A5 sau khi sửa bị tệ đi:** 96 → 81, calls 3 → 7. Sau validation step 5, step 6 đọc lại details đã có; answer vẫn đưa ví dụ năm 2026 vào yêu cầu năm 2025.
- **Chi phí:** tổng token tăng **45,829**; usage không có số tiền. Các số trên chỉ tính hai file eval.

## 5. Giới hạn và kiểm tra nộp bài

- Validation chỉ kiểm tra sự hiện diện của ít nhất một URL, không xác minh từng nhận định hay nguồn từng provider. Tests pass và `completed` không chứng minh answer đúng.
- Snapshot không xác nhận trạng thái hiện tại. Tool tìm theo chuỗi trong 50 incident gần nhất, trả tối đa 5 kết quả; details giữ tối đa 3 update, mỗi update tối đa 900 ký tự. Không tìm thấy không chứng minh không có incident.
