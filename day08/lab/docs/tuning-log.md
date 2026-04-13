# Tuning Log — RAG Pipeline (Day 08 Lab)

> Nguồn tổng hợp: `results/ab_comparison.csv`, `results/scorecard_baseline.md`, `results/scorecard_variant.md`
> A/B Rule: Chỉ đổi **một biến** trong mỗi lần thử.

---

## Baseline (Sprint 2)

**Ngày:** 2026-04-13  
**Config:**
```python
retrieval_mode = "dense"
chunk_size = 400
overlap = 80
top_k_search = 10
top_k_select = 3
use_rerank = False
llm_model = "gpt-5.4-mini"
```

**Scorecard Baseline:**
| Metric | Average Score |
|--------|--------------|
| Faithfulness | 4.50/5 |
| Answer Relevance | 4.90/5 |
| Context Recall | 5.00/5 |
| Completeness | 4.40/5 |

**Câu hỏi yếu nhất (điểm thấp):**
- `q06` (Escalation P1): điểm completeness chỉ `1/5`. Hệ thống có retrieve đúng evidence nhưng answer lại đi sâu vào quyền tạm thời/audit thay vì cơ chế auto-escalate sau 10 phút.
- `q10` (VIP urgent refund): faithfulness chỉ `1/5`. Model nói "không biết" nhưng vẫn thêm chi tiết không có trong context về trường hợp VIP/khẩn cấp.
- `q02` và `q08`: không sai trọng tâm, nhưng còn thiếu nuance như `7 ngày làm việc` hoặc điều kiện `Team Lead approval`.

**Giả thuyết nguyên nhân (Error Tree):**
- [ ] Indexing: Chunking cắt giữa điều khoản
- [ ] Indexing: Metadata thiếu effective_date
- [ ] Retrieval: Dense bỏ lỡ exact keyword / alias
- [ ] Retrieval: Top-k quá ít → thiếu evidence
- [x] Retrieval selection/ranking: đã lấy đúng source nhưng chưa ưu tiên đúng chunk quan trọng nhất vào prompt
- [x] Generation: Prompt chưa ép model abstain đủ chặt khi câu hỏi hỏi về ngoại lệ không có trong docs
- [ ] Generation: Context quá dài → lost in the middle

**Nhận định từ baseline:**
Baseline không gặp vấn đề recall, vì `Context Recall = 5.0/5`. Điểm yếu chủ yếu nằm ở **chọn evidence** và **kỷ luật grounding khi context không trả lời trực tiếp biến thể của câu hỏi**.

---

## Variant 1 (Sprint 3)

**Ngày:** 2026-04-13  
**Biến thay đổi:** bật rerank bằng `jina-reranker-v3`  
**Lý do chọn biến này:**
Evidence từ baseline cho thấy recall đã đủ tốt, nên đổi sang hybrid chưa chắc giải quyết được bài toán chính. `q06` là tín hiệu mạnh nhất: tài liệu đúng đã được retrieve, nhưng chunk được đưa vào prompt chưa phải chunk quan trọng nhất, vì vậy rerank là biến hợp lý nhất để thử trước.

**Config thay đổi:**
```python
retrieval_mode = "dense"
chunk_size = 400
overlap = 80
top_k_search = 10
top_k_select = 3
use_rerank = True
reranker_model = "jina-reranker-v3"
# Các tham số còn lại giữ nguyên như baseline
```

**Scorecard Variant 1:**
| Metric | Baseline | Variant 1 | Delta |
|--------|----------|-----------|-------|
| Faithfulness | 4.50/5 | 4.50/5 | +0.00 |
| Answer Relevance | 4.90/5 | 4.80/5 | -0.10 |
| Context Recall | 5.00/5 | 5.00/5 | +0.00 |
| Completeness | 4.40/5 | 4.70/5 | +0.30 |

**Nhận xét:**
- Cải thiện lớn nhất là `q06`: completeness tăng từ `1/5` lên `5/5`. Đây là bằng chứng mạnh rằng rerank giúp đưa đúng evidence quan trọng lên trên trước khi generate.
- `q02` tốt hơn về faithfulness (`4 -> 5`), nhưng completeness giảm nhẹ (`5 -> 4`) do answer nói `7 ngày` thay vì `7 ngày làm việc`.
- `q07` giảm nhẹ về faithfulness (`5 -> 4`) vì answer vẫn thêm filename/path cụ thể không được support trực tiếp từ retrieved context.
- `q10` vẫn là câu khó nhất. Rerank không giải quyết được lỗi hallucination khi người dùng hỏi một trường hợp đặc biệt không có trong docs; relevance còn giảm nhẹ (`5 -> 4`).

**Kết luận:**
Variant 1 **tốt hơn baseline ở mức nhẹ nhưng có ý nghĩa**, vì nó sửa được lỗi nghiêm trọng nhất của baseline (`q06`) và tăng trung bình completeness từ `4.40` lên `4.70` mà không làm giảm faithfulness tổng thể. Tuy vậy, rerank **không xử lý được triệt để lỗi abstain/hallucination**, nên bước tối ưu tiếp theo nên tập trung vào prompt grounding hoặc answer guardrail hơn là tiếp tục tăng recall.

---

## Variant 2 (nếu có thời gian)

**Biến thay đổi:** Chưa chạy  
**Config:**
```python
# Không có dữ liệu Variant 2 trong results/
```

**Scorecard Variant 2:**
| Metric | Baseline | Variant 1 | Variant 2 | Best |
|--------|----------|-----------|-----------|------|
| Faithfulness | 4.50 | 4.50 | N/A | Baseline = Variant 1 |
| Answer Relevance | 4.90 | 4.80 | N/A | Baseline |
| Context Recall | 5.00 | 5.00 | N/A | Baseline = Variant 1 |
| Completeness | 4.40 | 4.70 | N/A | Variant 1 |

Lý do chưa chạy thêm variant: để giữ đúng A/B rule và tránh trộn nhiều thay đổi khi chưa xử lý xong failure mode chính là grounding ở generation.

---

## Tóm tắt học được

1. **Lỗi phổ biến nhất trong pipeline này là gì?**
   > Lỗi phổ biến nhất là answer chưa đủ grounded khi câu hỏi hỏi tới một ngoại lệ hoặc tình huống không được mô tả trực tiếp trong tài liệu, điển hình là `q10`.

2. **Biến nào có tác động lớn nhất tới chất lượng?**
   > Trong các biến đã thử, rerank tác động lớn nhất tới chất lượng trả lời vì nó cải thiện rõ khâu chọn evidence, đặc biệt ở `q06`, dù không làm tăng recall.

3. **Nếu có thêm 1 giờ, nhóm sẽ thử gì tiếp theo?**
   > Thử prompt/guardrail để ép abstain chặt hơn, ví dụ yêu cầu model chỉ trả lời các ý có citation tương ứng và cấm suy luận về trường hợp đặc biệt nếu context không nêu rõ. Sau đó mới cân nhắc tune `top_k_select` hoặc thêm answer verification step.
