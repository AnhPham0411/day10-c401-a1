# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Phạm Tuấn Anh | Ingestion / Raw Owner/Monitoring / Docs Owner | Bintuananh2003@gmail.com |
| Vũ Lê Hoàng | Cleaning & Quality Owner | hoanglevu1705@gmail.com |
| Vũ Hồng Quang | Embed & Idempotency Owner | quangzetsu@gmail.com |

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
| Data Ingestion & Freshness | 10 raw records | FAIL (Age: 118h) | `manifest_2026-04-15T05-42Z.json` |
| [Rule] Lọc System Error | 0 quarantine do lỗi | 1 bị loại (`contains_system_error`) | `quarantine_2026-04-15T09-29Z.csv` (row 11) |
| [Rule] Lọc text quá ngắn | 0 quarantine do ngắn | 1 bị loại (`chunk_text_too_short_for_llm`) | `quarantine_2026-04-15T09-29Z.csv` (row 12) |
| [Rule] Lọc bản draft | 0 quarantine do draft | 1 bị loại (`draft_policy_not_allowed`) | `quarantine_2026-04-15T09-29Z.csv` (row 13) |
| [Expectation] Không chứa system error | N/A | OK (**halt**) :: errors=0 | `run_inject-bad.log` dòng 130 |
| [Expectation] Không chứa bản nháp | N/A | OK (**halt**) :: drafts=0 | `run_inject-bad.log` dòng 131 |

**3 Rule mới (cleaning_rules.py — dòng 155–168):**

- `contains_system_error` — Lọc chunk chứa từ khóa lỗi hệ thống (`error`, `traceback`, `fatal`, `exception`). Ngăn rác từ trích xuất DB lọt vào vector store.
- `chunk_text_too_short_for_llm` — Lọc chunk dưới 15 ký tự (ví dụ: "Too short"). Văn bản quá ngắn không đủ ngữ cảnh cho Agent/LLM.
- `draft_policy_not_allowed` — Lọc policy có nhãn `[DRAFT]` hoặc "bản nháp". Đảm bảo chỉ chính sách đã phê duyệt mới được nạp.

**2 Expectation mới (expectations.py — dòng 139–159):**

- `no_system_error_messages` (**halt**) — Đảm bảo cleaned output không còn thông báo lỗi hệ thống. Nếu FAIL → dừng pipeline.
- `no_draft_policies` (**halt**) — Đảm bảo cleaned output không còn bản nháp. Nếu FAIL → dừng pipeline.

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

Thử nghiệm xóa rule lọc draft khỏi `cleaning_rules.py` dẫn đến expectation `no_draft_policies` báo `FAIL (drafts=1)` và dừng (**HALT**) toàn bộ pipeline. Lỗi này đảm bảo rác không thể được đưa vào ChromaDB.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject (run_id=`inject-bad`):**
Chúng tôi đã cố tình chạy pipeline với cờ `--no-refund-fix --skip-validate` (bỏ qua bước fix "14 ngày" thành "7 ngày" trong policy refund và ép ghi đè các lỗi Expectation vào Vector DB). Việc này nhằm mô phỏng lại lỗi khi một kỹ sư vô tình bỏ qua khâu Validation.

**Kết quả định lượng (từ CSV / bảng):**

| Câu hỏi eval | Trước (inject-bad / eval_dirty) | Sau (fix-good / eval_clean) | File chứng cứ |
|---|---|---|---|
| `q_refund_window` | `hits_forbidden=yes` ❌ | `hits_forbidden=no` ✅ | `eval_dirty.csv` vs `eval_clean.csv` |
| `q_leave_version` | `contains_expected=yes`, `top1_doc_matches=yes` | `contains_expected=yes`, `top1_doc_matches=yes` ✅ | `before_after_eval.csv` |

Theo kết quả đánh giá (trong file `artifacts/eval/eval_dirty.csv`), việc inject đã làm hỏng Retrieval cho `q_refund_window`: trường `hits_forbidden` bị đổi thành `yes` — text cũ chứa "14 ngày làm việc" đã bị ChromaDB lấy ra. Sau khi chạy lại pipeline chuẩn (run_id=`fix-good`), `hits_forbidden=no` và `q_leave_version` cũng đạt `top1_doc_matches=true`.

Như vậy, hệ thống Cleaning & Expectations hiện hành đóng vai trò tối quan trọng, tránh tình trạng LLM Agent sinh ra câu trả lời sai lệch gây mất uy tín với khách hàng.

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

Dữ liệu sau khi Embed của Day 10 (lưu vào collection `day10_kb`) chạy hoàn toàn độc lập nhưng được thiết kế để cấp nguồn dữ liệu chuẩn cho Multi-Agent Day 09.
Thay vì đọc từ file tĩnh, các Agent (như Supervisor, QA Worker) ở Day 09 có thể trực tiếp query vào ChromaDB của Day 10 để tìm kiếm tài liệu. Cách tiếp cận này giúp các Agent luôn lấy được luật bảo hiểm/nhân sự "tươi" nhất và đã được tự động loại bỏ rác (như file nháp, lỗi hệ thống) qua bộ quy tắc Data Quality.

---

## 6. Rủi ro còn lại & việc chưa làm
Rủi ro: Pipeline hiện tại chưa tự động chặn (Halt) luồng Embed khi Freshness báo FAIL, dẫn đến việc dữ liệu cũ vẫn bị ghi đè vào Vector DB nếu không có sự can thiệp thủ công.

Việc chưa làm: Thiết lập hệ thống thông báo tự động (Alerting) qua Telegram/Email cho Monitoring Owner khi trạng thái manifest chuyển sang FAIL.
