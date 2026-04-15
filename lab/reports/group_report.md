# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Phạm Tuấn Anh | Ingestion / Raw Owner/Monitoring / Docs Owner | Bintuananh2003@gmail.com |
| ___ | Cleaning & Quality Owner | ___ |
| ___ | Embed & Idempotency Owner | ___ |

**Ngày nộp:** 15/04/2026
**Repo:** https://github.com/AnhPham0411/day10-c401-a1
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan (150–200 từ)

**Tóm tắt luồng:**
Tóm tắt luồng:
-Hệ thống thực hiện nạp liệu từ file trích xuất thô policy_export_dirty.csv. Luồng đi qua 4 giai đoạn chính:

-Ingest: Đọc dữ liệu thô, định danh phiên chạy bằng run_id dựa trên timestamp UTC để đảm bảo tính duy nhất.

-Transform & Validate: Làm sạch dữ liệu và kiểm tra qua bộ quy tắc chất lượng (expectations).

-Embed: Thực hiện upsert dữ liệu sạch vào ChromaDB với cơ chế prune để xóa các vector cũ không còn tồn tại, đảm bảo tính Idempotent.

-Monitor: Xuất file Manifest và kiểm tra độ tươi (freshness) so với SLA.
_________________

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**
python etl_pipeline.py run
_________________

---

## 2. Cleaning & expectation (150–200 từ)

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| Data Ingestion & Freshness | 10 raw records | FAIL (Age: 118h) | manifest_2026-04-15T05-42Z.json |

**Rule chính (baseline + mở rộng):**

- …

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

_________________

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

_________________

**Kết quả định lượng (từ CSV / bảng):**

_________________

---

## 4. Freshness & monitoring (100–150 từ)

Nhóm thống nhất thiết lập SLA = 24 giờ cho độ tươi của dữ liệu chính sách bảo hiểm.

PASS: Dữ liệu cập nhật trong < 24h (Dữ liệu tin cậy).

WARN: 24h - 48h (Cần kiểm tra hệ thống trích xuất).

FAIL: > 48h (Dữ liệu cũ, có nguy cơ gây sai lệch phản hồi của AI).

Phân tích phiên chạy 2026-04-15T05-42Z:
Hệ thống báo FAIL với age_hours là 118.058. Điều này cho thấy dữ liệu nguồn (latest_exported_at: 2026-04-10) đã không được cập nhật trong gần 5 ngày. Cơ chế giám sát đã phát hiện chính xác rủi ro dữ liệu lỗi thời (stale data) trước khi đưa vào phục vụ Agent.

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

_________________

---

## 6. Rủi ro còn lại & việc chưa làm
Rủi ro: Pipeline hiện tại chưa tự động chặn (Halt) luồng Embed khi Freshness báo FAIL, dẫn đến việc dữ liệu cũ vẫn bị ghi đè vào Vector DB nếu không có sự can thiệp thủ công.

Việc chưa làm: Thiết lập hệ thống thông báo tự động (Alerting) qua Telegram/Email cho Monitoring Owner khi trạng thái manifest chuyển sang FAIL.