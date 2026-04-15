# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:**Phạm Tuấn Anh
**Vai trò:** Ingestion / Raw Owner/Monitoring / Docs Owner 
**Ngày nộp:** 15/04/2026
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

-etl_pipeline.py: Phụ trách chính phần nạp liệu (Ingestion) và quản lý phiên chạy (run_id).

-artifacts/manifests/: Thiết lập cấu trúc file Manifest để lưu trữ metadata của pipeline.

-contracts/data_contract.yaml: Định nghĩa SLA cho độ tươi của dữ liệu (Freshness).

-docs/runbook.md: Xây dựng quy trình xử lý sự cố dựa trên các trạng thái giám sát.

**Kết nối với thành viên khác:**

Tôi là "người gác cổng" đầu tiên. Tôi cung cấp run_id đồng nhất để bạn Cleaning Owner phân loại dữ liệu và bạn Embed Owner đánh dấu metadata trong Vector DB. Kết quả freshness_check của tôi giúp nhóm biết dữ liệu có đủ độ tin cậy để thực hiện Evaluation hay không.

**Bằng chứng (commit / comment trong code):**

Commit: "Feat: Implement unique run_id generation and manifest export logic."

Log thực tế: run_id=2026-04-15T05-42Z, manifest_written=artifacts\manifests\manifest_2026-04-15T05-42Z.json.
---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi quyết định sử dụng chiến lược Timestamp-based Versioning kết hợp với Strict SLA Monitoring để quản lý độ tươi của dữ liệu. Cụ thể, thay vì chỉ kiểm tra xem file có tồn tại hay không, tôi trích xuất trường latest_exported_at từ dữ liệu thô và so sánh với thời gian thực thi pipeline.

Tôi thiết lập ngưỡng SLA_HOURS = 24. Quyết định này giúp hệ thống có tính "tự nhận thức" (Self-awareness). Nếu dữ liệu cũ quá 1 ngày, hệ thống sẽ tự động chuyển trạng thái FAIL trong manifest. Điều này cực kỳ quan trọng đối với các chatbot tư vấn bảo hiểm, nơi một chính sách cũ có thể dẫn đến sai lệch pháp lý nghiêm trọng. Việc lưu manifest dưới dạng JSON giúp các module sau này (như Dashboard giám sát) có thể đọc hiểu tự động mà không cần can thiệp thủ công.

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Triệu chứng: Trong phiên chạy thử nghiệm, tôi phát hiện hệ thống báo PIPELINE_OK nhưng kết quả tư vấn của Agent lại không thay đổi dù đã cập nhật file thô.

Phát hiện: Qua kiểm tra lệnh python etl_pipeline.py freshness, tôi phát hiện thông số:

"age_hours": 118.058, "reason": "freshness_sla_exceeded".

Dữ liệu thô đang được nạp thực chất là snapshot từ 5 ngày trước (2026-04-10), không phải dữ liệu mới nhất.

Xử lý: Tôi đã cập nhật lại data_contract.yaml để làm rõ quy định về độ tươi của dữ liệu và viết bổ sung phần Diagnosis trong runbook.md. Tôi hướng dẫn nhóm cách kiểm tra mốc latest_exported_at trong manifest để xác định lỗi do nguồn cấp (Upstream) thay vì lỗi code tại pipeline.

## 4. Bằng chứng trước / sau (80–120 từ)

Dưới đây là minh chứng từ file manifest được trích xuất trong quá trình vận hành:

Run ID: 2026-04-15T05-42Z

Trước khi Ingest: Hệ thống trống, không có vết quản lý.

Sau khi Ingest: ```json
"raw_records": 10,
"cleaned_records": 6,
"quarantine_records": 4,
"freshness_check": "FAIL" (Age: 118h)

Bằng chứng này cho thấy tôi đã kiểm soát được chính xác số lượng dữ liệu lỗi bị loại bỏ (4 records) và cảnh báo thành công trạng thái dữ liệu cũ.

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ lập trình tính năng Hard-Stop Pipeline. Hiện tại khi Freshness báo FAIL, hệ thống vẫn cho phép tiếp tục Embed. Tôi muốn thêm logic để pipeline tự động dừng hoàn toàn và gửi cảnh báo ngay lập tức qua Telegram Bot khi phát hiện dữ liệu vi phạm SLA, ngăn chặn tuyệt đối việc nạp dữ liệu cũ vào môi trường Production.
