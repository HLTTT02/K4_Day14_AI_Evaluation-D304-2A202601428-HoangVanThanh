# Day 14 — Evaluation Report & Reflection

## 1. Executive Summary

Evaluation core và golden dataset đã hoàn tất. Benchmark inference đang chờ OpenAI credentials; vì vậy báo cáo không bịa pass rate hoặc failure score. Khi có artifact thật, `evaluate_answers.py` sẽ tạo `artifacts/benchmark_results.json` để cập nhật phần 2–4.

## 2. Evaluation Design

- 20 cases phân tầng: 5 easy, 7 medium, 5 hard, 3 adversarial.
- Coverage: 10/10 tài liệu corpus; evidence là substring nguyên văn và đã qua validator.
- Answer-side: Faithfulness, Relevance, Completeness; pass khi cả ba >= 0.5.
- Retrieval-side: Context Recall và rank-aware Context Precision; hai metric này dùng chẩn đoán retriever, không thay đổi pass rule.
- CI thresholds đề xuất: Faithfulness 0.80, Relevance 0.70, Completeness 0.75; regression khi average giảm quá 0.05.

## 3. Benchmark Results

Trạng thái: **Pending — `.env` chưa có `OPENAI_API_KEY` và `OPENAI_MODEL`.**

Lệnh tái lập:

```powershell
.\.venv\Scripts\python.exe validate_golden_dataset.py
.\.venv\Scripts\python.exe domain_assistant.py
.\.venv\Scripts\python.exe evaluate_answers.py
```

Không thay actual answers bằng expected answers: cách đó làm benchmark mất độc lập và tạo data leakage.

## 4. Failure Analysis Plan — 5 Whys

Sau khi benchmark chạy, chọn ba case có `overall` thấp nhất và hoàn thiện mỗi chuỗi sau bằng trace retrieval và actual answer:

1. **Symptom:** metric nào thấp và claim/bước nào sai hoặc thiếu?
2. **Why 1:** generator sai vì thiếu evidence, dùng evidence sai hay prompt không buộc kiểm tra điều kiện?
3. **Why 2:** retriever bỏ lỡ chunk, ranking thấp, chunk quá rộng hay query không chứa từ khóa chính sách?
4. **Why 3:** dataset/prompt/index chưa biểu diễn version, deadline, ngoại lệ hoặc intent?
5. **Why 4:** quality gate nào đã thiếu để lỗi lọt qua?
6. **Why 5 / actionable root cause:** thay đổi nhỏ nhất có thể đo lại là gì?

Không quy nguyên nhân retrieval chỉ từ Faithfulness thấp: cần so Context Recall/Precision và kiểm tra retrieved chunks. Tương tự, recall tốt nhưng completeness thấp thường chỉ ra lỗi generation/prompt.

## 5. Expected Failure Clusters and Actions

| Cluster | Evidence cần kiểm tra | Improvement có thể đo |
|---|---|---|
| Hallucination / sai policy | Claim không xuất hiện trong gold hoặc retrieved context | Grounding guardrail, citation requirement, block nếu faithfulness < 0.80 |
| Incomplete / thiếu ngoại lệ | Recall cao nhưng completeness thấp | Checklist deadline/fee/exception/next action trong prompt |
| Retrieval miss | Context Recall thấp | Query expansion, chunk overlap, tăng top-k có kiểm soát |
| Ranking noise | Recall cao nhưng Context Precision thấp | Lexical/cross-encoder reranking và metadata filter |
| Adversarial failure | Làm theo injection, đoán false premise hoặc trả lời ngoài scope | Scope classifier, immutable system rules, adversarial regression tests |

## 6. Improvement Log Template

| Failure ID | Type | Root Cause | Suggested Fix | Priority | Owner | Status |
|---|---|---|---|---|---|---|
| Pending-1 | Pending benchmark | Chờ actual trace | Chạy inference và phân tích case thấp nhất | High | AI team | Open |
| Pending-2 | Pending benchmark | Chờ actual trace | Chạy inference và phân tích case thấp thứ hai | High | AI team | Open |
| Pending-3 | Pending benchmark | Chờ actual trace | Chạy inference và phân tích case thấp thứ ba | High | AI team | Open |

Ưu tiên theo tác động: safety/privacy và sai chính sách trước, sau đó missing actions/deadlines, cuối cùng mới tối ưu độ rõ ràng. Mỗi thay đổi phải gắn với case ID và metric mục tiêu.

## 7. Regression & CI/CD Strategy

```text
Code/prompt/retrieval change → unit tests → offline benchmark → compare baseline → quality gate → deploy → online monitoring
```

Deployment bị block khi:

1. Bất kỳ required test hoặc dataset validation nào fail.
2. Average metric giảm hơn 0.05 so với baseline.
3. Faithfulness < 0.80, Relevance < 0.70 hoặc Completeness < 0.75.
4. Một case safety/privacy/prompt-injection quan trọng fail, dù average toàn bộ vẫn đạt.

Offline eval chạy ở pull request và trước release. Online monitoring theo dõi failure rate theo intent/difficulty, latency, cost và user escalation. Human review lấy mẫu định kỳ, toàn bộ safety failure, và dùng để calibrate LLM judge.

## 8. Limitations

Heuristic token overlap nhanh, deterministic và phù hợp CI nhưng không hiểu tốt từ đồng nghĩa, phủ định hoặc tương đương ngữ nghĩa, đặc biệt khi question/answer dùng tiếng Việt còn corpus dùng tiếng Anh. Production nên bổ sung semantic embeddings hoặc judge đã calibrate, nhưng vẫn giữ deterministic tests làm lớp kiểm tra cơ bản. Golden dataset hiện nhỏ và cần được tăng cường bằng các failure thật sau mỗi vòng Evaluate → Analyze → Improve → Augment → Repeat.
