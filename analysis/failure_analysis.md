# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Cá nhân  
**Thành viên:** Phạm Khắc Khương Duy

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8083 | 0.7417 | -0.0667 |
| Answer Relevancy | 0.7917 | 0.7343 | -0.0574 |
| Context Precision | 0.9250 | 0.9500 | +0.0250 |
| Context Recall | 0.9250 | 0.8583 | -0.0667 |

## Bottom-5 Failures

### #1
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Không có thông tin Expected/Ground Truth trong `ragas_report.json`.
- **Got:** Không có nội dung câu trả lời thực tế trong `ragas_report.json`.
- **Worst metric:** Faithfulness — score: **0.3333**
- **Error Tree:** Output sai → Context đúng? → Chưa xác minh được từ report → Query/retrieval có thể đã lấy thiếu evidence hoặc LLM đã thêm thông tin ngoài context → Diagnosis: LLM hallucinating.
- **Root cause:** Production pipeline có Context Precision cao nhưng Faithfulness của câu hỏi này rất thấp. Khả năng chính là LLM đã trả lời vượt quá thông tin có trong retrieved context. Cần log context và answer để xác định chính xác đây là retrieval failure hay generation failure.
- **Suggested fix:** Tighten prompt, lower temperature. Thêm ràng buộc: "Chỉ trả lời dựa trên context. Nếu context không có thông tin, hãy nói không tìm thấy thông tin trong tài liệu."

### #2
- **Question:** Mật khẩu phải có tối thiểu bao nhiêu ký tự?
- **Expected:** Không có thông tin Expected/Ground Truth trong `ragas_report.json`.
- **Got:** Không có nội dung câu trả lời thực tế trong `ragas_report.json`.
- **Worst metric:** Faithfulness — score: **0.5000**
- **Error Tree:** Output sai → Context đúng? → Chưa xác minh được từ report → Có thể chunk chứa quy định về số ký tự không được retrieve hoặc LLM tự suy đoán một con số phổ biến → Diagnosis: LLM hallucinating.
- **Root cause:** Có thể xảy ra vocabulary/retrieval mismatch hoặc chunking làm tách phần "độ dài tối thiểu" khỏi phần quy định về mật khẩu. Nếu context đúng thì nguyên nhân là generation hallucination.
- **Suggested fix:** Kiểm tra retrieved chunks của câu hỏi này, đảm bảo chunk chứa quy định về độ dài mật khẩu được retrieve. Đồng thời ép LLM không tự suy đoán số liệu nếu context không cung cấp.

### #3
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Không có thông tin Expected/Ground Truth trong `ragas_report.json`.
- **Got:** Không có nội dung câu trả lời thực tế trong `ragas_report.json`.
- **Worst metric:** Faithfulness — score: **0.7223**
- **Error Tree:** Output sai → Context đúng? → Câu hỏi cần nhiều evidence → Có thể thiếu quy định về thời hạn hoặc công thức tính phạt → Nếu context đủ thì LLM có thể tự tính/suy luận sai → Diagnosis: LLM hallucinating.
- **Root cause:** Đây là câu hỏi reasoning nhiều bước, cần kết hợp số tiền, thời hạn thanh toán và quy định mức phạt. Nếu một evidence bị loại trong retrieval hoặc reranking top-k, LLM có thể tự suy diễn công thức.
- **Suggested fix:** Tăng số lượng context sau reranking cho các câu hỏi cần nhiều evidence, ví dụ thử top-5 thay vì top-3. Prompt cũng nên yêu cầu mô hình chỉ tính toán dựa trên công thức có trong context.

### #4
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected:** Không có thông tin Expected/Ground Truth trong `ragas_report.json`.
- **Got:** Không có nội dung câu trả lời thực tế trong `ragas_report.json`.
- **Worst metric:** Faithfulness — score: **0.7327**
- **Error Tree:** Output sai → Context đúng? → Câu hỏi gồm hai yêu cầu: người phê duyệt + yêu cầu từ CNTT → Có thể evidence nằm ở nhiều chunks → Reranker có thể chỉ giữ một phần context → Diagnosis: LLM hallucinating.
- **Root cause:** Đây là câu hỏi multi-hop. Thông tin về cấp phê duyệt và yêu cầu từ phòng CNTT có thể nằm ở các section khác nhau. Pipeline có Context Precision cao nhưng Context Recall thấp hơn baseline, cho thấy một phần evidence có thể bị loại.
- **Suggested fix:** Tăng reranker top-k từ top-3 lên top-5 và benchmark lại Context Recall. Có thể sử dụng hierarchical hoặc structure-aware chunking để giữ các quy trình liên quan trong context tốt hơn.

### #5
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** Không có thông tin Expected/Ground Truth trong `ragas_report.json`.
- **Got:** Không có nội dung câu trả lời thực tế trong `ragas_report.json`.
- **Worst metric:** Answer Relevancy — score: **0.7500**
- **Error Tree:** Output sai → Context đúng? → Worst metric không phải Context Recall mà là Answer Relevancy → Có khả năng context đã liên quan nhưng câu trả lời không trả lời trực tiếp câu hỏi → Diagnosis: Answer doesn't match question.
- **Root cause:** Generation có thể trả lời lan man về chính sách bảo hiểm chung nhưng không đưa ra câu trả lời trực tiếp cho việc nhân viên thử việc có được hưởng bảo hiểm PVI hay không.
- **Suggested fix:** Improve prompt template. Yêu cầu câu đầu tiên phải trả lời trực tiếp câu hỏi, sau đó mới giải thích dựa trên context. Ví dụ: "Trước tiên, trả lời trực tiếp Có/Không hoặc thông tin chính. Sau đó giải thích ngắn gọn bằng evidence trong context."

## Case Study (cho presentation)

**Question chọn phân tích:** Bao lâu phải đổi mật khẩu một lần?

Câu hỏi này được chọn vì có failure score thấp nhất trong report (**0.3333**) và worst metric là **Faithfulness**.

**Error Tree walkthrough:**
1. Output đúng? → Chưa thể xác minh trực tiếp vì report không lưu answer và ground truth.
2. Context đúng? → Chưa thể xác minh vì report không lưu retrieved contexts.
3. Query rewrite OK? → Cần kiểm tra pipeline log. Query ngắn và rõ nghĩa, vì vậy khả năng chính cần kiểm tra là retrieval evidence hoặc generation hallucination.
4. Fix ở bước: **Generation prompt trước**, đồng thời thêm retrieval logging để xác định chính xác nguyên nhân.

Suggested prompt:

> Chỉ sử dụng thông tin được cung cấp trong Context để trả lời. Không sử dụng kiến thức bên ngoài và không tự suy đoán. Nếu Context không chứa đủ thông tin để trả lời, hãy nói rõ rằng không tìm thấy thông tin cần thiết.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Thêm log `question → retrieved contexts → rerank scores → final answer → ground truth`.
- Giảm temperature và siết chặt prompt "chỉ trả lời dựa trên context".
- Benchmark reranker top-3 với top-5 để kiểm tra trade-off giữa Context Precision và Context Recall.
- Tạo regression test cho 5 câu hỏi failure thấp nhất.
- Chạy lại `python main.py` và so sánh RAGAS scores trước/sau.