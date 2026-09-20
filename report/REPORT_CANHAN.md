# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Thị Chinh
**Nhóm:** Gì cũng được
**Ngày:** 20/9/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
Độ tương tự cosine cao nghĩa là hai vector gần giống với nhau về hướng, tức là nội dung của chúng tương tự nhau.

**Ví dụ có độ tương tự CAO:**
- Câu A: "Học phí học kỳ 1 năm học 2026-2027 là bao nhiêu?"
- Câu B: "Mức học phí cho năm học 2026-2027 là bao nhiêu?"
- Tại sao tương đồng: Cả hai câu đều hỏi về mức học phí cho năm học 2026-2027.

**Ví dụ có độ tương tự THẤP:**
- Câu A: "Học phí học kỳ 1 năm học 2026-2027 là bao nhiêu?"
- Câu B: "Em muốn xin giấy giới thiệu để đi khám bệnh ở Bệnh viện Đại học Y dược?"
- Tại sao khác: Câu A hỏi về học phí, câu B hỏi về giấy giới thiệu đi khám bệnh, hoàn toàn khác về nội dung.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
Trong khi khoảng cách Euclid tính toán đường chim bay giữa hai điểm trong không gian nhiều chiều, độ tương tự cosine đo lường góc giữa hai vector. Đối với text embeddings, độ lớn của vector thường phụ thuộc vào độ dài câu (câu dài có vector lớn hơn). Nếu dùng khoảng cách Euclid, một câu ngắn nhưng có cùng nội dung sẽ bị coi là xa hơn một câu dài có cùng nội dung, dẫn đến kết quả tìm kiếm sai lệch. Ngược lại, độ tương tự cosine chỉ quan tâm đến hướng của vector, thể hiện chủ đề và ý nghĩa ngữ nghĩa, do đó phù hợp hơn để so sánh nội dung văn bản.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
Chunk đầu 500 ký tự, mỗi chunk tiếp theo dịch chuyển thêm 450 ký tự. Số chunks = 100000 / 450 + 1 = 23 chunks

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
Số chunks = 100000 / 400 + 1 = 26 chunks
Độ chồng chéo nhiều hơn giúp tăng khả năng truy xuất ngữ nghĩa, vì mỗi chunk chứa nhiều thông tin hơn và có thể bao phủ nhiều ngữ cảnh hơn.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
Chia văn bản thành các câu atomic, gom 3 câu lại thành một chunk, sử dụng biểu thức chính quy `[^.!?]+[.!?]?` để phát hiện câu. Nó tìm kiếm các ký tự không phải dấu chấm, dấu chấm than hoặc dấu chấm hỏi, theo sau là một trong các dấu câu này hoặc không có dấu câu nào. Các trường hợp ngoại lệ được xử lý bao gồm các khoảng trắng thừa ở đầu hoặc cuối chunk, cũng như các chunk rỗng được loại bỏ.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
Thuật toán hoạt động đệ quy, chia nhỏ văn bản thành các đoạn ngắn hơn bằng các separators ưu tiên. Base case là khi độ dài văn bản nhỏ hơn `chunk_size `, hoặc khi không thể chia nhỏ thêm.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
Khi thêm tài liệu (add_documents), mỗi Document được chuyển thành một bản ghi (record) gồm id, content, metadata và trường vector sinh ra từ hàm embedding self._embedding_fn, sau đó lưu vào danh sách trong bộ nhớ (in-memory list) hoặc ChromaDB. Khi tìm kiếm (search), câu truy vấn được chuyển thành vector, sau đó tính độ tương tự Cosine (tích vô hướng _dot trên các vector đã chuẩn hóa) với từng bản ghi trong kho, sắp xếp điểm số (score) giảm dần và lấy ra top_k kết quả cao nhất.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
Pre-filtering: duyệt để chọn ra các chunk thoả mãn điều kiện trong `metadata_filter`, sau đo mới tính độ tương tự cosine trên tập này để lấy top-k.
Với hàm xoá `delete_document`, hệ thống lọc bỏ toàn bộ các chunk có `id == doc_id` hoặc `metadata['doc_id] == doc_id`

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
Tác tử nhận câu hỏi từ người dùng, gọi EmbeddingStore.search để truy xuất các đoạn văn bản liên quan nhất làm bằng chứng ngữ cảnh (context). Sau đó, ngữ cảnh này được tiêm trực tiếp (inject context) vào prompt có cấu trúc cùng câu hỏi, yêu cầu mô hình chỉ căn cứ vào thông tin được cung cấp để tổng hợp câu trả lời chính xác, tránh hiện tượng ảo giác (hallucination).
---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```bash
# Dán kết quả (output) của: pytest tests/ -v
======================================= test session starts =======================================
platform darwin -- Python 3.11.16, pytest-9.1.1, pluggy-1.6.0 -- /opt/miniconda3/envs/k4-day07/bin/python3.11
cachedir: .pytest_cache
rootdir: /Users/hihi/Documents/aitc/K4-DAY07-NguyenThiChinh-2A202602876
plugins: anyio-4.15.1
collected 42 items                                                                                

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED       [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED                [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED         [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED          [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED               [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED     [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED      [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED    [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED                      [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED      [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED                 [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED             [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED                       [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED  [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED  [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED                      [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED        [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED          [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED                [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED     [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED       [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED        [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED                 [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED                [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED           [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED       [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED  [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED      [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED            [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED      [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]

======================================= 42 passed in 0.04s ========================================
```

**Số lượng bài test vượt qua (pass):** 42/42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Học phí chương trình đào tạo đại học 2025 - 2026 | Mức thu tiền học của sinh viên hệ chính quy năm 2025-2026 | cao | 0.72 | Đúng |
| 2 | Hướng dẫn nộp học phí trực tuyến qua cổng thanh toán| Quy trình quét mã QR trên hệ thống ERP để đóng tiền học | cao | 0.46 | Đúng |
| 3 | Thời hạn nộp học phí học kỳ 2 năm học 2025-2026 | Quy định về việc xét hoãn, miễn, giảm học phí | cao | 0.52 | Đúng |
| 4 | Sinh viên đã hoàn thành đầy đủ nghĩa vụ nộp học phí | Sinh viên nợ học phí bị đình chỉ thi và xóa tên khỏi danh sách | thấp | 0.57 | Sai |
| 5 | Định mức học phí đào tạo đại học năm học 2025-2026 | Điều chỉnh học phí sau đại học, giảm 5% | cao | 0.63 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
Kết quả bất ngờ nhất là cặp 4. Hai câu có nghĩa trái ngược nhưng điểm số lại tương đối cao, cho thấy cách embeddings biểu diễn ý nghĩa vẫn còn hạn chế, chưa phân biệt được ngữ nghĩa trái ngược.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân của bạn trong gói `src`. **5 câu hỏi này phải trùng với các thành viên cùng nhóm** (xem `REPORT_NHOM.md`).

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Trường Đại học Công Nghệ gia hạn nộp học phí học kỳ II năm học 2025-2026 đến khi nào? | Thông báo gia hạn thời gian nộp học phí HKII năm học 2025-2026 cho sinh viên có tên trong danh sách đến hết ngày 26/5/2026. | 0.7202 | Có | Ngày 26/5/2026 |
| 2 | Hướng dẫn đóng học phí học kỳ II năm 2025-2026 qua hệ thống ERP của sinh viên USTH | Phương thức đóng học phí Sinh viên nộp học phí bằng hình thức chuyển khoản qua mã QR hiển thị... (gồm 4 bước quy trình). | 0.7039 | Có | Đăng nhập hệ thống ERP (erp.usth.edu.vn/students), chọn mục Học phí, kiểm tra số tiền và quét mã QR để chuyển trạng thái "Đã đóng".|
| 3 | Đối với các khoá 2021 trở về trước thì học bằng kép ở Trường Đại học Công Nghệ năm học 2024-2025 hết bao nhiêu tiền 1 tín chỉ? | # Học phí NEU 2025-2026: chuẩn, CLC, tiên tiến | Khoa CNTT NEU | Khoa Công nghệ thông tin Học p... | 0.6431 | Có | Mức học phí chương trình đào tạo bằng kép là 450.000 đồng/tín chỉ. |
| 4 | Chương trình định hướng ứng dụng POHE của NEU năm học 2025-2026 có học phí bao nhiêu? | # Học phí NEU 2025-2026: chuẩn, CLC, tiên tiến | Khoa CNTT NEU | Khoa Công nghệ thông tin Học p... | 0.6887| Có | 42.000.000 đồng/năm học |
| 5 | Theo lộ trình được duyệt thì mức thu học phí đối với sinh viên quốc tế là bao nhiêu? | ### 1.2. Đối với sinh viên quốc tế (không phải diện hiệp định) Mức thu: 45.000.000đ/năm học/SV.... | 0.6262 | Có | 45.000.000đ/năm học/SV|

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 5 / 5


**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
Tôi học được cách kết hợp các chiến lược lại với nhau để tăng hiệu quả. Tuy nhiên, vẫn cần áp dụng các chiến lược một cách linh hoạt và phù hợp với từng trường hợp cụ thể. 
---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 27 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 4 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 10/ 10 |
| **Tổng phần cá nhân** | ** / 60** |
