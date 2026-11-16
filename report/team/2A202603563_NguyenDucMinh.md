# Báo cáo cá nhân — 2A202603563_NguyenDucMinh

## 1. Thông tin thành viên

- Họ tên: Nguyễn Đức Minh
- MSSV: 2A202603563
- Email: ducminh192004@gmail.com
- GitHub: ducminh.n.work@gmail.com

## 2. Phần việc phụ trách

Thành viên 4: kiểm tra crash/idempotency, chạy crash suite và benchmark, điền ADR và research card.

## 3. Đóng góp cụ thể

- Commit chính: `04ba675`
- Test đã thêm: `src/tests/test_tv4_crash_resume.py`. Crash ở 3 điểm (`before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`) rồi chạy lại đều cho đúng 1 dòng ledger, 1 audit `refund_committed`, số tiền 35.000.
- Quyết định: chọn research card R4, chỉ đổi shape, giữ nguyên MockLLM, tool, seed, bảng giá.

## 4. Bằng chứng

- `python -m pytest -q tests/test_tv4_crash_resume.py` cho 3 passed; cổng host 57 passed.

## 5. Công cụ AI đã dùng

Claude: hướng dẫn cài môi trường, giải thích khái niệm, soạn nháp test và báo cáo. Tôi tự chạy lại mọi lệnh trên máy và chỉ ghi số liệu lấy từ file công cụ của lab sinh ra.

## 6. Tự đánh giá

Tôi dựng được môi trường, viết và chạy xanh test crash/resume, và chuẩn bị khung ADR, research card để điền số liệu từ `benchmark.json`. Điều tôi học được là chọn kiến trúc phải dựa vào số đo trên cùng claim, cùng seed, không dựa vào cảm giác rằng multi-agent thì tốt hơn. Hạn chế: bộ public chỉ có 14 claim và dùng MockLLM, nên kết luận chỉ đúng trong phạm vi này.
