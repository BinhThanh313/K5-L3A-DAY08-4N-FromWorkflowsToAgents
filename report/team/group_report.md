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

### 3.B Shape B — `single` (một ReAct agent) — phụ trách: Hoàng Lam Minh

- **Đọc gì:** `get_claim_view` (chỉ `claim_id, principal_id, order_id, ticket`). Trong vòng ReAct (`run_react`, `max_steps=8`), agent đọc qua `lookup_order`, `lookup_refund_history` và `compute_compensation`. Agent không có tool ghi. `dispatch` cũng chặn `WRITE_TOOLS`.
- **Model quyết gì:** chỉ chọn tool đọc tiếp theo và khi nào dừng (`final`). Mỗi bước có một model call (`purpose="react"`), sau vòng lặp có một lần `draft` để soạn phản hồi. Tất cả đều được cộng vào `Budget`.
- **Code quyết gì:** sau vòng lặp, shape dùng **kết quả tool thật** (không dùng câu chữ của model) để chọn nhánh:
  - Thiếu `order_id` → `needs_info`. Nhánh này nằm trước vòng lặp và không đoán mã đơn từ ticket.
  - Đơn không thấy hoặc không thuộc khách → `escalated`.
  - Hết bước → `failed / step_cap`.
  - quote = 0 → `denied`.
  - quote > 0 → `durability.issue_refund(claim_id, quote đầy đủ, crash_at=crash_at)`. Host kiểm approval, trần theo đơn và idempotency.
- **Chống ticket nói dối:** `compute_compensation` luôn nhận order row từ DB. Nếu ticket khai số ngày trễ khác DB, trace ghi cờ `ticket_mismatch` (không ghi nội dung ticket) và vẫn dùng DB. Đây là chỗ khác với swarm, vốn tin ticket ở seed 55 (T02).
- **Crash:** `InjectedCrash` được ghi trace rồi `raise` lại. Exception khác thành `failed / exception:<TênLớp>`.
- **Trace:** các event là `claim` → `react_step` × N (observation + action, field allowlist) → `quote` → `refund` → `decision`. Mỗi vòng ReAct đúng một event, nên metric `steps` phản ánh số bước thật. Trace không có ticket, claim row thô, `parent_secret`, header hay PII. Test `test_single_trace_has_no_secret_pii_or_ticket` kiểm điều này.

## 4. Kết quả

Bảng so sánh 4 shape trên bộ public (từ `src/reports/benchmark.json`): pass rate, pass^5, seed min–max, variance, cost/success, mean/p95 steps. Kèm crash safety (`src/reports/crash.json`) và, nếu có, kết quả tự kiểm trên `student_drop/`.

...

### 4.B Phân tích hành vi Shape B (`single`)

<!-- TODO (Lam Minh): điền số từ src/reports/benchmark.json chính thức của nhóm (by_shape.single), không chép từ terminal. -->

| Metric (single) | Giá trị từ `benchmark.json` |
| --- | --- |
| pass rate / pass^5 | |
| seed min–max / variance | |
| mean / p95 steps | |
| mean model calls / tool calls | |
| cost per success (mô phỏng, VND) | |

Nhận xét:

- **Step:** mỗi claim có thể trả tiền chạy đủ 4 vòng ReAct (3 tool đọc + `final`). Claim `escalated` dừng sau 2 vòng, `needs_info` dừng trước vòng lặp. Không claim public nào chạm trần 8 bước.
- **Token / cost:** so với workflow, single tốn thêm một model call cho mỗi bước ReAct. Phần chi phí thêm này không mang lại thêm claim pass nào trên bộ public, vì số tiền vẫn do `compute_compensation` tính.
- **Variance:** vì `decide()` không bao giờ lấy số từ ticket, kết quả không phụ thuộc seed (khác swarm ở T02 seed 55). Lưu ý giới hạn: MockLLM chỉ sinh text và token theo seed, không ảnh hưởng lựa chọn tool. Với LLM thật, việc chọn tool có thể dao động, và trần 8 bước cùng nhánh `failed / step_cap` là lớp bảo vệ.

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
