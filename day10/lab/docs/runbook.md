# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

User hoặc Agent trả lời sai thông tin so với chính sách hiện hành (VD: trả lời “14 ngày” thay vì 7 ngày làm việc).
Hoặc hệ thống nhận được dữ liệu cũ mà không được cảnh báo từ pipeline (freshness SLA bị trễ hoặc fail).

---

## Detection

- Pipeline metric báo `freshness_check=FAIL` hoặc `WARN`.
- Expectation fail (vd `no_stale_refund_window`, `refund_single_active_chunk`).
- Retrieval eval ghi nhận `hits_forbidden=True`.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Tìm file `manifest_<run-id>.json` gần nhất xem `freshness_check` là FAIL/WARN/PASS và `latest_exported_at` là khi nào. Dữ liệu có thể vượt quá SLA_HOURS. |
| 2 | Mở `artifacts/quarantine/*.csv` | Kiểm tra file quarantine tương ứng với `run_id` để xem có bị đưa vào quarantine do `stale_source_marker` hay các error khác ko. |
| 3 | Chạy `python eval_retrieval.py` | Kiểm tra metrics từ `artifacts/eval/` (như trước/sau) để xác nhận tập test có đúng không qua file csv đầu ra. |

---

## Mitigation

- Rerun lại pipeline để cập nhật collection (đảm bảo source data đúng chuẩn).
- Trong trường hợp lỗi dữ liệu diện rộng, tạm rollback lại embedding snapshot (hoặc snapshot folder của chroma_db).
- Nếu dữ liệu snapshot `exported_at` bị quá tải / trễ, report lại với owner `Ingestion`.

---

## Prevention

- Cập nhật thêm expectation (VD: cảnh báo tự động về tuổi đời data khi ingest).
- Gắn cảnh báo SLA Freshness thông qua các kênh Alert Channel báo về teams/slack khi `freshness_check=FAIL`.
