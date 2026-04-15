# Quality report — Lab Day 10 (nhóm)

**run_id:** lab/artifacts/logs/run_fix-good.log
**Ngày:** 15/04/2016

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước | Sau | Ghi chú |
|--------|-------|-----|---------|
| raw_records | 14| 14| |
| cleaned_records | 6| 6| |
| quarantine_records |8 |8 | |
| Expectation halt? | | | |

---

## 2. Before / after retrieval (bắt buộc)

**Trước khi sửa:** lab/artifacts/eval/inject_bad.csv
**Sau khi sửa:**   lab/artifacts/eval/before_after_eval.csv

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước:** q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,yes,,3
**Sau:** q_refund_window,Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền kể từ khi xác nhận đơn?,policy_refund_v4,Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.,yes,no,,3

**Merit (khuyến nghị):** versioning HR — `q_leave_version` (`contains_expected`, `hits_forbidden`, cột `top1_doc_expected`)

**Trước:**  q_leave_version,Theo chính sách nghỉ phép hiện hành (2026), nhân viên dưới 3 năm kinh nghiệm được bao nhiêu ngày phép năm?,hr_leave_policy,Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026.,yes,no,yes,3
**Sau:** q_leave_version,Theo chính sách nghỉ phép hiện hành (2026), nhân viên dưới 3 năm kinh nghiệm được bao nhiêu ngày phép năm?,hr_leave_policy,Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026.,yes,no,yes,3

---

## 3. Freshness & monitor

> Kết quả `freshness_check` (PASS/WARN/FAIL) và giải thích SLA bạn chọn.
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_fix-good.json
PASS {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 122.482, "sla_hours": 130.0}
Tôi chọn SLA là 130 là để phù hợp với dữ liệu demo. SLA mặc định là 24 giờ nhưng tần suất cập nhật nguồn đang là khoảng 120 giờ.
---

## 4. Corruption inject (Sprint 3)

> Mô tả cố ý làm hỏng dữ liệu kiểu gì (duplicate / stale / sai format) và cách phát hiện.
Tôi làm hỏng dữ liệu bằng cách duplicate và sai format. Tôi kiểm tra bằng cách xem trong quarantine_records, và thấy chunk trùng trong đó và sai format trong đó
---

## 5. Hạn chế & việc chưa làm
freshness hiện chỉ được kiểm tra thủ công bằng manifest và FRESHNESS_SLA_HOURS; chưa có cơ chế tự động alert hoặc dừng pipeline khi FAIL.
SLA hiện chưa có cấu hình thực sự động/đúng nguồn: 130 đang là giá trị thử nghiệm để phù hợp dữ liệu demo, không phải SLA sản xuất.
Chưa tích hợp alert channel thực tế (Slack/Email) như đã định nghĩa trong contracts/data_contract.yaml.
- …
