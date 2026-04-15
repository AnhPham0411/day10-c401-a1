# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| Policy Export System | Batch (CSV Export) |Hệ thống thượng nguồn không xuất file hoặc xuất file trống.|raw_records == 0 (Critical Alert) |
| Local File System | Path-based loading |File bị lỗi định dạng (Corrupted) hoặc sai phân quyền (Permission denied).|IOError / FileNotFound |
| Data Freshness | Timestamp Comparison |Dữ liệu cũ, không được cập nhật hàng ngày (Stale data).|age_hours > 24 (Freshness FAIL) |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | Định dạng: {doc_id}_v{version}_{index} |
| doc_id | string | Có | Mã định danh văn bản (ví dụ: POL-001) |
| chunk_text | string | Có | Nội dung đã làm sạch, độ dài tối thiểu 8 ký tự |
| effective_date | date | Có | Ngày hiệu lực (ISO 8601: YYYY-MM-DD) |
| exported_at | datetime | Có | Thời điểm trích xuất dữ liệu từ DB gốc |

---

## 3. Quy tắc quarantine vs drop

Quarantine (Cách ly): Các bản ghi vi phạm quy tắc logic (ví dụ: doc_id trống, effective_date sai định dạng, hoặc chính sách hoàn tiền nằm ngoài cửa sổ cho phép).

Hành động: Ghi vào artifacts/quarantine/quarantine_<run_id>.csv.

Approve: Vai trò Cleaning & Quality Owner (Role 2) sẽ rà soát và sửa rule nếu cần.

Drop (Hủy bỏ): Các dòng dữ liệu hoàn toàn vô nghĩa, dòng trống hoặc không có nội dung văn bản.

Hành động: Loại bỏ hoàn toàn khỏi pipeline và ghi log số lượng bị hủy.

## 4. Phiên bản & canonical

Source of Truth: File data/raw/policy_export_dirty.csv là nguồn thực hiện duy nhất (Snapshot) cho bài Lab này.

Version Control: Dựa vào trường exported_at. Bản ghi có exported_at mới nhất cho cùng một doc_id sẽ được coi là Canonical Version (Phiên bản chuẩn).

Vector Sync: Sử dụng cơ chế upsert trong ChromaDB dựa trên chunk_id để đảm bảo tính Idempotent (không trùng lặp dữ liệu khi chạy lại).
