# Team

## Thông tin nhóm

- Tên nhóm: 4N
- Lớp: L3A
- Repo nhóm: `K5-L3A-DAY08-4N-FromWorkflowsToAgents` (xem [SUBMISSION.md](SUBMISSION.md))
- URL repo, nhánh nộp, commit chốt: `https://github.com/BinhThanh313/K5-L3A-DAY08-4N-FromWorkflowsToAgents`, nhánh `main`, commit `d5803cd`

## Thành viên

| STT | Họ tên | MSSV | Email | GitHub | Vai trò |
| --: | ------ | ---- | ----- | ------ | ------- |
|   1 | Dương Thanh Bình | 2A202603531 | 26ai.binhdt2@vinuni.edu.vn | BinhThanh313 | Nhóm trưởng: Shape A (`workflow`) + integration |
|   2 | Hoàng Lam Minh | 2A202603566 | 26ai.minhhl2@vinuni.edu.vn | laminh-h | Shape B (`single`: ReAct agent) |
|   3 | Nguyễn Thị Hồng Hạnh | 2A202603537 | 26ai.hanhnth2@vinuni.edu.vn | hanhshi96 | Shape C (`orchestrator` + handoff/trace) |
|   4 | Nguyễn Đức Minh | 2A202603563 | 26ai.minhnd3@vinuni.edu.vn | ducminhnwork | Durability/crash + benchmark & báo cáo |

## Bảng đóng góp

`python -m lab8.check_submission` đọc bảng này bằng máy: giữ nguyên tiêu đề `## Bảng đóng góp` và các cột. Mỗi người một dòng: commit có nội dung và ít nhất một test đã thêm hoặc sửa.

| Người | Vai | Commit | Test đã thêm hoặc sửa |
| --- | --- | --- | --- |
| Dương Thanh Bình | Nhóm trưởng: workflow + integration | `6172aa4` | `src/tests/test_workflow_extra.py` (`test_workflow_unknown_claim_returns_failed_with_valid_trace`, `test_workflow_deterministic_and_zero_model_calls`) |
| Hoàng Lam Minh | Shape B: single | `5dfe685` | `src/tests/test_single_react.py` (`test_single_ignores_overstated_ticket_delay`, `test_single_loop_dispatches_only_read_tools`, `test_single_counts_model_calls_and_respects_step_cap`, `test_single_step_cap_fails_closed_without_refund`, `test_single_trace_has_no_secret_pii_or_ticket`) |
| Nguyễn Thị Hồng Hạnh | Shape C: orchestrator + handoff | `437f797` | `src/tests/test_orchestrator_handoff.py` (`test_orchestrator_handoff_is_allowlisted_and_readonly`, `test_orchestrator_trace_has_no_secret_or_pii`) |
| Nguyễn Đức Minh | Durability + evaluation | `04ba675` | `src/tests/test_tv4_crash_resume.py` (`test_crash_then_resume_gives_one_refund`) |

## Báo cáo cá nhân

Mỗi thành viên đã hoàn thiện báo cáo cá nhân trong `report/team/`:

- [2A202603531_DuongThanhBinh.md](report/team/2A202603531_DuongThanhBinh.md)
- [2A202603566_HoangLamMinh.md](report/team/2A202603566_HoangLamMinh.md)
- [2A202603537_NguyenThiHongHanh.md](report/team/2A202603537_NguyenThiHongHanh.md)
- [2A202603563_NguyenDucMinh.md](report/team/2A202603563_NguyenDucMinh.md)
