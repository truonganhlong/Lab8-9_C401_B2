# Báo Cáo Cá Nhân — Lab Day 09: Multi-Agent Orchestration

**Họ và tên:** Trương Anh Long  
**Vai trò trong nhóm:** MCP Owner  
**Ngày nộp:** 14/04/2026  
**Độ dài yêu cầu:** 500–800 từ

---

## 1. Tôi phụ trách phần nào? (100–150 từ)

Tôi phụ trách Sprint 3 — implement MCP interface cho hệ thống multi-agent.

**Module/file tôi chịu trách nhiệm:**
- File chính: `mcp_server.py`
- Functions tôi implement: `tool_search_kb()`, `tool_get_ticket_info()`, `dispatch_tool()`, `list_tools()`

Cụ thể, tôi implement hai MCP tools theo đúng interface của đề bài: `search_kb` kết nối ChromaDB thật thông qua `workers/retrieval.py`, và `get_ticket_info` trả về mock data với 3 tickets (`P1-LATEST`, `IT-1234`, `IT-0001`). Dispatch layer `dispatch_tool()` đóng vai trò router thống nhất — bất kỳ worker nào cũng chỉ cần gọi `dispatch_tool(tool_name, input)` thay vì hard-code từng API.

**Cách công việc của tôi kết nối với phần của thành viên khác:**

`mcp_server.py` là dependency của `workers/policy_tool.py` — policy worker import `dispatch_tool` để gọi `search_kb` và `get_ticket_info` thay vì truy cập ChromaDB trực tiếp. Nếu MCP server chưa xong, policy worker không thể hoạt động đúng.

**Bằng chứng:**
- File `mcp_server.py` với 2 tools implement đầy đủ
- Trace `run_20260414_194509.json` có `mcp_tools_used` với 3 lần gọi MCP thành công

---

## 2. Tôi đã ra một quyết định kỹ thuật gì? (150–200 từ)

**Quyết định:** Dùng Jina Embeddings API (`jina-embeddings-v5-text-small`, dimension 1024) để index và query ChromaDB, thay vì dùng SentenceTransformer local.

Ban đầu ChromaDB được index bằng một model khác có dimension 512. Khi `search_kb` query với SentenceTransformer (`all-MiniLM-L6-v2`, dimension 384), ChromaDB trả lỗi dimension mismatch. Tôi có 3 lựa chọn: (1) dùng SentenceTransformer và index lại, (2) dùng OpenAI embeddings, (3) dùng Jina API theo cấu hình sẵn có trong `workers/retrieval.py`.

Tôi chọn Jina vì cùng quyết định vói các thành viên trong team. `retrieval.py` đã implement sẵn `embed_jina()` với `JINA_API_KEY` từ `.env` ở các sprint trước — nhất quán với thiết kế của cả hệ thống. Tôi xóa collection cũ, index lại toàn bộ 5 file `.txt` với Jina (`task: retrieval.passage`), và query cũng dùng Jina (`task: retrieval.query`).

**Trade-off đã chấp nhận:** Jina API call có latency (~1-2s/request) so với local embedding (~50ms). Tuy nhiên, độ nhất quán giữa index và query quan trọng hơn tốc độ trong context lab này.

**Bằng chứng từ trace:**
```json
{
  "tool": "search_kb",
  "output": {
    "total_found": 3,
    "sources": ["access_control_sop.txt", "sla_p1_2026.txt", "hr_leave_policy.txt"]
  },
  "timestamp": "2026-04-14T19:45:09.991743"
}
```

---

## 3. Tôi đã sửa một lỗi gì? (150–200 từ)

**Lỗi:** `UnicodeDecodeError` khi index docs vào ChromaDB, và sau đó `total_found: 0` do script lọc sai extension.

**Symptom:** Script index chạy xong nhưng báo `Done! 0 docs indexed` — ChromaDB không có data, dẫn đến `search_kb` luôn trả về `total_found: 0`.

**Root cause:** Có 2 lỗi liên tiếp:
1. Script dùng `open(fpath)` không chỉ định encoding → `UnicodeDecodeError` trên Windows (mặc định `cp1252`) khi gặp ký tự UTF-8 trong file tiếng Việt.
2. Script lọc `if not fname.endswith('.md')` nhưng thực tế file trong `./data/docs` đều là `.txt` → không index được file nào.

**Cách sửa:**
- Thêm encoding fallback: thử lần lượt `utf-8`, `utf-8-sig`, `cp1252`, `latin-1`
- Đổi filter từ `.md` sang `.txt`
- Xóa collection cũ (dimension mismatch) và index lại với Jina

**Bằng chứng trước/sau:**
- Trước khi sửa:
```json
Done! 0 docs indexed. Total in DB: 0
→ search_kb: total_found: 0
```
→ search_kb: total_found: 0
- Sau khi sửa:
```json
OK: access_control_sop.txt
OK: hr_leave_policy.txt
OK: it_helpdesk_faq.txt
OK: policy_refund_v4.txt
OK: sla_p1_2026.txt
Done! 5 docs. Total: 5
→ search_kb: total_found: 3, sources: ['access_control_sop.txt', 'sla_p1_2026.txt', 'hr_leave_policy.txt']
```
---

## 4. Tôi tự đánh giá đóng góp của mình (100–150 từ)

**Tôi làm tốt nhất ở điểm nào?**

Thiết kế dispatch layer gọn và đúng interface MCP — `dispatch_tool(tool_name, input)` là single entry point, dễ mở rộng thêm tool mới mà không cần sửa caller. `policy_tool.py` gọi MCP mà không biết bên trong là ChromaDB hay mock data.

**Tôi làm chưa tốt hoặc còn yếu ở điểm nào?**

Tool `check_access_permission` được `policy_tool.py` gọi nhưng không có trong `mcp_server.py`, dẫn đến lỗi `"Tool không tồn tại"` trong trace. Nên implement thêm tool này để pipeline hoàn chỉnh hơn.

**Nhóm phụ thuộc vào tôi ở đâu?**

`policy_tool_worker` bị block hoàn toàn nếu `mcp_server.py` chưa có `dispatch_tool` — vì đây là interface duy nhất policy worker dùng để lấy data.

**Phần tôi phụ thuộc vào thành viên khác:**

Tôi cần `workers/retrieval.py` có `retrieve_dense()` để `tool_search_kb()` hoạt động. Nếu retrieval worker chưa implement, `search_kb` sẽ fallback về mock data.

---

## 5. Nếu có thêm 2 giờ, tôi sẽ làm gì? (50–100 từ)

Tôi sẽ implement thêm tool `check_access_permission` vào `mcp_server.py`, vì trace `run_20260414_194509.json` cho thấy `policy_tool_worker` đang gọi tool này nhưng nhận lỗi `"Tool không tồn tại"`. Kết quả là `mcp_validation: {"can_grant": null, "required_approvers": null}` — thông tin quan trọng cho quy trình cấp quyền khẩn cấp bị thiếu hoàn toàn. Thêm tool này sẽ làm trace đầy đủ và `final_answer` chính xác hơn.

---

*Lưu file này với tên: `reports/individual/[ten_ban].md`*