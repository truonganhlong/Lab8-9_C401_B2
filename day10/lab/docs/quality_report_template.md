# Quality report — Lab Day 10 (nhóm)

**run_id:** `inject-bad` / `sprint2-good`  
**Ngày:** `2026-04-15`

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước | Sau | Ghi chú |
|--------|-------|-----|---------|
| raw_records | 10 | 10 | Cùng số dòng đầu vào, chỉ khác corruption scenario |
| cleaned_records | 6 | 5 | Run bad giữ thêm 1 chunk refund stale |
| quarantine_records | 4 | 5 | Run good đưa stale refund về quarantine |
| Expectation halt? | Có, nhưng bỏ qua bằng `--skip-validate` | Không | `inject-bad` fail ở `refund_no_stale_14d_window` và `refund_single_active_chunk` |

---

## 2. Before / after retrieval (bắt buộc)

**Artifact:** `artifacts/eval/eval_bad.csv` và `artifacts/eval/eval_good.csv`  
Hai file này là output retrieval được export từ môi trường nhóm có Chroma hoạt động; repo hiện giữ lại chúng như evidence before/after cùng với log clean/expectation.

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước:** `eval_bad.csv` trả về top-1 preview “Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc kể từ xác nhận đơn hàng.”; `contains_expected=yes`, `hits_forbidden=yes`. Điều này cho thấy top-k retrieval vẫn chứa chunk stale nên câu trả lời nhìn có vẻ đúng nhưng context bị nhiễm policy cũ.  
**Sau:** `eval_good.csv` trả về top-1 preview “Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.”; `contains_expected=yes`, `hits_forbidden=no`. Retrieval đã quay về đúng version mới và không còn stale chunk trong top-k.

**Merit (khuyến nghị):** versioning HR — `q_leave_version` (`contains_expected`, `hits_forbidden`, cột `top1_doc_expected`)

**Trước:** `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`  
**Sau:** `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`

Kết quả cho thấy inject được cô lập vào slice refund, trong khi slice HR vẫn giữ đúng policy 2026.

---

## 3. Freshness & monitor

**Artifact:** `artifacts/manifests/manifest_sprint2.json`

Manifest baseline ghi `freshness_check=FAIL` vì `latest_exported_at` của sample là `2026-04-10T08:00:00`, vượt SLA 24 giờ tại thời điểm chạy. Đây là cảnh báo observability hợp lệ cho dữ liệu nguồn stale, không phải lỗi clean/validate/embed. Nhóm giữ nguyên SLA 24 giờ để minh họa rõ sự khác biệt giữa “pipeline chạy thành công” và “snapshot dữ liệu còn mới hay không”.

---

## 4. Corruption inject (Sprint 3)

Nhóm tạo `data/raw/policy_export_inject_bad.csv` làm inject scenario riêng, vẫn giữ `policy_export_dirty.csv` làm baseline chuẩn. File inject này giữ lại chunk refund `14 ngày làm việc` nhưng bỏ marker stale cũ, nhờ đó khi chạy:

```bash
python etl_pipeline.py run --raw data/raw/policy_export_inject_bad.csv --run-id inject-bad --no-refund-fix --skip-validate
```

pipeline bad sẽ để chunk stale đi qua bước clean và làm expectation fail có chủ đích. Log tại `artifacts/logs/run_inject-bad.log` cho thấy:
- `refund_no_stale_14d_window` fail với `violations=1`
- `refund_single_active_chunk` fail với `active_refund_rows=2`

Sau khi chạy lại pipeline chuẩn với `run_id=sprint2-good`, log `artifacts/logs/run_sprint2-good.log` cho thấy refund stale bị loại khỏi publish boundary và eval quay về đúng policy `7 ngày`.

---

## 5. Hạn chế & việc chưa làm

- Freshness hiện mới đo trên watermark `latest_exported_at` của manifest; chưa tách riêng boundary ingest và publish.
- Quality report đang dùng keyword retrieval baseline, chưa mở rộng sang LLM-judge hoặc bộ câu hỏi lớn hơn.
