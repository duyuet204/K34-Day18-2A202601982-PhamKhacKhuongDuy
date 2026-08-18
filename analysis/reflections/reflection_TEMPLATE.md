# Individual Reflection — Lab 18

**Tên:** Phạm Khắc Khương Duy 
**Module phụ trách:** M1 / M2 / M3 / M4

---

## 1. Đóng góp kỹ thuật

- **Module đã implement:**
  - **M1: Advanced Chunking** — implement Semantic Chunking, Hierarchical Chunking và Structure-Aware Chunking.
  - **M2: Hybrid Search** — implement Vietnamese word segmentation, BM25 Search, Dense Search với Qdrant và Reciprocal Rank Fusion (RRF).
  - **M3: Reranking** — implement Cross-Encoder Reranker sử dụng `sentence_transformers.CrossEncoder`.
  - **M4: Evaluation** — implement RAGAS Evaluation và Failure Analysis theo Diagnostic/Error Tree.
  - Ngoài ra, đã chạy toàn bộ pipeline bằng `python main.py`, tạo `ragas_report.json` và so sánh với Naive Baseline.

- **Các hàm/class chính đã viết:**
  - M1:
    - `chunk_semantic()`
    - `chunk_hierarchical()`
    - `chunk_structure_aware()`
  - M2:
    - `segment_vietnamese()`
    - `BM25Search.index()`
    - `BM25Search.search()`
    - `DenseSearch.index()`
    - `DenseSearch.search()`
    - `reciprocal_rank_fusion()`
  - M3:
    - `CrossEncoderReranker._load_model()`
    - `CrossEncoderReranker.rerank()`
  - M4:
    - `evaluate_ragas()`
    - `failure_analysis()`

- **Số tests pass:**  
  - M1: 8/8
  - M2: 5/5
  - M3: 5/5
  - M4: 4/4
  - Tổng cộng: **22/22 tests**

## 2. Kiến thức học được

- **Khái niệm mới nhất:**
  - Semantic Chunking: sử dụng embedding và cosine similarity để nhóm các câu có nội dung tương tự thay vì chỉ cắt theo số ký tự.
  - Hierarchical Chunking: retrieve child chunk để tăng precision nhưng có thể sử dụng parent chunk để cung cấp context rộng hơn cho LLM.
  - Structure-Aware Chunking: giữ cấu trúc logic của Markdown như header, section, table và code block.
  - Hybrid Search: kết hợp BM25 và Dense Search để tận dụng cả keyword matching và semantic similarity.
  - RRF (Reciprocal Rank Fusion): gộp nhiều ranked lists mà không phụ thuộc trực tiếp vào việc các hệ thống có score cùng thang đo.
  - Cross-Encoder Reranking: đánh giá trực tiếp cặp `(query, document)` để sắp xếp lại các kết quả retrieval.
  - RAGAS: đánh giá RAG dựa trên Faithfulness, Answer Relevancy, Context Precision và Context Recall.

- **Điều bất ngờ nhất:**
  
  Điều bất ngờ nhất là một Production RAG Pipeline phức tạp hơn không nhất thiết sẽ tốt hơn Naive Baseline ở tất cả metrics. Trong lần chạy thực tế, Context Precision tăng từ **0.9250 lên 0.9500**, nhưng Faithfulness, Answer Relevancy và Context Recall lại giảm.
  
  Kết quả này cho thấy việc retrieval được các context chính xác hơn chưa đủ để đảm bảo câu trả lời cuối cùng tốt hơn. Reranking có thể loại bỏ một số evidence cần thiết, hoặc LLM vẫn có thể hallucinate nếu prompt chưa được ràng buộc đủ chặt.

- **Kết nối với bài giảng (slide nào):**
  - Phần **Chunking Strategies** kết nối với M1, đặc biệt là Semantic, Hierarchical và Structure-Aware Chunking.
  - Phần **Retrieval / Hybrid Search** kết nối với M2: BM25, Dense Retrieval và Reciprocal Rank Fusion.
  - Phần **Reranking** kết nối với M3: sử dụng Cross-Encoder để sắp xếp lại top-k documents.
  - Phần **RAG Evaluation** kết nối với M4: sử dụng RAGAS metrics và Error Tree để phân tích failure.
  - Các nội dung này giúp hiểu RAG không chỉ là `Embedding → Vector DB → LLM`, mà là một pipeline gồm nhiều bước cần được tối ưu và đánh giá riêng.

## 3. Khó khăn & Cách giải quyết

- **Khó khăn lớn nhất:**
  
  Khó khăn lớn nhất là vấn đề môi trường Python và dependency. Ban đầu khi chạy test gặp lỗi:

  - `ModuleNotFoundError: No module named 'sentence_transformers'`
  - `ModuleNotFoundError: No module named 'pypdf'`
  - Sau đó khi chạy bằng `python -m pytest`, lại gặp:
    - `No module named pytest`

  Mặc dù một số package đã được `pip install`, Python interpreter đang chạy test có thể không cùng môi trường với interpreter đã cài package.

  Ngoài ra, các model như `all-MiniLM-L6-v2` và `BAAI/bge-reranker-v2-m3` cần được tải lần đầu, dẫn đến thời gian chạy lâu và phụ thuộc vào mạng.

- **Cách giải quyết:**
  
  - Kiểm tra virtual environment đang được kích hoạt.
  - Dùng `python -m pip install ...` thay vì chỉ dùng `pip install ...` để đảm bảo package được cài vào đúng Python interpreter.
  - Cài đầy đủ các dependency cần thiết như:
    ```bash
    python -m pip install pytest pypdf sentence-transformers
    ```
  - Chạy test bằng:
    ```bash
    python -m pytest tests/test_m1.py -v
    ```
    thay vì gọi trực tiếp `pytest`.
  - Kiểm tra logic từng module bằng test hẹp trước khi chạy toàn bộ pipeline.
  - Sau khi từng module hoạt động, chạy entrypoint chính:
    ```bash
    python main.py
    ```
  - Dựa trên `ragas_report.json` để phân tích Bottom-5 failures thay vì chỉ nhìn vào aggregate metrics.

- **Thời gian debug:**
  
  Phần lớn thời gian debug tập trung vào:
  - Python virtual environment và package dependencies.
  - Tải embedding model và reranker model.
  - Kiểm tra sự khác biệt giữa API versions, đặc biệt với Qdrant.
  - Phân tích lý do Production Pipeline có Context Precision tốt hơn nhưng các metrics generation lại thấp hơn baseline.

  Thời gian chạy toàn bộ pipeline khoảng **1021.6 giây**, cho thấy evaluation và các model-based components có chi phí thời gian đáng kể.

## 4. Nếu làm lại

- **Sẽ làm khác điều gì:**
  
  Nếu làm lại, tôi sẽ xây dựng pipeline theo hướng đo lường từng bước rõ ràng hơn ngay từ đầu.

  Cụ thể:
  - Log `query → BM25 results → Dense results → RRF results → Rerank results → final context → final answer`.
  - Đánh giá riêng từng module thay vì chỉ đánh giá kết quả cuối cùng.
  - Benchmark nhiều cấu hình `top_k`, ví dụ top-3, top-5 và top-10 sau reranking.
  - Tạo regression test cho các câu hỏi có score thấp nhất.
  - Siết prompt để LLM chỉ sử dụng thông tin trong context và trả lời rõ khi không có đủ evidence.
  - Tách retrieval failure và generation failure bằng Error Tree thay vì giả định lỗi nằm ở một module duy nhất.

- **Module nào muốn thử tiếp:**
  
  Tôi muốn thử tiếp **M2 và M3**, đặc biệt là tối ưu Hybrid Search và Reranking.

  Một số hướng muốn thử:
  - Thay đổi trọng số hoặc cách fusion giữa BM25 và Dense Search.
  - So sánh RRF với weighted score fusion.
  - Thử top-5 thay vì top-3 sau reranking để cải thiện Context Recall.
  - Thử embedding model phù hợp hơn với tiếng Việt.
  - So sánh Cross-Encoder với lightweight reranker như FlashRank.
  - Thêm metadata filtering vào retrieval.
  - Đánh giá trade-off giữa latency và chất lượng RAG.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4/5 |
| Code quality | 4/5 |
| Teamwork | 5/5 |
| Problem solving | 4/5 |

### Nhận xét tự đánh giá

Tôi đánh giá mức độ hiểu bài giảng là **4/5** vì đã có thể implement các kỹ thuật chính của Production RAG và giải thích được vai trò của từng module trong pipeline. Tuy nhiên, tôi vẫn muốn tìm hiểu sâu hơn về cách tối ưu retrieval và đánh giá trade-off giữa các metrics.

Code quality được đánh giá **4/5** vì các module được tách thành các class và function tương đối rõ ràng, có fallback và xử lý một số lỗi dependency. Tuy nhiên, vẫn có thể cải thiện bằng cách thêm type checking, logging và test coverage cho các edge cases.

Teamwork được đánh giá **5/5** vì dù thực hiện bài cá nhân, tôi đã hoàn thành các phần được yêu cầu và tổng hợp kết quả theo format chung của Lab.

Problem solving được đánh giá **4/5** vì đã xử lý được các lỗi thực tế liên quan đến virtual environment, package dependency, model loading và evaluation. Kinh nghiệm quan trọng nhất là không chỉ sửa lỗi để code chạy, mà cần kiểm tra kết quả cuối cùng bằng metrics và failure analysis để xác định pipeline thực sự có cải thiện hay không.