# Group Report — Lab 18: Production RAG

**Nhóm:** Cá nhân — Phạm Khắc Khương Duy
**Ngày:** 18/08/2026

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Phạm Khắc Khương Duy | M1: Chunking | ☑ | 8/8 |
| Phạm Khắc Khương Duy | M2: Hybrid Search | ☑ | 5/5 |
| Phạm Khắc Khương Duy | M3: Reranking | ☑ | 5/5 |
| Phạm Khắc Khương Duy | M4: Evaluation | ☑ | 4/4 |

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.8083 | 0.7417 | -0.0667 |
| Answer Relevancy | 0.7917 | 0.7343 | -0.0574 |
| Context Precision | 0.9250 | 0.9500 | +0.0250 |
| Context Recall | 0.9250 | 0.8583 | -0.0667 |

## Key Findings

1. **Biggest improvement:**  
   Context Precision tăng từ **0.9250 lên 0.9500**, tương đương **+0.0250**. Điều này cho thấy production pipeline với Hybrid Search, Reranking và các kỹ thuật chunking đã giúp lọc context chính xác hơn, giảm số lượng chunk không liên quan được đưa vào LLM.

2. **Biggest challenge:**  
   Production pipeline chưa cải thiện toàn bộ metrics. Faithfulness giảm **0.0667**, Answer Relevancy giảm **0.0574**, và Context Recall giảm **0.0667** so với Naive Baseline. Challenge lớn nhất là cân bằng giữa việc chọn context chính xác hơn và việc giữ đủ thông tin cần thiết để trả lời câu hỏi. Reranking top-k có thể đã loại bỏ một số evidence cần thiết, đặc biệt với các câu hỏi cần nhiều thông tin từ nhiều chunks.

3. **Surprise finding:**  
   Mặc dù Context Precision đạt kết quả tốt nhất trong tất cả metrics (**0.9500**), Faithfulness và Answer Relevancy lại giảm so với baseline. Điều này cho thấy retrieval tốt hơn không tự động đảm bảo generation tốt hơn. Nếu context bị thiếu một phần evidence hoặc prompt chưa ràng buộc LLM đủ chặt, mô hình vẫn có thể hallucinate hoặc trả lời chưa trực tiếp vào câu hỏi.

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):**  
   - Faithfulness: **0.8083 → 0.7417 (-0.0667)**
   - Answer Relevancy: **0.7917 → 0.7343 (-0.0574)**
   - Context Precision: **0.9250 → 0.9500 (+0.0250)**
   - Context Recall: **0.9250 → 0.8583 (-0.0667)**

   Production pipeline cải thiện Context Precision nhưng chưa vượt baseline ở các metrics còn lại.

2. **Biggest win — module nào, tại sao:**  
   **M2: Hybrid Search kết hợp với M3: Reranking** là phần tạo ra cải thiện rõ nhất về Context Precision. Hybrid Search kết hợp BM25 và Dense Search giúp tận dụng cả keyword matching và semantic similarity. Sau đó, Cross-Encoder Reranker sắp xếp lại các kết quả để đưa các context liên quan hơn vào pipeline.

3. **Case study — 1 failure, Error Tree walkthrough:**  
   Chọn câu hỏi: **"Bao lâu phải đổi mật khẩu một lần?"**

   - Output đúng? → Chưa thể xác minh trực tiếp từ report vì report không lưu đầy đủ answer và ground truth.
   - Context đúng? → Cần kiểm tra pipeline logs hoặc retrieved chunks.
   - Worst metric → **Faithfulness = 0.3333**, là failure score thấp nhất.
   - Nguyên nhân có thể là context thiếu thông tin cần thiết hoặc LLM đã tự suy đoán câu trả lời ngoài context.
   - Fix → Siết prompt với yêu cầu chỉ sử dụng thông tin trong context, giảm temperature và thêm logging cho `question → retrieved contexts → reranked contexts → answer`.

4. **Next optimization nếu có thêm 1 giờ:**  
   - Tăng số context sau reranking từ top-3 lên top-5 và benchmark lại Context Recall.
   - Siết prompt bằng ràng buộc: **"Chỉ trả lời dựa trên Context. Nếu Context không đủ thông tin, hãy nói không tìm thấy thông tin."**
   - Giảm temperature để hạn chế hallucination.
   - Log toàn bộ pipeline để debug từng failure: query, BM25 results, dense results, RRF results, rerank scores và final context.
   - Tạo regression tests cho Bottom-5 failure questions.
   - Chạy lại `python main.py` và so sánh RAGAS scores trước/sau khi tối ưu.