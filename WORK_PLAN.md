# Kế hoạch phân công công việc nhóm 4 người — Lab 8 (LogiX-VN)

## 1. Tổng quan & Nguyên tắc chung

- **Mục tiêu:** Cài đặt 3 shape còn thiếu (`workflow`, `single`, `orchestrator`) trong `src/lab8/shapes/` với cùng chữ ký `run_claim(claim_id, *, seed, store_path, crash_at=None) -> RunResult`, đối chứng với `swarm_reference.py`, chạy crash suite & benchmark, và chọn kiến trúc ít tự chủ nhất mà vẫn đạt yêu cầu.
- **File không được sửa:** `src/lab8/metrics.py`, `src/lab8/durability.py`, `src/lab8/policy.py`, `src/config/*.yaml`, `src/data/*`, và không sửa tay các file JSON trong `src/reports/`.
- **3 điều kiện Must-pass (vi phạm không bù điểm):**
  1. Không refund khi thiếu approval hợp lệ (`approval_bypass_count = 0`).
  2. Không lộ `CANARY_SECRET_*` hay PII trong trace/kết quả (`secret_leak_count = 0`).
  3. Crash không tạo hơn 1 refund (`crash_safety = 20/20`).
- **Quy tắc chung khi code:**
  - Tiền hoàn do `lab8.tools.compute_compensation` tính (không do `MockLLM`).
  - Luôn truyền **full quote** vào `durability.issue_refund(..., crash_at=crash_at)` và không bắt `InjectedCrash`.
  - Mọi nhánh return (kể cả `needs_info`, `escalated`, `denied`) đều phải ghi trace đúng chuẩn `Trace contract v1` (`{"run_id": ..., "events": [...]}`).
  - **Mỗi thành viên** phải có commit kỹ thuật thực chất và **ít nhất 1 test thêm/sửa** trong `src/tests/` (để đạt tiêu chí C8 và pass `python -m lab8.check_submission`).

---

## 2. Phân công chi tiết cho 4 thành viên

### Bình — Nhóm trưởng: Shape A (`workflow`) + Integration & Nộp bài
- **File code phụ trách:**
  - `src/lab8/shapes/workflow.py` (`TODO CP2`, `TODO CP3`).
  - Quản lý repo nhóm (`K5-L3A-DAY08-TenNhom-FromWorkflowsToAgents`), merge PR từ các thành viên, giữ CI và cổng host (`python -m pytest -q`) luôn xanh.
- **Nhiệm vụ kỹ thuật:**
  - Đọc claim qua `get_claim_view` (không `SELECT *`).
  - Xử lý đầy đủ các nhánh: thiếu `order_id` → `needs_info`; `lookup_order` không thấy → `escalated`; quote = 0 → `denied`; quote > 0 → gọi `durability.issue_refund` với full quote + `crash_at`.
  - Chỉ dùng `MockLLM` để soạn text phản hồi (nếu cần), không để model quyết định số tiền.
  - Ghi trace đầy đủ ở mọi nhánh return.
- **Test phụ trách:**
  - Đảm bảo pass 62/62 test student cho `workflow`: `python -m pytest -q -m student -o addopts= -k workflow`.
  - Thêm/sửa ít nhất 1 test trong `src/tests/` cho `workflow` (ví dụ: nhánh `needs_info`, `escalated`, hoặc quote = 0).
- **Tài liệu & Nộp bài:**
  - Điền thông tin nhóm, thành viên và **bảng đóng góp** trong `TEAM.md`.
  - Chốt commit hash, chạy `python -m lab8.check_submission` đạt `0 FAIL`, push và nhắc cả nhóm nộp link trên LMS.
  - Viết báo cáo cá nhân `report/team/<MSSV>_<HoTen>.md`.

---

### Lam Minh — Shape B (`single`: ReAct Agent)
- **File code phụ trách:**
  - `src/lab8/shapes/single.py` (`TODO CP2`, `TODO CP3`).
- **Nhiệm vụ kỹ thuật:**
  - Cài đặt vòng lặp ReAct bằng `lab8.runtime.run_react` với `max_steps=8`.
  - Chỉ cấp các **tool đọc** cho `decide` (tuyệt đối không cấp `issue_refund` trong vòng lặp), đếm số model call vào `Budget`.
  - Đảm bảo `decide()` đối chiếu dữ liệu từ order row, không tin số delay hoặc điều khoản cũ khách hàng viết trong ticket.
  - Sau vòng lặp, code của shape tính/lấy quote hợp lệ và gọi `durability.issue_refund` với **full quote** + `crash_at`.
  - Ghi trace đúng chuẩn ở mọi nhánh return.
- **Test phụ trách:**
  - Đảm bảo pass 62/62 test student cho `single`: `python -m pytest -q -m student -o addopts= -k single`.
  - Thêm/sửa ít nhất 1 test trong `src/tests/` (ví dụ: kiểm tra giới hạn `max_steps=8`, đếm `Budget`, hoặc case khách khai khống delay trong ticket).
- **Tài liệu:**
  - Viết phần mô tả và phân tích hành vi Shape B (step, token, variance) trong `report/team/group_report.md`.
  - Viết báo cáo cá nhân `report/team/<MSSV>_<HoTen>.md`.

---

### Hạnh — Shape C (`orchestrator`) + Bảo mật Handoff & Trace
- **File code phụ trách:**
  - `src/lab8/shapes/orchestrator.py` (`TODO CP2`, `TODO CP3`).
- **Nhiệm vụ kỹ thuật:**
  - Dựng gói handoff bằng `lab8.handoff.make_handoff` theo đúng **allowlist**; tuyệt đối không đưa `parent_secret`, raw headers, hay full claim row vào `facts`.
  - Chạy investigator chỉ đọc qua `lab8.runtime.run_readonly_investigators` (có thể dùng `lookup_refund_history`).
  - Orchestrator tính quote từ order view do investigator trả về và gọi `durability.issue_refund` với **full quote** + `crash_at`.
  - Trace chỉ ghi **sanitized handoff**, không ghi parent context hay PII/secret (`CANARY_SECRET_*`).
- **Test phụ trách:**
  - Đảm bảo pass 62/62 test student cho `orchestrator`: `python -m pytest -q -m student -o addopts= -k orchestrator`.
  - Thêm/sửa ít nhất 1 test trong `src/tests/` kiểm tra ranh giới handoff (allowlist, investigator không có tool ghi, hoặc không rò rỉ secret/PII trong trace).
- **Tài liệu:**
  - Viết phần phân tích bảo mật, ranh giới handoff và trace trong `report/team/group_report.md`.
  - Viết báo cáo cá nhân `report/team/<MSSV>_<HoTen>.md`.

---

### Nguyễn Đức Minh — Durability/Crash + Benchmark & Báo cáo quyết định (ADR / Research Card)
- **Code & Kiểm thử hệ thống phụ trách:**
  - Phối hợp kiểm tra idempotency (T11), crash/resume (T12), per-order cap (T13–T14) và cổng host (`python -m pytest -q`) trên cả 3 shape.
  - Chạy crash suite tạo `src/reports/crash.json`:
    ```bash
    python -m lab8.crash_test --count 20 --out reports/crash.json
    ```
    (Đảm bảo `crash_safety` = `20/20`).
  - Chạy benchmark 4 shape × 5 seed tạo `src/reports/benchmark.json`:
    ```bash
    python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json
    ```
  - Chạy tự kiểm hidden trên `student_drop/` ở CP5 (nếu có).
- **Test phụ trách:**
  - Thêm/sửa ít nhất 1 test trong `src/tests/` về idempotency, crash/resume, approval bypass hoặc kiểm tra metrics.
- **Tài liệu:**
  - Điền `report/team/decision_adr.md` (chọn shape ít tự chủ nhất đạt ngưỡng, ghi đúng `run_id`, commit hash và số liệu từ `benchmark.json`, giữ nguyên các dòng nhãn).
  - Điền `report/team/research_card.md` (cột "Nhóm điền", R1–R4, giới hạn của `MockLLM`).
  - Tổng hợp bảng số liệu benchmark vào `report/team/group_report.md`.
  - Viết báo cáo cá nhân `report/team/<MSSV>_<HoTen>.md`.

---

## 3. Lộ trình phối hợp theo Checkpoint

| Checkpoint | Thời điểm | Việc làm chung / Song song | Kết quả cần đạt |
| --- | --- | --- | --- |
| **CP1** | 15 phút đầu | Cả 4 người cài `.venv` trong `src/`, chạy `python -m lab8.smoke` & `python -m pytest -q`; đọc `docs/business_rules.md` và `docs/contracts.md`. TV1 điền danh sách vào `TEAM.md`. | Smoke pass, host test xanh trên cả 4 máy. |
| **CP2** | 13:45 | **Làm song song (không conflict file):** TV1 code `workflow.py`; TV2 code `single.py`; TV3 code `orchestrator.py`; TV4 viết test durability/approval & chuẩn bị khung báo cáo. | Mỗi shape pass 62/62 test student của mình. |
| **CP3** | 14:30 | Merge 3 shape vào nhánh chính; hoàn thiện `TODO CP3` (`crash_at`); kiểm tra handoff allowlist và T11, T12. | T11, T12 xanh trên cả 3 shape; cổng host vẫn xanh. |
| **CP4** | 15:30 | TV4 chạy `lab8.crash_test` tạo `src/reports/crash.json`; cả nhóm rà soát 3 điều kiện must-pass (approval, secret/PII, crash). | `crash_safety = 20/20`, không lỗi bảo mật/approval. |
| **CP5** | 16:15–17:00 | TV4 chạy `lab8.benchmark` tạo `src/reports/benchmark.json`, điền `decision_adr.md` & `research_card.md`. Cả 4 người hoàn thiện `group_report.md`, `TEAM.md` và `MSSV_HoTen.md`. TV1 chạy `check_submission` và push. | `python -m lab8.check_submission` ra `0 FAIL`; tất cả nộp link LMS. |
