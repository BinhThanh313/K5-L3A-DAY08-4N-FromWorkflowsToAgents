# Báo cáo cá nhân — <2A202603531>_<Dương Thanh Bình>

Đổi tên file theo quy tắc MSSV_HoTen.md.

Ví dụ: 20241234_NguyenVanAn.md.

## 1. Thông tin thành viên

- Họ tên: Dương Thanh Bình
- MSSV: 2A202603531
- Email: binhdt1303@gmail.com
- GitHub: BinhThanh313

## 2. Phần việc phụ trách

- **Vai trò:** Nhóm trưởng — Shape A (`workflow`) + Integration & Nộp bài.
- **Module phụ trách:**
  - `src/lab8/shapes/workflow.py` (`TODO CP2`, `TODO CP3`).
  - `src/tests/test_workflow_extra.py` (test bổ sung cho Shape A).
  - Quản lý repo nhóm, merge code các thành viên, giữ CI / cổng host xanh, tổng hợp `TEAM.md` và chạy `lab8.check_submission` trước khi nộp.

## 3. Đóng góp cụ thể

- **Commit chính (hash):** `6172aa4` (code & test Shape A trên nhánh `2A202603531-DuongThanhBinh`), `398a60c` (tích hợp 4 nhánh trên `merge`)
- **Cài đặt Shape A (`src/lab8/shapes/workflow.py`):**
  - Đọc claim qua `get_claim_view` (không dùng `SELECT *`); trả `failed` (`unknown_claim`) nếu không tồn tại.
  - Xử lý đầy đủ các nhánh nghiệp vụ: thiếu `order_id` → `needs_info`; `lookup_order` không thấy hoặc ngoài phạm vi principal → `escalated`; quote ≤ 0 → `denied` (không gọi `issue_refund`).
  - Tính tiền hoàn bằng `compute_compensation` dựa trên order row (bỏ qua thông tin delay tự khai trong ticket; không gọi `MockLLM` để quyết định số tiền, `model_calls = 0`).
  - Truyền **quote đầy đủ** và `crash_at` vào `durability.issue_refund(conn, claim_id, proposal, crash_at=crash_at)` để host kiểm tra approval, trần theo đơn và idempotency; flush trace trước khi re-raise `InjectedCrash` (không nuốt exception).
  - Ghi trace theo đúng `Trace contract v1` bằng `lab8.trace.Trace` và `lab8.shapes.common.finish()` trên mọi nhánh return, chỉ log các field đã qua allowlist (`ORDER_FIELDS`).
- **Tích hợp nhánh & Kiểm thử hợp nhất (Integration):**
  - Tạo nhánh `merge` từ `main`, gộp đầy đủ 4 nhánh thành viên (`2A202603531-DuongThanhBinh`, `2A202603566-HoangLamMinh`, `Hanh_03537/orchestrator`, `tv4-duc-minh`), đảm bảo không conflict code và không sửa các file cấm.
  - Chạy lại toàn bộ test hợp nhất, `crash_test` (`20/20`), `benchmark` (4 shape × 5 seed), hoàn thiện `TEAM.md`, `group_report.md`, `decision_adr.md`, `research_card.md` và đưa `lab8.check_submission` từ `11 FAIL` về `0 FAIL`.
- **Test đã thêm (`src/tests/test_workflow_extra.py`):**
  - `test_workflow_unknown_claim_returns_failed_with_valid_trace`: kiểm tra claim không tồn tại trả `status="failed"`, `proposal_amount=0`, `error_reason="unknown_claim"` và ghi file trace hợp lệ khớp `run_id`.
  - `test_workflow_deterministic_and_zero_model_calls`: kiểm tra claim `C-001` trên seed `11` và `55` cho kết quả nhất quán (`status="resolved"`, `model_calls=0`, `committed_audit_count=1`) và trace không chứa `CANARY_SECRET`.

## 4. Bằng chứng

- **Repo & nhánh:** `https://github.com/BinhThanh313/K5-Day8-From-Workflows-to-Agents` (commit `6172aa4`, hợp nhất tại `398a60c`)
- **Kết quả chạy test hợp nhất cả 4 thành viên (trong thư mục `src/`):**
  - `python -m lab8.smoke` → `lab8.smoke pass`
  - `python -m pytest -q tests/test_workflow_extra.py` → `2 passed`
  - `python -m pytest -q` (cổng host + test nhóm thêm) → `75 passed, 248 deselected`
  - `python -m pytest -q -m student -o addopts=` (toàn bộ 4 shape) → `248 passed, 75 deselected`
  - `python -m lab8.crash_test --count 20 --out reports/crash.json` → `crash_safety 20/20`
  - `python -m lab8.check_submission` → `0 FAIL`
- **Run ID / trích đoạn report:** `R-workflow-C-001-s11` trong `src/reports/benchmark.json` (`workflow`: `pass_rate=1.000`, `pass^5=1.000`, `seed_variance=0.000000`, `cost_per_success_vnd=27.86`).

## 5. Công cụ AI đã dùng

- **Công cụ:** Trợ lý AI trong IDE (Antigravity).
- **Mục đích sử dụng:**
  - Đọc cấu trúc repo, phân tích tài liệu đề bài và lập kế hoạch chia việc cho 4 thành viên (`WORK_PLAN.md`).
  - Hỗ trợ cài đặt `src/lab8/shapes/workflow.py`, viết test `src/tests/test_workflow_extra.py` và soạn nháp báo cáo cá nhân.
- **Cách tự kiểm tra:**
  - Đối chiếu code với `docs/business_rules.md` và `docs/contracts.md` (không sửa các file cấm như `metrics.py`, `durability.py`, `policy.py`, `config/`, `data/`).
  - Tự chạy lại toàn bộ lệnh kiểm thử `lab8.smoke`, `pytest -q`, `pytest -q tests/test_workflow_extra.py`, và `pytest -q -m student -o addopts= -k workflow` để xác nhận tất cả đều pass.

## 6. Tự đánh giá

- Hoàn thành đúng tiến độ các phần việc của Nhóm trưởng ở giai đoạn CP1–CP3 cho Shape A (`workflow`): pass 62/62 test student, giữ cổng host xanh và thêm test kiểm tra edge-case riêng.
- Cần tiếp tục phối hợp cùng các thành viên còn lại ở giai đoạn merge, chạy `crash_test` (`20/20`), `benchmark`, hoàn thiện `TEAM.md` và kiểm tra `lab8.check_submission` (`0 FAIL`) trước khi nộp.
