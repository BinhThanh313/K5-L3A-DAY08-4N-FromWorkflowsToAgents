# Decision record

Điền sau dấu `:` trên cùng dòng, không xóa nhãn. Giảng viên và `python -m lab8.check_submission` (chạy trong `src/`) đọc các dòng này bằng máy. Không đổi tên hay dời file này khỏi `report/team/`.
`Shape chọn` phải là đúng một trong: `workflow`, `single`, `orchestrator`, `swarm`. Đây là shape được chấm cổng hidden.

- Shape chọn: workflow
- Commit hash: 398a60c
- Run id dẫn chứng: R-workflow-C-001-s11 (src/reports/benchmark.json)
- Pass rate / pass^5 / seed min-max / variance: 1.000 / 1.000 / 1.000-1.000 / 0.000000
- Cost per success (mô phỏng, VND): 27.86 VND
- Crash safety: 20/20 (src/reports/crash.json)
- Secret leak / approval bypass: 0 / 0 (70/70 graded_trials ok trong src/reports/benchmark.json; 248/248 student tests passed)

Ví dụ một dòng đã điền: `- Shape chọn: workflow`, `- Crash safety: 20/20 (src/reports/crash.json)`.

## Vì sao shape này

Trên cùng 14 claim public × 5 seed (`11,22,33,44,55`, tổng 70 trial mỗi shape trong `src/reports/benchmark.json`), cả ba shape nhóm cài đặt (`workflow`, `single`, `orchestrator`) đều đạt `pass_rate = 1.000`, `pass^5 = 1.000`, `seed_variance = 0.000000`, trong khi `swarm` tham chiếu chỉ đạt `pass_rate = 0.986`, `pass^5 = 0.929` (fail `C-002` ở seed 55 do tin số ngày trễ trong ticket). Nhóm chọn `workflow` vì đây là kiến trúc ít tự chủ nhất mà vẫn hoàn thành 100% nghiệp vụ với chi phí mô phỏng thấp nhất (`27.86 VND/success`, `0` model call, `4.43` bước trung bình so với `39.14 VND` của `single` và `47.28 VND` của `orchestrator`).

## Có đáng đổi lấy multi-agent không

Không đáng đổi trong bài toán này: `single` và `orchestrator` chỉ ngang bằng `workflow` về độ chính xác nhưng tốn chi phí hơn lần lượt ~40% và ~70%, còn `swarm` vừa đắt nhất (`49.57 VND/success`) vừa kém ổn định hơn (`seed_variance = 0.000816`). Quyết định chỉ đảo lại khi quy trình nghiệp vụ không còn cố định (cần truy vấn nhiều nguồn không xác định trước, xử lý ngoại lệ mở rộng hoặc cần phân rã tác vụ phức tạp vượt quá luồng tất định).

## Trace và bằng chứng

Trace của `workflow` ghi các event: `claim` (chỉ `claim_id`, `principal_id`, `order_id`), `lookup_order` (chỉ `ORDER_FIELDS` đã allowlist), `quote`, `issue_refund`, `crash` (nếu có) và `decision`. Nhóm cố ý **không** ghi `parent_secret`, `raw_headers`, nguyên văn `ticket`, SĐT, địa chỉ, tên hàng hay ghi chú nội bộ vào trace hoặc stdout. Nếu trace nằm ngoài thư mục mà scanner quét được (`src/traces/` và tiền tố `<run_id>*`), grader fail-closed (`missing_trace` / `invalid_trace`) vì không thể chứng minh log sạch.

## Giới hạn

MockLLM không phải model sản xuất. Latency để `unavailable`. Kết quả `1.000` mới đo trên 14 claim public; không sửa JSON cho đủ ngưỡng.
