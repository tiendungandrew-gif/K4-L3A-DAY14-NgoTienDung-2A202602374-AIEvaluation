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
| Faithfulness | Câu chào hỏi xã giao thông thường, câu từ chối lịch sự khi ngoài phạm vi ("Tôi không tìm thấy thông tin..."), hoặc các câu giải thích ngữ cảnh chung không chứa số liệu/cam kết cụ thể. | Câu trả lời bịa đặt chính sách (hallucination) về số ngày đổi trả, phí hoàn tiền, điều kiện bảo hành hoặc giá cả trái ngược với context nguồn của OrbitTech. | Siết chặt system prompt ("chỉ dùng context được cung cấp"), hạ temperature LLM xuống gần 0, bổ sung few-shot guardrails phát hiện và phạt hallucination. |
| Answer Relevance | Câu trả lời lịch sự từ chối prompt injection / out-of-scope, hoặc phản hồi có bổ sung thêm cảnh báo an toàn/lưu ý bảo mật quan trọng khiến độ trùng lặp từ ngữ trực tiếp với câu hỏi giảm đi. | Câu trả lời lạc đề hoàn toàn, lặp lại thông tin không liên quan (ví dụ: hỏi về chính sách đổi trả hàng phụ kiện nhưng trả lời quy trình trả góp qua thẻ tín dụng). | Cải thiện prompt phân tích ý định (Intent Parsing / Query Reformulation), tinh chỉnh system instruction để đi thẳng vào trọng tâm câu hỏi của khách hàng. |
| Context Recall | Câu hỏi factual đơn giản chỉ cần 1 thông tin duy nhất, các chi tiết phụ trợ trong expected answer không thực sự bắt buộc đối với việc giải quyết vấn đề của khách hàng. | Retriever bỏ sót các điều kiện tiên quyết, ngoại lệ chính sách quan trọng (ví dụ: điều kiện hàng phải nguyên seal hoặc trừ phí restocking 15%), dẫn đến câu trả lời thiếu sót gây tranh chấp. | Tăng Top-K, điều chỉnh tham số BM25 (k1, b), kết hợp hybrid search (BM25 + Dense Semantic Embeddings), tối ưu hóa chunk size và chunk overlap. |
| Context Precision | Top-K được lấy rộng (K=5 hoặc K=10) để tối ưu Recall tối đa; context chứa các chunk nhiễu nhưng đứng sau và LLM vẫn đủ năng lực chắt lọc đúng thông tin để sinh câu trả lời chuẩn xác. | Các chunk tài liệu chứa thông tin đúng bị xếp ở vị trí cuối (rank thấp) hoặc bị chìm trong nhiều chunk rác ở đầu, dẫn đến hiện tượng "lost-in-the-middle" hoặc LLM bị phân tâm sinh lỗi. | Triển khai Reranker (Cross-Encoder / Cohere Rerank / BM25 rerank), tối ưu scoring function, áp dụng similarity threshold để lọc bỏ các chunk có điểm relevance quá thấp trước khi nạp vào LLM. |
| Completeness | Khách hàng chỉ hỏi xác nhận có/không hoặc câu hỏi tóm tắt nhanh; câu trả lời đi thẳng vào kết luận súc tích thay vì giải thích dông dài toàn bộ quy trình như expected answer mẫu. | Khách hàng hỏi quy trình/hồ sơ đầy đủ (ví dụ: hồ sơ gửi bảo hành gồm những giấy tờ gì) nhưng câu trả lời bỏ quên hơn một nửa yêu cầu bắt buộc khiến khách làm sai thủ tục. | Bổ sung hướng dẫn định dạng có cấu trúc vào prompt (dùng checklist, bullet points), áp dụng Chain-of-Thought để rà soát đầy đủ các tiêu chí trước khi hoàn tất phản hồi. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> *Câu trả lời:*
> - **Thiết kế Experiment (Pairwise Swap Evaluation):**
>   - Chọn tập test gồm $N = 30-50$ cặp câu trả lời $(A, B)$ từ hai hệ thống khác nhau cho cùng một tập câu hỏi.
>   - **Condition 1 (Order A-B):** Đưa vào prompt của LLM Judge: `Candidate 1: A`, `Candidate 2: B`. Yêu cầu Judge chọn câu trả lời tốt hơn hoặc chấm điểm từng câu.
>   - **Condition 2 (Order B-A):** Đảo ngược vị trí cùng cặp câu trả lời: `Candidate 1: B`, `Candidate 2: A` với cùng rubric và prompt template.
> - **Đo lường & Phân tích:**
>   - Tính tỷ lệ chọn Candidate 1 ở cả hai điều kiện: $P(\text{Win} \mid \text{Pos 1})$ so với $P(\text{Win} \mid \text{Pos 2})$.
>   - Nếu $P(\text{Pos 1}) \gg 50\%$ hoặc tỷ lệ mâu thuẫn (chọn A ở Condition 1 nhưng chọn B ở Condition 2) vượt quá 10–15%, Judge có Position Bias rõ rệt.
>   - **Giải pháp:** Chạy song song cả hai thứ tự và lấy trung bình điểm (Position Permutation Averaging), hoặc chỉ công nhận kết quả khi cả hai lượt đảo vị trí đều cho cùng một phán quyết.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> *Câu trả lời:*
> 1. **Tiêu chí rõ ràng về mật độ thông tin (Information Density):** Bổ sung tiêu chí "Conciseness & Directness" vào rubric; quy định rõ ràng rằng câu trả lời dài dòng, chứa nhiều từ đệm sáo rỗng hoặc lặp lại ý sẽ bị trừ điểm (ví dụ: không được quá 4/5 nếu dài dòng không cần thiết).
> 2. **Chấm điểm dựa trên Factual Coverage thay vì độ dài:** Định nghĩa checklist các "key facts / mandatory conditions" cần có. Judge chỉ đếm số lượng factual claims chính xác, không tính điểm dựa trên cảm giác đầy đặn của văn bản.
> 3. **Ràng buộc độ dài mục tiêu (Length Constraint):** Cung cấp khung độ dài mong muốn trong prompt (ví dụ: "Phản hồi lý tưởng từ 2–4 câu hoặc dưới 120 từ; vượt quá giới hạn mà không bổ sung thông tin mới sẽ bị trừ điểm").
> 4. **Chain-of-Thought trước khi chấm:** Yêu cầu Judge thực hiện trích xuất facts: `[Trích xuất fact] -> [So khớp evidence] -> [Đánh giá độ súc tích] -> [Cho điểm]`.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> *Câu trả lời:*
> 1. **Căn chỉnh với chuẩn mực con người (Human Alignment):** LLM Judge có thể có các thiên kiến nội tại (như quá dễ dãi - leniency bias, quá khắt khe - severity bias, hoặc tự ưu ái mô hình cùng họ - self-preference bias). Calibration với tập nhãn của chuyên gia người thật giúp đo lường mức độ tương đồng qua các chỉ số như Cohen's Kappa, Spearman's rank correlation hoặc Pearson correlation.
> 2. **Tối ưu hóa Rubric và Prompt:** Kết quả so sánh với Human Ground Truth chỉ ra các điểm mù (blind spots) và trường hợp đánh giá sai (edge cases), từ đó hiệu chỉnh lại tiêu chí chấm và bổ sung các few-shot examples thực tế vào prompt của Judge.
> 3. **Đảm bảo tính tin cậy khi làm CI/CD Quality Gate:** Chỉ khi LLM Judge đạt độ tương quan cao với con người (thường target Cohen's Kappa $\ge 0.75$), chúng ta mới có thể tin tưởng giao cho hệ thống tự động chặn/duyệt bản build trong pipeline triển khai liên tục.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | 0.85 | **An toàn thông tin & Chống Hallucination:** Trong domain CSKH OrbitTech, thông tin sai sự thật về chính sách hoàn tiền, thời hạn bảo hành hay tương thích phần cứng sẽ gây thiệt hại tài chính và tranh chấp pháp lý nghiêm trọng với khách hàng. Đây là tiêu chí sống còn, phải đặt ngưỡng khắt khe nhất để làm hard gate. |
| Answer Relevance | 0.75 | **Trải nghiệm khách hàng & Giảm tải vận hành:** Câu trả lời phải đi thẳng vào thắc mắc của khách. Nếu Relevance dưới 0.75, câu trả lời sẽ vòng vo, lạc đề, khiến khách hàng ức chế và buộc phải leo thang (escalate) lên tổng đài viên người thật. |
| Completeness | 0.70 | **Đầy đủ điều kiện cần thiết:** Khách hàng cần đủ thông tin để thực hiện thủ tục (ví dụ: giấy tờ cần chuẩn bị khi bảo hành). Ngưỡng 0.70 cho phép câu trả lời ngắn gọn, súc tích nhưng vẫn đảm bảo không bỏ sót các điều khoản loại trừ hay điều kiện tiên quyết cốt lõi. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> *Câu trả lời:*
> - **Offline Evaluation (Pre-deployment / Quality Gate CI/CD):**
>   - *Thời điểm:* Sử dụng trước khi deploy bất kỳ thay đổi nào về prompt, model version, retrieval parameters (Top-K, chunk size, BM25) hoặc cập nhật tài liệu knowledge base.
>   - *Mục đích:* Chạy tự động trên bộ Golden Dataset chuẩn (20–100+ cases) nhằm phát hiện sớm hồi quy (regression) và chặn đứng các bản build không đạt chuẩn threshold trước khi đến tay người dùng.
> - **Online Evaluation (Production Monitoring & Drift Detection):**
>   - *Thời điểm:* Sử dụng liên tục trong quá trình hệ thống đang phục vụ traffic người dùng thật trên môi trường Production.
>   - *Mục đích:* Đo lường các chỉ số tương tác thực tế như tỷ lệ thumbs up/down, implicit feedback (tỷ lệ copy, thời gian đọc, tỷ lệ chuyển tiếp nhân viên - escalation rate), latency, chi phí token; đồng thời lấy mẫu (sampling 1–5%) để LLM Judge ngầm chấm điểm nhằm phát hiện data drift / concept drift.
> - **Human Review (Calibration, Dispute & High-Risk Auditing):**
>   - *Thời điểm:* Thực hiện định kỳ (hàng tuần/hàng tháng) hoặc khi xảy ra sự cố (P0/P1 incidents, khiếu nại của khách hàng, escalation tăng đột biến).
>   - *Mục đích:* Chuyên gia domain thẩm định các trường hợp ranh giới (borderline cases, điểm judge dao động quanh ngưỡng threshold), kiểm toán an toàn (safety/jailbreak), và tạo nhãn chuẩn để định kỳ hiệu chỉnh (calibrate) lại chính LLM Judge và mở rộng Golden Dataset.

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
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

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
