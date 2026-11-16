# Báo cáo nhóm — Lab 8 LogiX-VN

Số liệu trong báo cáo được trích từ `src/reports/benchmark.json` và `src/reports/crash.json`.

## 1. Thông tin nhóm

- Tên nhóm: Nhóm Lab 8 (L3A)
- Thành viên: xem `TEAM.md` (Dương Thanh Bình, Hoàng Lam Minh, Nguyễn Thị Hồng Hạnh, Nguyễn Đức Minh)
- Commit chốt (trùng `report/team/decision_adr.md`): `398a60c` *(cập nhật hash cuối sau khi commit báo cáo lên `main`)*

## 2. Mục tiêu và cách tiếp cận

Bài toán yêu cầu xử lý claim hoàn phí vận chuyển cho LogiX-VN trên cùng chữ ký `run_claim(claim_id, *, seed, store_path, crash_at=None) -> RunResult` với 4 kiến trúc có mức tự chủ tăng dần (`workflow`, `single`, `orchestrator`, `swarm`), nhằm đo lường và chọn kiến trúc **ít tự chủ nhất mà vẫn đạt yêu cầu nghiệp vụ, an toàn crash và bảo mật**.

Nhóm phân chia theo 4 vai trò song song (`WORK_PLAN.md`):
- **Dương Thanh Bình (Nhóm trưởng):** Shape A (`workflow`) + tích hợp nhánh, CI và kiểm tra nộp bài.
- **Hoàng Lam Minh:** Shape B (`single` — ReAct agent).
- **Nguyễn Thị Hồng Hạnh:** Shape C (`orchestrator` + bảo mật handoff & trace).
- **Nguyễn Đức Minh:** Kiểm thử durability/crash/idempotency, chạy `crash_test` & `benchmark`, điền `decision_adr.md` & `research_card.md`.

## 3. Giải pháp

Ở cả ba shape nhóm cài đặt, ranh giới **code / model** được giữ chặt theo `docs/architecture.md`: **code** chịu trách nhiệm đọc dữ liệu chuẩn từ DB, tính tiền bằng `compute_compensation`, gọi `durability.issue_refund` với **quote đầy đủ** và `crash_at` (không bắt `InjectedCrash`), còn `MockLLM` không được quyết định số tiền hay ghi ledger.

### 3.A Shape A — `workflow` (luồng tất định) — phụ trách: Dương Thanh Bình

- **Đọc gì:** đọc claim qua `get_claim_view` (không dùng `SELECT *` để không nạp `parent_secret` hay `raw_headers`), sau đó gọi `lookup_order` để lấy đúng 7 trường trong `ORDER_FIELDS` (không có SĐT, địa chỉ, tên hàng, ghi chú nội bộ).
- **Quyết định theo code:**
  - Claim không tồn tại → `failed` (`unknown_claim`).
  - Thiếu `order_id` → `needs_info` (không suy đoán từ text trong ticket).
  - `lookup_order` không hợp lệ / ngoài phạm vi principal → `escalated`.
  - Tính quote bằng `compute_compensation` trên order row (bỏ qua số ngày trễ hay điều khoản khách viết trong ticket; `model_calls = 0`).
  - `proposal <= 0` → `denied` (`proposal_amount=0`, không gọi `issue_refund`).
  - `proposal > 0` → gọi `durability.issue_refund(conn, claim_id, proposal, crash_at=crash_at)` với full quote để host kiểm tra approval, trần theo đơn và idempotency; ghi event `crash` và flush trace trước khi re-raise `InjectedCrash`.
- **Trace:** mọi nhánh `return` đi qua `finish()`, ghi các event `claim` (chỉ `claim_id`, `principal_id`, `order_id`), `lookup_order` (`ORDER_FIELDS`), `quote`, `issue_refund`, `crash` (nếu có) và `decision`.

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

### 3.C Shape C — `orchestrator` + investigator chỉ đọc — phụ trách: Nguyễn Thị Hồng Hạnh

**Luồng xử lý**

| Bước | Ai làm | Đọc gì | Kết quả |
|---|---|---|---|
| 1 | Orchestrator (code) | `get_claim_view`: chỉ `claim_id`, `principal_id`, `order_id`, `ticket` | Không có claim → `failed`; thiếu `order_id` → `needs_info` |
| 2 | Orchestrator (code) | — | Dựng handoff bằng `make_handoff`, `facts={}`, tool: `lookup_order`, `lookup_refund_history` |
| 3 | Investigator (chỉ đọc) | Order view đã lọc (`ORDER_FIELDS`) và tổng tiền đã hoàn của đơn | Đơn không thấy hoặc không thuộc khách → `escalated` |
| 4 | Orchestrator (code) | Order row từ investigator | `compute_compensation` cho ra full quote; quote = 0 → `denied` |
| 5 | Host (`durability.issue_refund`) | Full quote + `crash_at` | Host kiểm approval, ownership và idempotency, trừ phần đã hoàn của đơn; kết quả `resolved` / `denied` / `failed` |

**Ranh giới code / model:** MockLLM chỉ được gọi một lần (`purpose="draft"`) để soạn câu trả lời cho khách. Số tiền do code tính từ order row trong DB. Số ngày trễ khách tự ghi trong ticket không được dùng. Đây là điểm khác chính so với swarm: ở seed chia hết cho 5, swarm tin số trong ticket.

### 3.D Phân tích bảo mật: handoff và trace

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

## 4. Kết quả

### 4.1 Bảng so sánh 4 shape trên bộ public (`src/reports/benchmark.json`)

14 claim public (`C-001`..`C-014`) × 5 seed (`11,22,33,44,55`) = 70 trial mỗi shape:

| Shape | Pass rate | Pass^5 | Seed min–max | Seed variance | Cost/success (VND mô phỏng) | Mean / p95 steps |
| --- | ---: | ---: | --- | ---: | ---: | --- |
| `workflow` | **1.000** | **1.000** | 1.000–1.000 | **0.000000** | **27.86** | 4.43 / 5.0 |
| `single` | 1.000 | 1.000 | 1.000–1.000 | 0.000000 | 39.14 | 7.07 / 8.0 |
| `orchestrator` | 1.000 | 1.000 | 1.000–1.000 | 0.000000 | 47.28 | 5.36 / 6.0 |
| `swarm` (đối chứng) | 0.986 | 0.929 | 0.929–1.000 | 0.000816 | 49.57 | 2.80 / 3.0 |

- **Crash safety (`src/reports/crash.json`):** `20/20` kịch bản crash (`before_effect`, `after_effect_before_checkpoint`, `after_checkpoint`) đều pass, không tạo hơn một refund.
- **Kiểm thử tự động:** `python -m pytest -q` → `75 passed`; `python -m pytest -q -m student -o addopts=` → `248 passed`; `python -m lab8.check_submission` → `0 FAIL`.

### 4.B Phân tích hành vi Shape B (`single`)

| Metric (single) | Giá trị từ `benchmark.json` |
| --- | --- |
| pass rate / pass^5 | `1.000` / `1.000` |
| seed min–max / variance | `1.000–1.000` / `0.000000` |
| mean / p95 steps | `7.07` / `8.0` |
| mean model calls / tool calls | `4.57` / `3.64` |
| cost per success (mô phỏng, VND) | `39.14` |

Nhận xét:
- **Step:** mỗi claim có thể trả tiền chạy đủ 4 vòng ReAct (3 tool đọc + `final`). Claim `escalated` dừng sau 2 vòng, `needs_info` dừng trước vòng lặp. Không claim public nào chạm trần 8 bước.
- **Token / cost:** so với workflow, single tốn thêm một model call cho mỗi bước ReAct. Phần chi phí thêm này không mang lại thêm claim pass nào trên bộ public, vì số tiền vẫn do `compute_compensation` tính.
- **Variance:** vì `decide()` không bao giờ lấy số từ ticket, kết quả không phụ thuộc seed (khác swarm ở T02 seed 55). Lưu ý giới hạn: MockLLM chỉ sinh text và token theo seed, không ảnh hưởng lựa chọn tool. Với LLM thật, việc chọn tool có thể dao động, và trần 8 bước cùng nhánh `failed / step_cap` là lớp bảo vệ.

Quyết định kiến trúc và lý do: xem `report/team/decision_adr.md`. Phiếu nghiên cứu và ablation: xem `report/team/research_card.md`.

## 5. Khó khăn và cách xử lý

- **Rủi ro tin dữ liệu trong ticket (`C-002`, seed 55):** `swarm` tham chiếu bị fail `status resolved != denied` vì tin số ngày trễ viết trong ticket ở `seed % 5 == 0`. Nhóm xử lý bằng cách luôn dùng `order` row từ `lookup_order` làm nguồn sự thật duy nhất cho `compute_compensation`.
- **Truyền full quote cho host (T13, T14):** nếu shape tự trừ phần đã hoàn trước đó của cùng đơn hàng rồi mới gọi `issue_refund`, host sẽ trừ thêm lần nữa. Cả 3 shape đều truyền full quote và để host thực thi trần theo đơn.
- **Đảm bảo trace khi crash (T12):** nếu `issue_refund` ném `InjectedCrash` mà shape chưa flush trace xuống đĩa, grader sẽ fail `missing_trace`. Cả 3 shape đều bắt `InjectedCrash` chỉ để ghi event `crash` + `trace.write()` rồi `raise` tiếp.

## 6. Tổng hợp đóng góp

Chi tiết đóng góp của từng thành viên nằm trong `TEAM.md` (bảng đóng góp) và báo cáo cá nhân `report/team/MSSV_HoTen.md`.

## 7. Hạn chế và hướng cải thiện

- `MockLLM` là mô hình giả lập theo seed, `latency` là `unavailable`, nên chưa phản ánh đầy đủ độ trễ và sai số chọn tool của LLM sản xuất thật.
- Bộ dữ liệu public chỉ gồm 14 claim; kết quả thực tế còn phụ thuộc bộ hidden (40 claim × 5 seed). Nếu bài toán mở rộng sang các claim phi cấu trúc cần nhiều bước suy luận linh hoạt, nhóm có thể chuyển từ `workflow` sang `single` hoặc `orchestrator` mà vẫn giữ nguyên lớp bảo vệ `durability` và `handoff` allowlist.

## 8. Tài liệu tham khảo

- `README.md`, `CHECKPOINTS.md`, `RUBRIC.md`, `SUBMISSION.md`
- `docs/business_rules.md`, `docs/contracts.md`, `docs/architecture.md`, `docs/metrics.md`
- EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* (thư viện tham khảo bài giảng, slide 68–69).
