# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Lê Thanh Long  
**Mã sinh viên:** 2A202600105  
**Vai trò:** Cleaning & Quality Owner (Sprint 2)  
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**
- `day10/lab/transform/cleaning_rules.py` — Toàn bộ 9 cleaning rule: allowlist `doc_id`, chuẩn hoá `effective_date` (YYYY-MM-DD) và `exported_at` (ISO datetime), quarantine stale/draft marker, deduplication theo nội dung, fix stale refund window "14 ngày → 7 ngày", temporal conflict (`effective_date_after_exported_at`), quarantine HR policy cũ (effective_date < 2026-01-01), sinh `chunk_id` ổn định bằng SHA-256.
- `day10/lab/quality/expectations.py` — Suite 10 expectation phân mức halt/warn: dữ liệu cleaned phải vượt quality gate trước khi sang bước Publish/Embed.

**Kết nối với thành viên khác:**
Tôi nhận raw CSV từ Ingestion Owner (Đỗ Xuân Bằng), output `cleaned_csv` + `quarantine_csv` được Eval Owner (Trương Anh Long) dùng làm đầu vào cho Sprint 3. Monitoring/Docs Owner (Lã Thị Linh) dùng các quarantine reason và metric của tôi để viết runbook.

**Bằng chứng:** log `day10/lab/artifacts/logs/run_sprint2.log` ghi rõ `run_id=sprint2`, `raw_records=10`, `cleaned_records=5`, `quarantine_records=5`, toàn bộ expectation halt đều `OK`.

---

## 2. Một quyết định kỹ thuật

**Dùng `_stable_chunk_id` (SHA-256) thay vì row index để đảm bảo idempotency khi embed/upsert.**

Khi bắt đầu Sprint 2, tôi cân nhắc dùng row index tuần tự làm `chunk_id`. Tuy nhiên khi thảo luận với bước Embed, tôi nhận ra nếu pipeline re-run với dữ liệu clean thay đổi thứ tự hàng, ChromaDB sẽ upsert nhầm record — chunk cũ bị ghi đè bởi chunk mới không liên quan, gây lỗi silent mà log không phát hiện được.

Giải pháp: tôi implement `_stable_chunk_id(doc_id, chunk_text, seq)` dùng SHA-256, lấy 16 ký tự đầu, format `{doc_id}_{seq}_{hash}`. Cùng nội dung luôn sinh cùng `chunk_id` → pipeline upsert nhiều lần mà không phình DB (idempotent). Trade-off: sau khi refund fix thêm tag `[cleaned: stale_refund_window]`, chunk sinh ID mới — đây là đúng ý muốn vì chunk đã fix phải tách khỏi chunk gốc stale để tránh nhầm lẫn khi ChromaDB ghi đè.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Chạy thử lần đầu, chunk refund "14 ngày" vẫn lọt vào `cleaned` dù đã có fix text — expectation `refund_no_stale_14d_window` báo `violations=1` trong khi log cleaning lại cho `cleaned_records=6`.

**Phát hiện:** Tôi trace lại thứ tự rule trong `clean_rows()` và nhận ra stale marker check đang được chạy **sau** khi append record vào danh sách cleaned. Cụ thể: fix text "14 → 7 ngày" chạy trước, nhưng kiểm tra `_contains_stale_source_marker(fixed_text)` lại bị đặt nhầm sau lệnh `cleaned.append(...)`.

**Fix:** Đảo thứ tự — chạy `_contains_stale_source_marker(fixed_text)` **trước** khi append, quarantine ngay nếu có marker stale. Sau fix, `run_id=sprint2` log: `expectation[refund_no_stale_14d_window] OK (halt) :: violations=0`, `cleaned_records=5`, `quarantine_records=5` (5 lý do quarantine: `duplicate_chunk_text`, `stale_source_marker`, `missing_effective_date`, `stale_hr_policy_effective_date`, `unknown_doc_id`).

---

## 4. Bằng chứng trước / sau

`run_id=sprint2` (log: `artifacts/logs/run_sprint2.log`): `raw_records=10` → `cleaned_records=5`, `quarantine_records=5`. Sau đó pipeline embed: `embedding_runtime=provider=jina model=jina-embeddings-v5-text-small`, `embed_upsert count=5 collection=day10_kb`.

Hai dòng từ eval CSV chứng minh cleaning tác động thật lên retrieval:

**eval_bad.csv** (`run_id=inject-bad`) — Before (dirty):
```
q_refund_window → "14 ngày làm việc..." | hits_forbidden=yes
```

**eval_good.csv** (`run_id=sprint2-good`) — After (cleaned):
```
q_refund_window → "7 ngày làm việc..."  | hits_forbidden=no
```

Chunk stale bị quarantine → không được embed → retrieval trả đúng policy.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ thêm rule kiểm tra **min_chunk_length thích nghi theo `doc_id`** thay vì ngưỡng global 8 ký tự: với `policy_refund_v4` yêu cầu tối thiểu 50 ký tự, vì chunk quá ngắn không đủ context cho embedding — tránh trường hợp chunk vượt expectation E4 nhưng vẫn sinh vector kém chất lượng khi query retrieval.
