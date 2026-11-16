# Báo cáo cá nhân — 2A202603537_NguyenThiHongHanh

## 1. Thông tin thành viên

- Họ tên: Nguyễn Thị Hồng Hạnh
- MSSV: 2A202603537
- Email: 26ai.hanhnth2@vinuni.edu.vn
- GitHub: hanhshi96
...

## 2. Phần việc phụ trách

- Thành viên số 3 theo WORK_PLAN.md: Shape C — orchestrator `(src/lab8/shapes/orchestrator.py)`, gồm cả TODO CP2 và TODO CP3.
- Ranh giới bảo mật của handoff và trace: handoff theo allowlist, investigator chỉ có tool đọc, trace không lộ secret/PII.
- Phần phân tích bảo mật, ranh giới handoff và trace trong `report/team/group_report.md`.
...

## 3. Đóng góp cụ thể

#### Commit chính:
<437f797bc1026aadffddcb716ecec6a7c21e9784> trên nhánh `Hanh_03537/orchestrator`
#### Code: 
src/lab8/shapes/orchestrator.py
#### Shape chạy 5 bước:
1. **Đọc claim**: `orchestrator` đọc claim qua `get_claim_view`, không dùng `SELECT *`, nên `parent_secret` và `raw_headers` không bao giờ được nạp vào.
    - Claim không tồn tại → failed.
    - Thiếu `order_id → needs_info`.
2. **Dựng handoff**: handoff được tạo bằng make_handoff.
    - `facts={}`, vì lúc này orchestrator chưa có dữ kiện nào đã kiểm chứng. Không đẩy claim row hay ticket xuống worker.
    - `allowed_tools = ["lookup_order", "lookup_refund_history"]`, chỉ gồm tool đọc.
3. **Điều tra**: `run_readonly_investigators` tra đơn hàng và lịch sử hoàn tiền. Đơn không thấy hoặc không thuộc khách này → escalated.
4. **Tính tiền**: orchestrator tự tính quote bằng `compute_compensation`, dựa trên order row lấy từ DB. Model không quyết định số tiền. MockLLM chỉ dùng một lần để soạn câu trả lời. Quote = 0 → denied, không gọi host.
5. **Hoàn tiền**: gọi `durability.issue_refund(conn, claim_id, <full quote>, crash_at=crash_at)`.
Truyền full quote vì host tự trừ phần đã hoàn trước đó (T13, T14).
Không bắt `InjectedCrash`: chỉ ghi trace rồi ném tiếp, để T12 thấy crash thật

#### Test thêm: 
`**src/tests/test_orchestrator_handoff.py**`

- `test_orchestrator_handoff_is_allowlisted_and_readonly`: kiểm tra handoff ghi trong trace chỉ có field thuộc `ALLOWED_FIELDS` và không có tool ghi `(issue_refund, append_audit)`.
- `test_orchestrator_trace_has_no_secret_or_pii`: chạy toàn bộ claim public, rồi kiểm tra không có chuỗi nào trong `forbid_substrings` xuất hiện trong trace. Danh sách này gồm `CANARY_SECRET_*`, header Bearer, SĐT, địa chỉ, tên hàng và ghi chú nội bộ.

#### Quyết định kỹ thuật:
- Không ghi `ticket` vào `trace`. Ticket là chữ khách tự viết, có thể chứa SĐT hoặc địa chỉ. Trace chỉ ghi `has_order_id`.
- Không cấp `compute_compensation` cho `investigator`. Orchestrator tự tính tiền từ order view. Nhờ vậy việc quyết định tiền nằm ở đúng một chỗ, đúng như docstring đề bài yêu cầu.
- Mọi nhánh return đều đi qua một hàm `done()`. Hàm này gọi `finish()`, nên nhánh nào cũng ghi trace đúng Trace contract v1 và luôn có một event decision.
...
## 4. Bằng chứng

Các lệnh dưới đây chạy trong src/ trên máy cá nhân:
```text
$ python -m pytest -q -m student -o addopts= -k orchestrator
62 passed, 242 deselected

$ python -m pytest -q -o addopts= tests/test_orchestrator_handoff.py
2 passed

$ python -m pytest -q
56 passed, 248 deselected
```

- Run id có dạng `make_run_id("orchestrator", <claim_id>, <seed>)`. Trace ghi các event `claim`, `handoff` (bản đã sanitize), `investigator`, `quote`, `issue_refund` và `decision`.
- Số liệu so sánh chính thức lấy từ `src/reports/benchmark.json`, bảng ở mục 4 của `group_report.md`.
...
## 5. Công cụ AI đã dùng

- **Claude (Anthropic):** dùng để đọc hiểu đề, giải thích `handoff.py`, `runtime.py` và `durability.py`, gợi ý bản cài đặt `orchestrator.py` cùng test ranh giới handoff.
- **Cách tự kiểm tra:**
  - Chạy lại toàn bộ test student của orchestrator (62/62), test của mình (2/2) và cổng host trên máy cá nhân.
  - Đối chiếu từng bước code với docstring đề bài và `docs/contracts.md`, gồm bảng status/proposal_amount và Trace contract v1.
  - Mở file trace để xác nhận không có `CANARY_SECRET_` hay SĐT.
...

## 6. Tự đánh giá

- **Làm tốt:** Shape C đạt đủ test student. Ranh giới đọc/ghi rõ ràng: chỉ orchestrator gọi host, worker không có tool ghi. Có test tự động chứng minh trace không rò rỉ secret/PII, giúp đảm bảo must-pass số 2 (`secret_leak_count = 0`).
- **Hạn chế:** investigator hiện là bộ chạy tool cố định (`run_readonly_investigators`), chưa có model tự suy luận. Vì vậy Shape C chưa thể hiện hết điểm mạnh và điểm yếu của một hệ multi-agent thật. Số liệu chi phí cũng chỉ là mô phỏng từ MockLLM.
- **Học được:** một bản tóm tắt do LLM viết không thay được việc lọc dữ liệu. Ranh giới bảo mật phải được viết bằng code (allowlist) và kiểm chứng bằng test.

