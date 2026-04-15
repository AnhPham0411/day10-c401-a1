# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Vũ Hồng Quang
**Ngày nộp:** 15/04/2026
**Vai trò:** Embed & Evidence Owner
**Độ dài yêu cầu:** 400–650 từ

---

> Viết "tôi", đính kèm run_id, tên file, đoạn log hoặc dòng CSV thật.
> Lưu: `reports/individual/VuHongQuang.md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Tôi phụ trách phần Embed và Evidence trong pipeline Day 10. Công việc chính của tôi là đảm bảo dữ liệu sạch được upsert vào ChromaDB một cách idempotent và tạo ra bằng chứng before/after cho retrieval quality. Xác minh rằng việc chạy lại pipeline (etl_pipeline.py run) nhiều lần không làm tăng số lượng vector trong ChromaDB một cách không kiểm soát.

**Bằng chứng:** tôi đã dùng log `artifacts/logs/run_fix-good.log` và file eval `artifacts/eval/inject-bad.csv`.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Kiểm tra log để chắc chắn rằng cơ chế prune (xóa vector cũ) hoạt động, thể hiện qua log embed_prune_removed.
Việc này cos bằng chứng rõ ràng: log `embed_prune_removed=1` trong `artifacts/logs/run_2026-04-15T07-49Z.log` cho thấy prune đã thực sự chạy. Nếu không dùng cơ chế prune, rerun sẽ dần chất thêm id cũ và mất idempotency.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Một anomaly tôi xử lý là tình huống dữ liệu bị ảnh hưởng khi chạy pipeline với `--no-refund-fix --skip-validate`. Trong run này, expectation `refund_no_stale_14d_window` đã fail với `violations=1`, nhưng pipeline vẫn tiếp tục embed do skip-validate. Kết quả là ChromaDB có thể chứa dữ liệu không chuẩn.

Tôi dùng log `artifacts/logs/run_2026-04-15T07-44Z.log` để chứng minh anomaly và kiểm tra lại bằng `artifacts/eval/eval_dirty.csv`. Sự khác biệt là `q_refund_window` bị `hits_forbidden=yes` trong data dirty, cho thấy phần embedding đã nhận dữ liệu corrupt và cần guardrail cứng hơn trong tương lai.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Bằng chứng before/after được lấy từ hai file đánh giá thực tế: `artifacts/eval/eval_clean.csv` và `artifacts/eval/eval_dirty.csv`. Với `q_refund_window`, clean có `hits_forbidden=no` còn dirty có `hits_forbidden=yes`. Điều này chứng tỏ tuy top1 vẫn cùng `policy_refund_v4`, nhưng top-k của retrieval đã nhiễm thông tin sai, làm tăng rủi ro endpoint trả lời không đúng.

Tôi cũng dùng `run_id=2026-04-15T07-49Z` cho chạy sạch và `run_id=2026-04-15T07-44Z` cho chạy inject để so sánh trực tiếp.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Tiếp theo tôi muốn thêm kiểm tra số lượng vector thực tế trong ChromaDB trước/sau run để so sánh idempotency, và tích hợp alert auto khi `embed_prune_removed` xuất hiện kèm `hits_forbidden=yes`. Cũng nên bổ sung trạng thái HALT khi freshness FAIL để ngăn embed dữ liệu stale.


