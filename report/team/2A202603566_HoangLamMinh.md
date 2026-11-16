# Báo cáo cá nhân — 2A202603566_HoangLamMinh

## 1. Thông tin thành viên

- Họ tên: Hoàng Lam Minh
- MSSV: 2A202603566
- Email: <!-- TODO: tự điền -->
- GitHub: <!-- TODO: tự điền -->

## 2. Phần việc phụ trách

- Shape B `single` (một ReAct agent): `src/lab8/shapes/single.py` (TODO CP2, TODO CP3).
- Test cho shape single: `src/tests/test_single_react.py`.
- Mục mô tả và phân tích Shape B trong `report/team/group_report.md`.

## 3. Đóng góp cụ thể

**Commit chính:** `5dfe685` — `feat(single): implement Shape B ReAct agent with read-only tools` (nhánh `2A202603566-HoangLamMinh`).

**Thiết kế Shape B:**

1. Đọc claim bằng `get_claim_view` (qua `runtime.dispatch`, có tính tool call vào `Budget`), không dùng `SELECT *`. Claim không tồn tại → `failed / unknown_claim`. Thiếu `order_id` → `needs_info`, không đoán mã đơn từ ticket.
2. Vòng `lab8.runtime.run_react(..., max_steps=8)`. `decide()` là agent và chỉ được chọn ba tool đọc theo thứ tự `lookup_order` → `lookup_refund_history` → `compute_compensation` → `final`. Danh sách tool của agent không có `issue_refund`. Nếu có, `dispatch` cũng sẽ ném `ToolRejected`.
3. Mỗi bước ReAct gọi `mock_llm.complete(purpose="react")`, sau vòng lặp có một lần `draft`. Nhờ vậy mọi model call đều được cộng vào `Budget`, và chi phí so được với các shape khác.
4. `compute_compensation` luôn nhận **order row từ `lookup_order`**, không nhận số ngày trễ khách tự viết. Khi ticket khai khác DB (ví dụ C-002 ghi "trễ 5 ngày" trong khi DB ghi 2), trace ghi cờ `ticket_mismatch` và vẫn dùng DB. Policy cũ (`SHIP-LATE-OLD`) bị bỏ qua vì `compute_compensation` chỉ dùng policy hiện hành.
5. Sau vòng lặp, **code của shape** quyết định nhánh dựa trên kết quả tool thật (không dựa trên câu chữ của model):
   - Đơn không thấy hoặc không thuộc khách → `escalated`.
   - Hết bước hoặc thiếu quote → `failed`.
   - quote = 0 → `denied`.
   - quote > 0 → `durability.issue_refund(conn, claim_id, quote_đầy_đủ, crash_at=crash_at)`. Shape không tự trừ phần đã hoàn (T14), host lo trần theo đơn và approval.
6. CP3: `InjectedCrash` được ghi vào trace rồi `raise` lại, không bị nuốt (T12). Các exception khác được chuyển thành `failed / exception:<TênLớp>`, vì `run_claim` không được ném lỗi. Shape chỉ ghi tên lớp lỗi, không ghi message.
7. Trace: mỗi vòng ReAct đúng một event `react_step` (observation của bước trước + action). Nhờ vậy metric `steps` (đếm event) phản ánh đúng số bước. Trace chỉ chứa field đã allowlist: order view, quote, kết quả refund. Không ghi ticket, claim row thô, `parent_secret`, header hay PII.

**Test đã thêm:** `src/tests/test_single_react.py`, 14 test chạy trong cổng host:

- `test_single_ignores_overstated_ticket_delay[11..55]`: C-002 bị `denied` ở cả 5 seed. Swarm trả tiền nhầm ở seed 55 vì tin ticket.
- `test_single_loop_dispatches_only_read_tools`: spy `runtime.dispatch` để chứng minh vòng lặp chỉ gọi tool đọc, không gọi tool nào trong `WRITE_TOOLS`.
- `test_single_counts_model_calls_and_respects_step_cap`: `model_calls = số bước ReAct + 1`, số bước ≤ 8.
- `test_single_step_cap_fails_closed_without_refund`: ép `MAX_STEPS=2`. Kết quả phải là `failed / step_cap`, không ghi ledger, trace vẫn đúng `run_id`.
- `test_single_trace_has_no_secret_pii_or_ticket[...]`: quét trace và `RunResult` để chắc không có `forbid_substrings`, `CANARY_SECRET`, `Bearer` hay nguyên văn ticket.

## 4. Bằng chứng

Chạy trong `src/` trên commit `5dfe685`:

- `python -m pytest -q -m student -o addopts= -k "single and not workflow and not orchestrator and not swarm"` → `62 passed`. Lưu ý: lệnh `-k single` trong CHECKPOINTS cũng khớp tên test `test_public_single_run_cases` của các shape khác, nên phải lọc như trên.
- `python -m pytest -q` → `68 passed` (54 test host + 14 test mới).
- `python -m lab8.check_submission` → `[ OK ] shape single: format đúng, 12/12 claim public pass ở seed 11`.
- Run id ví dụ: `R-single-C-002-s55` (denied, có `ticket_mismatch`), `R-single-C-005-s11` (escalated, trace không có PII), `R-single-C-014-s11` (resolved, proposal 150000, ledger 120000).
- Số liệu benchmark chính thức: lấy từ `src/reports/benchmark.json` của nhóm (xem `report/team/group_report.md`).

## 5. Công cụ AI đã dùng

- Claude Code: đọc repo và contract, đề xuất cài đặt `single.py` và file test.
- Cách tôi tự kiểm tra:
  - Chạy ma trận student của single (62/62), cổng host và `check_submission`.
  - Mở trace của C-005 và C-010 để xác nhận không có SĐT, địa chỉ hay `CANARY_*`.
  - Đối chiếu từng nhánh với bảng output contract trong `docs/contracts.md`.
  - Kiểm tra code không sửa file thước đo nào (`metrics.py`, `durability.py`, `policy.py`, `config/`, `data/`).

## 6. Tự đánh giá

- Shape single đạt 100% pass trên bộ public ở cả 5 seed, variance 0, và qua T11/T12. Ranh giới "model chọn tool đọc, code quyết tiền" được kiểm bằng test riêng.
- Hạn chế: `decide()` hiện là chính sách cố định chạy trên MockLLM, nên variance = 0 không chứng minh một LLM thật sẽ ổn định như vậy. Shape single tốn nhiều model call hơn workflow (khoảng 4–5 call mỗi claim), đổi lại không thêm năng lực nào trên bộ public. Đây là dữ kiện cho ADR chọn shape ít tự chủ nhất.
