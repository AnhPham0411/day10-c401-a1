# Báo cáo cá nhân — Vũ Lê Hoàng

**Họ và tên:** Vũ Lê Hoàng  
**Vai trò:** Cleaning & Quality Owner  
**Độ dài:** ~500 từ

---

## 1. Phụ trách

Tôi đảm nhận vai trò **Cleaning & Quality Owner**, chịu trách nhiệm toàn bộ khâu làm sạch dữ liệu và kiểm tra chất lượng trước khi đẩy vào ChromaDB.

**Phạm vi code:**

- `transform/cleaning_rules.py` — Thêm 3 rule mới (dòng 155–168): `contains_system_error`, `chunk_text_too_short_for_llm`, `draft_policy_not_allowed`. Các rule này bổ sung trước vòng kiểm allowlist baseline, đảm bảo rác bị chặn sớm nhất.
- `quality/expectations.py` — Thêm 2 expectation mới (dòng 139–159): `no_system_error_messages` (halt) và `no_draft_policies` (halt). Hai expectation này hoạt động như lưới an toàn cuối cùng, đảm bảo cleaned output hoàn toàn sạch trước khi embed.
- `data/raw/policy_export_dirty.csv` — Bổ sung 3 dòng inject (row 11–13) để kiểm tra các rule mới hoạt động đúng.
- `docs/quality_report.md` — Viết tài liệu tổng hợp về các rule và expectation mới.

**Bằng chứng:** commit `6e70764` ("Expectations and cleaning_rules update"), `cfc9e41` ("Testing etl pipeline for the 1st time"), `fcc36da` ("Update group_report.md").

---

## 2. Quyết định kỹ thuật

**Halt vs warn cho 2 expectation mới:** Tôi chọn severity **halt** cho cả `no_system_error_messages` và `no_draft_policies` thay vì warn. Lý do: nếu cleaned output vẫn chứa message lỗi hệ thống (ví dụ: `"ERROR: database connection lost during fetch"`) hoặc bản nháp chưa duyệt (`"[DRAFT] bản nháp"`), dữ liệu này khi lọt vào ChromaDB sẽ làm Agent trả lời sai lệch — đây là rủi ro nghiêm trọng hơn nhiều so với việc dừng pipeline. Ngược lại, `exported_at_iso_or_empty` (E8 baseline) chỉ là **warn** vì `exported_at` rỗng không gây sai nội dung trả lời, chỉ ảnh hưởng freshness tracking.

**Thứ tự đặt rule:** Tôi đặt 3 rule mới **trước** vòng kiểm allowlist (`doc_id not in ALLOWED_DOC_IDS`). Lý do: nếu đặt sau, các chunk có `doc_id` hợp lệ nhưng chứa lỗi hệ thống hoặc bản nháp sẽ không bị chặn. Thứ tự cleaning: `_clean_chunk_text()` → `contains_system_error` → `chunk_text_too_short_for_llm` → `draft_policy_not_allowed` → allowlist → normalize date → dedupe → refund fix.

---

## 3. Sự cố / anomaly

Khi inject 3 dòng mới (row 11: `"ERROR: database connection lost"`, row 12: `"Too short"`, row 13: `"[DRAFT] bản nháp"`) vào `policy_export_dirty.csv`, lần chạy đầu tiên (`run_id=2026-04-15T09-29Z`) cho kết quả:
- `raw_records=14`, `cleaned_records=6`, `quarantine_records=8` (trước đó chỉ 4 quarantine với 10 raw records).

Điều đáng chú ý: quarantine tăng đúng 3 bản ghi tương ứng 3 rule mới, trong khi `cleaned_records` vẫn giữ nguyên = 6. Điều này chứng minh các rule mới **không ảnh hưởng** đến dữ liệu sạch hợp lệ, chỉ chặn đúng rác.

**Chứng cứ:** `quarantine_2026-04-15T09-29Z.csv` — row 11 lý do `contains_system_error`, row 12 lý do `chunk_text_too_short_for_llm`, row 13 lý do `draft_policy_not_allowed`.

---

## 4. Before/after

**Before (run_id=`inject-bad`, chế độ `--no-refund-fix --skip-validate`):**

```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
expectation[no_system_error_messages] OK (halt) :: errors=0
expectation[no_draft_policies] OK (halt) :: drafts=0
```
→ `eval_dirty.csv`: dòng `q_refund_window` có `hits_forbidden=yes` ❌

**After (run_id=`fix-good`, pipeline chuẩn):**

```
expectation[refund_no_stale_14d_window] OK (halt) :: violations=0
expectation[no_system_error_messages] OK (halt) :: errors=0
expectation[no_draft_policies] OK (halt) :: drafts=0
```
→ `eval_clean.csv`: dòng `q_refund_window` có `hits_forbidden=no` ✅  
→ `before_after_eval.csv`: dòng `q_leave_version` có `top1_doc_matches=yes` ✅

---

## 5. Cải tiến thêm 2 giờ

Thêm bộ dữ liệu inject chuyên biệt vào `policy_export_dirty.csv` để mỗi rule mới đều có chứng cứ quarantine rõ ràng, thay vì chỉ test trên dữ liệu baseline. Đồng thời viết `docs/quality_report.md` hướng dẫn cách reproduce và kiểm tra impact của từng rule, giúp các thành viên khác (Embed Owner, Monitoring Owner) hiểu rõ dữ liệu nào bị chặn và tại sao.
