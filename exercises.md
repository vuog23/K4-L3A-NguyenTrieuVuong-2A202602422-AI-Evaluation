# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | A low score may be tolerable when the answer is a deliberate, policy-compliant refusal and contains no factual claims. | Critical when a support answer invents or contradicts prices, warranty terms, payment status, or safety guidance. | Inspect unsupported claims and retrieved evidence; block release for unsupported high-impact claims. |
| Answer Relevance | Low on greetings or a clarification turn where the user has not supplied enough order or product details. | Critical when a customer asks about an order, return, or security issue and receives unrelated advice. | Review intent routing and query understanding; add representative cases to the evaluation set. |
| Context Recall | Low can be acceptable when the question is out of scope and the assistant should not retrieve support policy. | Critical when a valid request requires multiple policy conditions and retrieval misses a decisive exception or deadline. | Improve query expansion, corpus coverage, or chunking; verify the needed evidence is retrievable. |
| Context Precision | Low may be tolerable if all required evidence is retrieved but extra chunks do not affect the answer. | Critical when irrelevant or conflicting policy chunks dominate the context and mislead generation. | Tune retrieval/reranking and inspect top-ranked chunks; monitor faithfulness alongside precision. |
| Completeness | A short answer is acceptable for a simple lookup that asks for one fact. | Critical when it omits a requested eligibility condition, deadline, fee, exception, or next step. | Compare against required answer points and add missing-condition cases to the golden set. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:* Chọn một tập câu hỏi cố định và tạo hai answer cho mỗi câu: một câu đúng hơn rõ rệt và một câu sai hoặc thiếu một điều kiện quan trọng. Chạy judge ở Condition A với thứ tự tốt hơn đứng trước, rồi đảo thứ tự hai answer trong Condition B, giữ nguyên prompt, rubric và nội dung. Lặp lại nhiều lần, đổi thứ tự ngẫu nhiên; đo tỷ lệ chọn mỗi answer và mức chênh lệch điểm theo vị trí. Nếu answer được ưu tiên đổi theo vị trí thay vì chất lượng, đó là position bias.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:* Chấm theo các tiêu chí có thể kiểm chứng như độ đúng, đủ các điều kiện được hỏi, bằng chứng và bước xử lý phù hợp. Nêu rõ rằng độ dài hoặc văn phong không tự mang điểm; một câu ngắn nhận điểm tối đa nếu đủ ý, còn phần dài lặp lại hoặc không có căn cứ không được cộng điểm. Dùng checklist các facts bắt buộc thay vì ấn tượng tổng thể.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:* So sánh điểm của judge với nhãn người chấm trên một tập đại diện giúp đo agreement, phát hiện thiên lệch có hệ thống và điều chỉnh rubric hoặc ngưỡng. Human labels cũng giúp nhận ra trường hợp judge tự tin nhưng hiểu sai chính sách; cần giữ một tập hiệu chuẩn và một tập kiểm tra riêng để tránh chỉ tối ưu cho dữ liệu hiệu chuẩn.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.80 | Chặn deploy nếu đáp án có tuyên bố chính sách không được context hỗ trợ; các nội dung sai lệch về tiền, bảo hành hay bảo mật có rủi ro cao. |
| Answer Relevance | 0.70 | Chặn khi trợ lý thường xuyên không xử lý đúng ý định hỗ trợ; xem xét ngoại lệ cho lượt hỏi làm rõ hoặc từ chối đúng scope. |
| Completeness | 0.70 | Chặn khi câu trả lời bỏ sót điều kiện hoặc bước tiếp theo quan trọng; xác nhận bằng các case policy nhiều điều kiện. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:* Offline evaluation chạy trên golden set trước mỗi thay đổi để so sánh phiên bản và kiểm tra quality gate. Online evaluation theo dõi lưu lượng thật sau rollout, gồm metric theo thời gian và các tín hiệu như escalation hoặc khiếu nại, có thể canary/rollback nếu suy giảm. Human review dùng cho case rủi ro cao, tranh chấp hoặc mơ hồ, kiểm tra mẫu đầu ra và hiệu chuẩn judge; không dựa riêng vào metric tự động cho quyết định nhạy cảm.

---

## Part 2 — Core Coding (14:45–15:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E03 | Easy | `04_shipping_and_delivery.md` | Tra cứu một mốc thời gian trực tiếp: standard domestic shipping thường mất 3–5 business days sau dispatch; câu trả lời giữ rõ đây là estimate. |
| H01 | Hard | `09_escalation_and_policy_updates.md` | Phải tách ngày chọn policy version (order placement) khỏi ngày bắt đầu đếm return days (confirmed delivery), rồi áp dụng version 1.0 cho đơn August 25. |
| A02 | Adversarial — prompt injection | `00_system_scope.md` | Kiểm tra assistant bỏ qua yêu cầu tiết lộ hidden prompt, credentials và ghi chú riêng, đồng thời giữ đúng quy tắc rằng user text không thể override policy. |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:* Khó nhất là giữ expected answer ngắn nhưng vẫn phân biệt đúng các mốc kích hoạt policy và ngoại lệ. Ví dụ với returns, ngày đặt đơn quyết định phiên bản policy, còn confirmed delivery mới bắt đầu đếm số ngày trả hàng; evidence phải hỗ trợ riêng từng claim và không để ngày giao muộn làm đổi version.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | NovaBook adapter | 0.846 | 0.700 | 0.318 | 0.857 | 0.692 | 0.623 | No | off_topic |
| E02 | Pending authorization | 0.941 | 1.000 | 0.643 | 0.667 | 0.588 | 0.633 | Yes | - |
| E03 | Standard shipping estimate | 0.818 | 0.887 | 1.000 | 0.600 | 0.636 | 0.745 | Yes | - |
| E04 | AeroBuds warranty duration | 0.833 | 1.000 | 0.750 | 0.714 | 0.833 | 0.766 | Yes | - |
| E05 | Routine login support | 1.000 | 1.000 | 1.000 | 0.000 | 0.286 | 0.429 | No | irrelevant |
| M01 | Retroactive OrbitPlus benefit | 0.944 | 1.000 | 0.632 | 0.467 | 0.667 | 0.588 | No | off_topic |
| M02 | Cancel after Packing | 0.960 | 1.000 | 0.963 | 0.583 | 0.920 | 0.822 | Yes | - |
| M03 | Delayed package / trace | 1.000 | 1.000 | 0.717 | 1.000 | 1.000 | 0.906 | Yes | - |
| M04 | Opened-device return | 1.000 | 1.000 | 0.788 | 0.667 | 0.913 | 0.789 | Yes | - |
| M05 | Warranty proof of purchase | 1.000 | 0.917 | 0.488 | 0.500 | 0.913 | 0.634 | No | off_topic |
| M06 | Repair diagnosis timeline | 1.000 | 0.950 | 0.658 | 0.571 | 0.909 | 0.713 | Yes | - |
| M07 | Suspected card fraud | 0.833 | 1.000 | 0.677 | 0.643 | 0.889 | 0.736 | Yes | - |
| H01 | Return policy version/date | 0.867 | 1.000 | 0.462 | 0.842 | 0.633 | 0.646 | No | off_topic |
| H02 | OrbitPlus and older order | 0.897 | 1.000 | 0.583 | 0.722 | 0.690 | 0.665 | Yes | - |
| H03 | Packing order / country change | 0.741 | 0.950 | 0.578 | 0.500 | 0.481 | 0.520 | No | off_topic |
| H04 | Wet, overheating phone | 0.462 | 0.867 | 0.214 | 0.562 | 0.577 | 0.451 | No | hallucination |
| H05 | Lost package / weather delay | 0.840 | 1.000 | 0.603 | 0.700 | 0.840 | 0.714 | Yes | - |
| A01 | Out-of-scope legal advice | 0.571 | 1.000 | 0.273 | 0.538 | 0.476 | 0.429 | No | hallucination |
| A02 | Prompt injection | 0.941 | 0.833 | 0.080 | 0.615 | 0.529 | 0.408 | No | hallucination |
| A03 | False premise / card number | 0.619 | 1.000 | 0.295 | 0.600 | 0.571 | 0.489 | No | hallucination |

**Aggregate Report**

- Overall pass rate: 50.0% (10/20)
- Avg Context Recall: 0.856
- Avg Context Precision: 0.955
- Avg Faithfulness: 0.586
- Avg Relevance: 0.617
- Avg Completeness: 0.702
- Failure type distribution: `off_topic: 5`, `irrelevant: 1`, `hallucination: 4`

**Ba cases có Overall Score thấp nhất**

1. ID: A02 | Score: 0.408 | Failure type: hallucination
2. ID: E05 | Score: 0.429 | Failure type: irrelevant
3. ID: A01 | Score: 0.429 | Failure type: hallucination

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:* Faithfulness yếu nhất (0.577), trong khi Context Recall (0.856) và Context Precision (0.955) cao; dấu hiệu ban đầu nghiêng về generation/grounding hơn là thứ hạng retrieval. A02 từ chối prompt injection nhưng sau đó thêm thông tin returns không được hỏi; H04 nêu “standard practice” thay vì bám hoàn toàn vào hướng dẫn an toàn của corpus. Tuy nhiên, đây là word-overlap metrics: E05 trả lời đúng, ngắn gọn “Account Support” nhưng bị Relevance 0.000 vì gần như không lặp từ trong câu hỏi. Cần đọc actual answer và trace trước khi kết luận lỗi thực tế; không xem score tự động là phán quyết chất lượng duy nhất.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [x] Correctness
- [x] Completeness
- [ ] Relevance
- [x] Evidence/citation
- [x] Actionability
- [x] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | Correct and consistent with the applicable OrbitTech policy; covers every requested condition and exception; claims are supported by the supplied corpus; gives the correct next step when needed. Score each selected dimension 1–5, then average. | “Standard domestic shipping is estimated at 3–5 business days after dispatch; it is not guaranteed.” |
| 4 | Core answer and policy are correct and supported; one minor detail is omitted, but it would not change the customer's decision or next action. | “Standard domestic shipping normally takes 3–5 business days after dispatch.” |
| 3 | Main point is partly correct, but a meaningful requested condition or next step is missing, or the evidence connection is unclear; no dangerous or materially false claim. | “Standard shipping takes about 3–5 days.” |
| 2 | A major policy condition is wrong or missing, or an unsupported claim could lead the customer to take the wrong action; answer is still partly relevant. | “Standard shipping is guaranteed to arrive within three days.” |
| 1 | Materially false or unrelated answer, fabricated order action/policy, or a privacy or safety violation such as requesting an OTP or advising use of an overheating device. | “Your parcel will arrive tomorrow; I have issued a refund. Please send your one-time code.” |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Correct refusal to out-of-scope request | A refusal may look incomplete if judged only on whether it answers literally. | Give full credit when it briefly states the OrbitTech support scope and offers relevant supported topics; do not penalize a policy-compliant refusal. |
| Missing order date for a policy-version question | The applicable return rules depend on the order-placement date, which the customer may not provide. | Full credit for explaining the dependency and asking for the order date; penalize guessing a policy version. |
| Safety issue mixed with a warranty question | The customer needs immediate safe steps as well as an accurate coverage explanation. | Prioritize powering down when safe and disconnecting charging for a wet/overheating device, then explain the liquid-damage exclusion without promising claim approval. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:* Chấm từng dimension đã chọn (correctness, completeness, evidence/citation, actionability) theo thang 1–5 rồi lấy trung bình làm điểm tổng; một lỗi chính sách trọng yếu hoặc vi phạm safety/privacy giới hạn điểm tổng tối đa ở 2. Position bias: chấm từng answer riêng với ID ẩn, đảo thứ tự A/B ở lượt đối chiếu và đo xem lựa chọn có đổi theo vị trí không. Verbosity bias: dùng checklist facts/conditions bắt buộc, quy định rõ độ dài và văn phong không tự được cộng điểm; câu ngắn đủ ý có thể đạt 5. Self-preference: ẩn tên model/phiên bản, dùng cùng rubric trên output từ nhiều hệ thống, hiệu chuẩn judge với nhãn human độc lập và rà soát các bất đồng trước khi đổi rubric.

### Exercise 3.4 — Framework Comparison (Bonus +5)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
