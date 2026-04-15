Checklist cho Vai trò 1: Ingestion & Monitoring Owner (Người phụ trách Nạp liệu & Giám sát)
Vai trò này đảm bảo dữ liệu thô được đưa vào pipeline một cách đáng tin cậy và có thể giám sát được, đặc biệt là về độ tươi (freshness) của dữ liệu.

[x] Thiết lập Pipeline ban đầu:
Chịu trách nhiệm chính cho phần code đọc dữ liệu từ data/raw/policy_export_dirty.csv trong etl_pipeline.py.
Đảm bảo mỗi lần chạy pipeline (python etl_pipeline.py run) sinh ra một run_id duy nhất.
Đảm bảo run_id và số lượng raw_records được ghi lại chính xác trong file log.
[x] Tạo Artifacts & Manifests:
Đảm bảo pipeline tạo ra file manifests/manifest_<run-id>.json sau mỗi lần chạy.
Kiểm tra file manifest chứa đủ các thông tin quan trọng như run_id, raw_records, cleaned_records, quarantine_records, và các timestamp.
[x] Giám sát Freshness:
Thảo luận và cấu hình FRESHNESS_SLA_HOURS trong file contracts/data_contract.yaml.
Thực thi lệnh python etl_pipeline.py freshness --manifest ... để kiểm tra độ tươi của dữ liệu.
[x] Hoàn thành Tài liệu:
Điền vào phần "source map" trong docs/data_contract.md, mô tả ít nhất 2 nguồn dữ liệu, các rủi ro và cách đo lường.
Viết phần "Symptom" (Triệu chứng) và "Detection" (Phát hiện) trong docs/runbook.md, giải thích ý nghĩa của các trạng thái freshness (PASS/WARN/FAIL).
Sản phẩm chính cần hoàn thành:

Code nạp dữ liệu trong etl_pipeline.py.
Các file trong artifacts/logs/ và artifacts/manifests/.
Tài liệu docs/data_contract.md.
Một phần của docs/runbook.md.
✅ Checklist cho Vai trò 2: Cleaning & Quality Owner (Người phụ trách Làm sạch & Chất lượng)
Đây là vai trò trung tâm, quyết định "sức khỏe" của dữ liệu trước khi nó được sử dụng. Bạn sẽ viết các quy tắc làm sạch và các kỳ vọng về chất lượng.

[ ] Viết Quy tắc Làm sạch (Cleaning Rules):
Nghiên cứu các rule làm sạch có sẵn trong transform/cleaning_rules.py.
Viết thêm ít nhất 3 rule làm sạch mới để xử lý các vấn đề trong policy_export_dirty.csv (ví dụ: xử lý doc_id không hợp lệ, chuẩn hóa version, loại bỏ dòng lỗi...).
Đảm bảo logic của bạn tạo ra được dữ liệu đã sạch và dữ liệu bị "cách ly" (quarantine).
Đảm bảo pipeline tạo ra file artifacts/quarantine/quarantine_<run-id>.csv và ghi log số lượng quarantine_records.
[ ] Viết Kỳ vọng Chất lượng (Quality Expectations):
Nghiên cứu các "expectation" có sẵn trong quality/expectations.py.
Viết thêm ít nhất 2 expectation mới để kiểm tra chất lượng của dữ liệu sau khi đã làm sạch.
Quyết định xem expectation nào khi vi phạm sẽ chỉ cảnh báo (WARN) và expectation nào sẽ dừng toàn bộ pipeline (HALT).
[ ] Chứng minh Tác động (Chống Trivial):
Điền thông tin vào bảng metric_impact trong reports/group_report.md. Bạn phải chứng minh rằng các rule/expectation mới của mình có tác động thực tế và có thể đo lường được (ví dụ: làm tăng quarantine_records khi cố tình đưa dữ liệu lỗi vào).
Sản phẩm chính cần hoàn thành:

File transform/cleaning_rules.py và quality/expectations.py đã được mở rộng.
Các file trong artifacts/quarantine/.
Bảng metric_impact trong reports/group_report.md.
Đóng góp chính vào docs/quality_report.md.
✅ Checklist cho Vai trò 3: Embed & Evidence Owner (Người phụ trách Vector hóa & Bằng chứng)
Vai trò của bạn là đảm bảo dữ liệu sạch được chuyển thành vector một cách an toàn, có thể chạy lại nhiều lần (idempotent), và quan trọng nhất là tạo ra bằng chứng "trước và sau" để chứng minh hiệu quả của pipeline.

[ ] Đảm bảo Idempotency:
Xác minh rằng việc chạy lại pipeline (etl_pipeline.py run) nhiều lần không làm tăng số lượng vector trong ChromaDB một cách không kiểm soát.
Kiểm tra log để chắc chắn rằng cơ chế prune (xóa vector cũ) hoạt động, thể hiện qua log embed_prune_removed.
[ ] Tạo Bằng chứng "Trước và Sau" (Before/After Evidence):
Bước 1 (Sau khi sửa): Chạy pipeline ở chế độ chuẩn, sau đó chạy eval_retrieval.py để có kết quả retrieval trên dữ liệu sạch. Lưu lại file eval này.
Bước 2 (Trước khi sửa): Cố tình chạy pipeline với dữ liệu bị lỗi, ví dụ: python etl_pipeline.py run --no-refund-fix --skip-validate.
Bước 3 (Đo lường lỗi): Chạy lại eval_retrieval.py trên cơ sở dữ liệu vector bị lỗi. Lưu lại file eval này.
Bước 4 (Tổng hợp): So sánh 2 file kết quả eval và trình bày rõ ràng trong reports/group_report.md, cho thấy retrieval đã tệ đi như thế nào và tốt lên ra sao.
[ ] Hoàn thiện và Chấm điểm:
Sau 17:00, chạy grading_run.py để tạo ra file artifacts/eval/grading_run.jsonl để nộp bài.
Hoàn thành sơ đồ trong docs/pipeline_architecture.md.
Viết phần "Diagnosis", "Mitigation", và "Prevention" trong docs/runbook.md.
Sản phẩm chính cần hoàn thành:

Code liên quan đến embed và prune trong etl_pipeline.py.
Các file kết quả trong artifacts/eval/, đặc biệt là file so sánh trước/sau và file grading_run.jsonl.
Phần "Before / after" trong reports/group_report.md.
Tài liệu docs/pipeline_architecture.md và các phần còn lại của docs/runbook.md.