# Báo cáo nhóm — Lab 8 LogiX-VN

Tối đa 5 trang ngoài phụ lục. Số liệu lấy từ file trong `src/reports/`, không chép tay từ terminal.

## 1. Thông tin nhóm

- Tên nhóm:
- Thành viên: xem `TEAM.md`
- Commit chốt (trùng `report/team/decision_adr.md`):

## 2. Mục tiêu và cách tiếp cận

Nhóm hiểu bài toán thế nào, chia việc ra sao, thứ tự làm bốn shape.

...

## 3. Giải pháp

Mô tả ba shape nhóm viết (workflow, single, orchestrator): mỗi shape đọc gì, quyết định gì, gọi host ra sao. Chỉ rõ ranh giới code / model (tham khảo [docs/architecture.md](../../docs/architecture.md)), cách handoff được sanitize và trace ghi những event nào.
### 3.3 Shape C — Orchestrator + investigator chỉ đọc (phụ trách: Hạnh)

**Luồng xử lý**

| Bước | Ai làm | Đọc gì | Kết quả |
|---|---|---|---|
| 1 | Orchestrator (code) | `get_claim_view`: chỉ `claim_id`, `principal_id`, `order_id`, `ticket` | Không có claim → `failed`; thiếu `order_id` → `needs_info` |
| 2 | Orchestrator (code) | — | Dựng handoff bằng `make_handoff`, `facts={}`, tool: `lookup_order`, `lookup_refund_history` |
| 3 | Investigator (chỉ đọc) | Order view đã lọc (`ORDER_FIELDS`) và tổng tiền đã hoàn của đơn | Đơn không thấy hoặc không thuộc khách → `escalated` |
| 4 | Orchestrator (code) | Order row từ investigator | `compute_compensation` cho ra full quote; quote = 0 → `denied` |
| 5 | Host (`durability.issue_refund`) | Full quote + `crash_at` | Host kiểm approval, ownership và idempotency, trừ phần đã hoàn của đơn; kết quả `resolved` / `denied` / `failed` |

**Ranh giới code / model:** MockLLM chỉ được gọi một lần (`purpose="draft"`) để soạn câu trả lời cho khách. Số tiền do code tính từ order row trong DB. Số ngày trễ khách tự ghi trong ticket không được dùng. Đây là điểm khác chính so với swarm: ở seed chia hết cho 5, swarm tin số trong ticket.

### 3.4 Phân tích bảo mật: handoff và trace

**1. Allowlist, không phải tóm tắt.** Handoff chỉ có 8 field cố định (`ALLOWED_FIELDS`), và `facts` chỉ nhận 6 key nghiệp vụ (`FACT_KEYS`). Mọi field khác (`parent_secret`, `raw_headers`, lịch sử chat, toàn bộ claim row) bị bỏ ngay trong code, trước khi đến investigator. Sau đó `sanitize()` quét lại một lần nữa: nếu còn sót chuỗi `CANARY_SECRET_`, `parent_secret` hoặc `raw_headers` thì báo lỗi và dừng, không âm thầm cho qua. Nhóm không dùng LLM để "tóm tắt cho gọn", vì một bản tóm tắt vẫn có thể chép lại bí mật.

**2. Tách quyền đọc và ghi (least privilege).**
- `make_handoff` từ chối mọi tool không nằm trong `READONLY_TOOLS`.
- `runtime.dispatch(worker=True)` từ chối `issue_refund` và `append_audit`.
- Chỉ orchestrator gọi `durability.issue_refund`, và host vẫn tự kiểm approval của chính claim đó.

Kết quả là kể cả khi investigator bị điều khiển sai (ví dụ do prompt injection trong ticket), nó cũng không có đường nào để ghi tiền.

**3. Trace là một phần ranh giới bảo mật.** Grader quét `trace_path` và mọi file `<run_id>*` để tìm secret/PII. Shape C chỉ ghi:
- `claim`: `claim_id` và `has_order_id`. Không ghi `ticket`, vì khách có thể tự viết SĐT hay địa chỉ vào đó.
- `handoff`: bản đã sanitize.
- `investigator`: order view đã lọc, cùng số tiền và số lần đã hoàn.
- `quote`, `issue_refund`, `decision`.

Mọi nhánh return đi qua `finish()`, nên luôn có trace đúng contract v1. Nhánh crash ghi event `crash` và flush trace trước khi ném `InjectedCrash` tiếp.

**4. Kiểm chứng bằng test.** File `tests/test_orchestrator_handoff.py` gồm:
- Kiểm tra handoff trong trace thuộc allowlist và không có tool ghi.
- Chạy toàn bộ claim public, rồi kiểm tra không có chuỗi nào trong `forbid_substrings` (canary, header Bearer, SĐT, địa chỉ, tên hàng, ghi chú nội bộ) xuất hiện trong trace.

**5. Đánh đổi của Shape C.**
- **So với workflow:** thêm một tầng handoff và vài tool call, đổi lại tách được quyền đọc và ghi. Đây là lợi thế khi sau này investigator là model thật.
- **So với swarm:** ít model call hơn và kết quả không đổi theo seed, vì không tin số ngày trễ trong ticket.

Số liệu chính thức (pass^5, variance, cost/success, steps) xem bảng ở mục 4, lấy từ `src/reports/benchmark.json`.

...

## 4. Kết quả

Bảng so sánh 4 shape trên bộ public (từ `src/reports/benchmark.json`): pass rate, pass^5, seed min–max, variance, cost/success, mean/p95 steps. Kèm crash safety (`src/reports/crash.json`) và, nếu có, kết quả tự kiểm trên `student_drop/`.

...

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

## 5. Khó khăn và cách xử lý

Lỗi gặp phải (kèm lệnh, output hoặc ảnh chụp) và cách nhóm xử lý.

...

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

## 7. Hạn chế và hướng cải thiện

MockLLM không phải model sản xuất; latency là `unavailable`. Nêu thêm giới hạn của nhóm và điều kiện làm kết luận thay đổi.

...

## 8. Tài liệu tham khảo

...
