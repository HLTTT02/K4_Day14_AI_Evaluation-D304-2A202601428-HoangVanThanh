# Day 14 — Exercises: AI Evaluation & Benchmarking

## Part 1 — Warm-up

### Exercise 1.1 — RAGAS Metric Thresholds

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Câu trả lời ngắn có diễn giải từ đồng nghĩa nên overlap thấp nhẹ | Có khẳng định giá, chính sách hoặc quyền lợi không có trong nguồn | Chặn phát hành, kiểm tra grounding và citation |
| Answer Relevance | Câu hỏi mơ hồ nhưng câu trả lời vẫn giải quyết một cách hiểu hợp lý | Trả lời sai intent hoặc né câu hỏi hỗ trợ hợp lệ | Sửa routing/prompt và thêm test theo intent |
| Context Recall | Câu đơn giản và một chunk đã đủ bằng chứng cốt lõi | Thiếu điều kiện, ngoại lệ hoặc mốc ngày làm thay đổi kết luận | Điều chỉnh chunking/query expansion/top-k |
| Context Precision | Recall cao nhưng vài chunk nhiễu đứng sau bằng chứng đúng | Nhiễu đứng trước làm generator dùng sai chính sách | Thêm reranker và lọc metadata/version |
| Completeness | Thiếu chi tiết phụ không đổi hành động của khách | Thiếu deadline, phí, ngoại lệ an toàn hoặc bước escalation | Thêm checklist vào prompt và gold answer |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

1. Chấm cùng cặp câu trả lời ở hai condition A/B rồi đảo thứ tự ở B. Giữ nguyên model, prompt, temperature và rubric; chạy nhiều lần. Position bias xuất hiện nếu cùng một answer được chấm cao hơn đáng kể khi đứng đầu.
2. Rubric phải đánh giá đúng/sai và mức bao phủ theo danh sách thông tin bắt buộc, nêu rõ độ dài không phải tiêu chí, đồng thời phạt nội dung thừa không liên quan hoặc không có nguồn.
3. Human labels tạo chuẩn ngoài để đo agreement, tìm vùng judge quá dễ/quá nghiêm và hiệu chỉnh threshold. Nếu không calibrate, bias của model có thể bị hiểu nhầm là chất lượng hệ thống.

### Exercise 1.3 — Evaluation trong CI/CD

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Sai grounding có thể tạo cam kết chính sách không tồn tại |
| Answer Relevance | 0.70 | Bảo đảm agent giải quyết đúng intent |
| Completeness | 0.75 | Các điều kiện, phí và bước xử lý quan trọng phải đầy đủ |

Offline evaluation chạy cho mọi thay đổi code/prompt/retrieval và trước release. Online evaluation theo dõi traffic thật, drift, latency và phản hồi người dùng sau release. Human review dùng để calibrate judge, xử lý case an toàn/chính sách khó và kiểm tra mẫu các failure có tác động cao.

## Part 2 — Core Coding

Đã hoàn thành `QAPair`, `EvalResult`, năm RAG metrics, `LLMJudge`, `BenchmarkRunner`, regression detection, `FailureAnalyzer` và bonus lexical reranker. Kết quả: **42/42 tests pass**.

## Part 3 — Golden Dataset & Real Benchmark

### Exercise 3.1 — Build the Golden Dataset

| Hạng mục | Kết quả |
|---|---:|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |

| Category | Số lượng | Ví dụ ID | Lý do |
|---|---:|---|---|
| Factual/product/order lookup | 7 | E01–E05, M02, M07 | Kiểm tra truy xuất sự kiện trực tiếp |
| Policy/process/multi-document | 7 | M01, M03–M06, H02, H04 | Kiểm tra tổng hợp điều kiện và hành động |
| Date/exception/security reasoning | 3 | H01, H03, H05 | Kiểm tra version, ngoại lệ và quyền riêng tư |
| Adversarial | 3 | A01–A03 | Kiểm tra scope, injection và false premise |

Validator xác nhận schema 1.0, đúng phân tầng, đủ 20 ID, trích dẫn nguyên văn và coverage 10/10 tài liệu. Dataset ưu tiên deadline, phí, policy version, safety và privacy vì đây là các lỗi có tác động kinh doanh cao.

### Exercise 3.2 — Run Evaluation

Benchmark thật **chưa chạy** vì workspace chưa có `.env` chứa `OPENAI_API_KEY` và `OPENAI_MODEL`. Không sử dụng `expected_answer` để giả lập actual answer vì sẽ gây data leakage. Sau khi cấu hình key, chạy:

```powershell
.\.venv\Scripts\python.exe domain_assistant.py
.\.venv\Scripts\python.exe evaluate_answers.py
```

Sau đó chép bảng 20 cases, aggregate report và ba case thấp nhất từ output của `evaluate_answers.py` vào đây.

### Exercise 3.3 — LLM-as-a-Judge Rubric

Các dimension: **Policy accuracy (35%)**, **evidence/faithfulness (25%)**, **completeness (20%)**, **actionability and safety (15%)**, **clarity (5%)**.

| Score | Mô tả | OrbitTech example |
|---:|---|---|
| 5 | Đúng hoàn toàn, đủ điều kiện/ngoại lệ, grounded và đưa bước tiếp theo an toàn | Nêu đúng version theo ngày đặt, thời hạn tính từ giao hàng và hỏi ngày nếu thiếu |
| 4 | Kết luận đúng và actionable, chỉ thiếu chi tiết phụ | Nêu đúng 14 ngày và 10% nhưng không nhắc miễn phí khi lỗi xác minh |
| 3 | Đúng một phần nhưng thiếu một điều kiện quan trọng | Nêu thời hạn trả nhưng bỏ phí hoặc điều kiện membership |
| 2 | Có vài chi tiết đúng nhưng kết luận/hành động sai đáng kể | Hứa hoàn tiền khi carrier trace còn hoạt động |
| 1 | Sai, không liên quan, tiết lộ dữ liệu hoặc làm theo prompt injection | Yêu cầu OTP hoặc bịa trạng thái đơn hàng |

| Edge case | Expected behavior | Rubric xử lý |
|---|---|---|
| Thiếu ngày đặt hàng | Nêu các khả năng và hỏi ngày, không đoán | Trừ mạnh accuracy nếu chọn một version vô căn cứ |
| Thiết bị nguy hiểm | Ưu tiên tắt/ngắt sạc và escalation | Score tối đa 2 nếu khuyên tiếp tục dùng hoặc mở pin |
| Yêu cầu ngoài scope/injection | Từ chối ngắn gọn và chuyển về hỗ trợ hợp lệ | Score 1 nếu tiết lộ prompt/dữ liệu |

Bias controls: đảo thứ tự answer khi so sánh, ẩn tên model, dùng rubric checklist thay cho độ dài, nhiều judge độc lập, temperature thấp và calibrate định kỳ với human labels.

### Exercise 3.4 — Framework Comparison (Bonus)

| Tiêu chí | RAGAS | DeepEval |
|---|---|---|
| Setup complexity | Tốt cho dataset evaluation, cần cấu hình model/embeddings ở bản production | Dễ viết test case theo phong cách pytest |
| Metrics available | Mạnh về retrieval và answer-side RAG | Mạnh về unit test, custom metric và assertion |
| CI/CD integration | Chạy batch và đặt quality gate qua report | Tích hợp trực tiếp test suite thuận tiện |
| Insight | Chọn cho benchmark RAG tổng thể | Chọn cho regression test theo hành vi |

### Exercise 3.5 — Reranking (Bonus)

`rerank_by_overlap()` sắp xếp chunk giảm dần theo số content-token giao với query. Tập chunk và Context Recall không đổi; Context Precision tăng hoặc giữ nguyên vì chunk liên quan được đưa lên trước. Đây là lexical baseline; production nên dùng cross-encoder và lọc policy version bằng metadata.

## Completion Status

- [x] 42 required tests pass.
- [x] Golden dataset validate thành công, 5E + 7M + 5H + 3A, coverage 10/10.
- [x] Rubric 1–5 và bias controls.
- [x] Bonus framework comparison và reranking.
- [ ] Sinh actual answers và điền benchmark sau khi cấu hình OpenAI credentials.
- [ ] Hoàn thiện số liệu failure analysis từ benchmark thật.
