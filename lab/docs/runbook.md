# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom
Người dùng thấy: AI trả lời thông tin cũ (ví dụ: vẫn báo chính sách hoàn tiền là 14 ngày trong khi thực tế đã đổi thành 7 ngày).

Agent/System: Hệ thống RAG trả về các chunk dữ liệu có exported_at quá xa so với hiện tại.

Cảnh báo hệ thống: Pipeline kết thúc với log freshness_check=FAIL.
---

## Detection
Freshness Metric: age_hours (117.72h) vượt quá SLA_HOURS (24h) trong file Manifest.

Expectation Fail: Các bộ lọc chất lượng tại quality/expectations.py báo lỗi (nếu có).

Eval Metric: Chạy eval_retrieval.py cho kết quả hits_forbidden tăng cao (AI lấy nhầm dữ liệu đáng lẽ phải bị loại bỏ).
---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xác định mốc latest_exported_at. Nếu cũ hơn 24h thì lỗi do nguồn cấp (Upstream). |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem các dòng bị loại. Nếu số lượng quarantine_records tăng đột biến, rule làm sạch đang quá chặt hoặc dữ liệu thô bị lỗi cấu trúc. |
| 3 | Chạy `python eval_retrieval.py` | Kiểm tra xem Vector DB có đang chứa dữ liệu "rác" (stale data) không. Chỉ số precision giảm là bằng chứng xác thực. |

---

## Mitigation
Tạm thời: Treo banner "Data Stale" hoặc "Dữ liệu đang được cập nhật" trên giao diện chatbot để cảnh báo người dùng.

Rollback: Nếu dữ liệu mới quá lỗi, thực hiện xóa bộ chỉ mục cũ và nạp lại bản backup manifest gần nhất có trạng thái PASS.

Rerun: Liên hệ bộ phận Upstream đẩy lại file policy_export_dirty.csv mới nhất và chạy lại python etl_pipeline.py run.
---

## Prevention
Thêm Guardrails: Cấu hình để Pipeline tự động HALT (dừng hẳn) nếu freshness_check trả về FAIL, không cho phép upsert vào Vector DB.

Alerting: Tích hợp gửi thông báo Telegram/Slack ngay khi manifest được ghi với trạng thái lỗi.

Owner: Thiết lập quy trình kiểm tra định kỳ (Daily Healthcheck) cho file data_contract.yaml.