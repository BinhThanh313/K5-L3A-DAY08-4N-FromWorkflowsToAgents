# Research card

File này nằm cố định ở `report/team/research_card.md`; `python -m lab8.check_submission` đọc cột "Nhóm điền" bằng máy.

Phần này nối tiếp phần mini research ở buổi lý thuyết (slide 62–63). Chọn **một** câu R1–R4, điền 6 dòng, rồi biến nó thành **một ablation** chạy được trong lab: giữ nguyên model (MockLLM), tool, seed, corpus và giá, chỉ thay đổi một biến.

| Câu | Nguồn (thư viện tham khảo, slide 68–69) | Câu hỏi gợi ý cho lab |
| --- | --- | --- |
| R1 · Scaling | Google Research 2026, *Towards a Science of Scaling Agent Systems* | Task claims chủ yếu tuần tự hay song song được? Dự đoán thêm investigator sẽ làm pass rate và cost/success thay đổi thế nào. |
| R2 · Failure taxonomy | MAST, NeurIPS 2025, *Why Do Multi-Agent LLM Systems Fail?* | Các trial fail của swarm (seed 55) thuộc nhóm nào: lỗi giao việc, lệch mục tiêu, hay lỗi kiểm chứng? |
| R3 · Architecture choices | AgentArch, AACL 2026 | Nếu đổi topology mà giữ nguyên mọi thứ khác, biến nào phải cố định để kết luận còn đúng? |
| R4 · Evaluation confounders | EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* | Làm sao chứng minh chênh lệch giữa A/B/C/D là do topology, không phải do prompt, tool hay seed? |

## Phiếu của nhóm

Điền vào cột cuối. Không xóa hai cột đầu.

| Field | Hướng dẫn | Nhóm điền |
| --- | --- | --- |
| CÂU | R1 / R2 / R3 / R4 | R4 |
| SOURCE | Paper/blog, năm, URL, loại bằng chứng (benchmark, production report, blog kinh nghiệm) | EMNLP 2026, *Rethinking Evaluation of MAS Efficiency* (thư viện tham khảo bài giảng, slide 68–69; nghiên cứu đánh giá hiệu quả hệ đa tác tử) |
| CLAIM | Một kết luận gốc của nguồn. Không viết rộng hơn phạm vi paper | So sánh hiệu quả giữa các kiến trúc chỉ công bằng khi giữ cố định các biến nhiễu (prompt, tool, seed, corpus, bảng giá) và chỉ thay đổi topology |
| BOUNDARY | Model / dataset / setup giới hạn kết luận đó | Phụ thuộc vào bộ task, bộ công cụ đọc/ghi và hàm tính chi phí cụ thể của môi trường đánh giá (trong lab là `MockLLM` và 14 claim public) |
| HYPOTHESIS | Nếu đổi shape A→B (hoặc bật/tắt một thành phần), metric X sẽ đổi theo hướng nào, khoảng bao nhiêu | Khi giữ nguyên corpus/seed/bảng giá và tăng mức tự chủ từ `workflow` → `single` → `orchestrator` → `swarm`, `cost/success` sẽ tăng do thêm model call, còn `pass_rate` không tăng thêm (thậm chí giảm nếu agent tin ticket như `swarm`) |
| METHOD | Biến giữ cố định: model, tools, seed, corpus, giá, evaluator. Biến duy nhất được thay đổi: … | Biến giữ cố định: `MockLLM`, cùng bộ read tools + host `durability.issue_refund`, seed `11,22,33,44,55`, `data/public_claims.jsonl`, bảng giá VND mô phỏng. Biến duy nhất thay đổi: `shape` (`workflow`, `single`, `orchestrator`, `swarm`) |
| FAILURE / METRIC | Một lỗi dự kiến và cách đo nó: pass rate, pass^5, variance, cost/success, leak, bypass | Lỗi dự kiến: `swarm` tin số delay tự khai trong ticket ở `seed % 5 == 0` (`status resolved != denied`). Đo bằng `pass_rate`, `pass^5`, `seed_variance`, `cost_per_success_vnd` trong `src/reports/benchmark.json` |

## Kết quả ablation

- Lệnh đã chạy (copy nguyên văn): `python -m lab8.benchmark --shapes workflow,single,orchestrator,swarm --cases data/public_claims.jsonl --seeds 11,22,33,44,55 --out reports/benchmark.json`
- Run id / file report: `R-workflow-C-001-s11` (và 280 trial trong `src/reports/benchmark.json`)
- Kết quả so với hypothesis (đúng / sai / không kết luận được), kèm số liệu: Đúng với giả thuyết — `workflow` đạt `pass_rate=1.000`, `pass^5=1.000`, `var=0.000000`, `cost/success=27.86 VND`; `single` đạt `1.000` (`39.14 VND`); `orchestrator` đạt `1.000` (`47.28 VND`); `swarm` giảm xuống `pass_rate=0.986`, `pass^5=0.929`, `var=0.000816`, `cost/success=49.57 VND` (1 lỗi `status resolved != denied`).
- Giới hạn: MockLLM không phải model sản xuất, latency là `unavailable`.

Muốn đi xa hơn một ablation để lấy điểm cộng thì xem [docs/bonus_topics.md](../../docs/bonus_topics.md).
