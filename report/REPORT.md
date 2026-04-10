# Báo Cáo Lab 7: Embedding & Vector Store

**Họ và tên:** Lương Anh Tuấn

**MSV:** 2A202600113

**Nhóm:** C401-E5

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
> Hai vector có cosine similarity cao nghĩa là chúng gần như cùng hướng, tức là nội dung của chúng rất giống nhau về mặt ngữ nghĩa.

**Ví dụ HIGH similarity:**
- Sentence A: "Tôi thích học lập trình Python."
- Sentence B: "Lập trình Python là sở thích của tôi."
- Tại sao tương đồng: Hai câu diễn đạt cùng một ý về sở thích học lập trình Python, chỉ khác cấu trúc câu.

**Ví dụ LOW similarity:**
- Sentence A: "Tôi thích học lập trình Python."
- Sentence B: "Hôm nay trời mưa rất to."
- Tại sao khác: Hai câu nói về hai chủ đề hoàn toàn khác nhau, không liên quan về ngữ nghĩa.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Cosine similarity đo lường sự tương đồng về hướng giữa các vector, không bị ảnh hưởng bởi độ dài vector, phù hợp với text embeddings vốn chỉ quan tâm đến ý nghĩa chứ không phải độ lớn tuyệt đối.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> *Trình bày phép tính:*
Số chunks = ceil((10000 - 500) / (500 - 50)) + 1 = ceil(9500 / 450) + 1 ≈ 22.11 + 1 ≈ 24 chunks
Đáp án: 24 chunks

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> Số chunk sẽ tăng lên vì mỗi chunk mới sẽ bắt đầu gần chunk trước hơn. Overlap nhiều giúp giảm nguy cơ mất thông tin ở ranh giới các chunk, tăng khả năng retrieval chính xác.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** [Tài liệu bói toán]

**Tại sao nhóm chọn domain này?**
> Nhóm chọn domain tài liệu bói toán vì đây là lĩnh vực có dữ liệu đa dạng, giàu tính ngữ nghĩa và nhiều cách diễn giải khác nhau, phù hợp để thử nghiệm các kỹ thuật như RAG và semantic search. Ngoài ra, nội dung bói toán thường mang tính mở và linh hoạt, giúp đánh giá khả năng suy luận và tổng hợp thông tin của hệ thống AI.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 |Tử vi đẩu số tân biên |https://drive.google.com/file/d/1DTBH6jhq0ia_RenD3my6LDQdv0Bk_HQO/edit?fbclid=IwY2xjawRFgxBleHRuA2FlbQIxMABicmlkETF2Rkt4UlVBZjFGWW5tOWFxc3J0YwZhcHBfaWQQMjIyMDM5MTc4ODIwMDg5MgABHlzYdm64HUDC9ciLL_NPBp-u0xjvLPJBIn2RTxi2hPQ2KH0Em0bwvZHSOhY6_aem__0NJhHXdZdtRnGFPlVHYBw |36904 |? |

### Metadata Schema

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| source|String | "tuvi_sach_goc.pdf", "blog_tuvi_2023" | Xác định độ tin cậy của nguồn tin. Khi có mâu thuẫn kiến thức, hệ thống có thể ưu tiên dữ liệu từ nguồn chính thống (sách gốc) hơn là các bài blog.|
| category|Enum | "chinh_tinh", "phu_tinh", "ngu_hanh" |Cho phép thu hẹp phạm vi tìm kiếm. Nếu người dùng hỏi về ""Sao Tử Vi"", hệ thống chỉ tìm trong vùng dữ liệu đã tag là chinh_tinh, tránh nhiễu từ các đoạn văn nói về các sao khác. |
| update|DateTime | "2024-03-15", "2022-11-01" |Giúp hệ thống thực hiện Recency Bias. Trong các tài liệu nghiên cứu mới về tử vi hiện đại, thông tin cập nhật gần nhất thường có giá trị hiệu chỉnh cao hơn các bản dịch cũ. |
| access|Integer |	0 (public), 1 (internal), 2 (private) |Quản lý quyền truy cập dữ liệu (Security). Đảm bảo các ghi chú cá nhân hoặc dữ liệu khách hàng nhạy cảm không bị lộ ra khi người dùng phổ thông truy vấn hệ thống. |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên tài liệu `data/data.md` với `chunk_size=200`:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| data.md | FixedSizeChunker (`fixed_size`) | 246 | 199.81 | Yes |
| data.md | SentenceChunker (`by_sentences`) | 108 | 338.48 | Yes |

### Strategy Của Tôi

**Loại:** [RecursiveChunker]

**Mô tả cách hoạt động:**
> Thuật toán sử dụng đệ quy với thứ tự ưu tiên separator từ lớn đến nhỏ: `\n\n` → `\n` → `.` → dấu cách, và cuối cùng là fallback khi không còn separator phù hợp. Điều kiện dừng gồm hai trường hợp là khi đoạn văn có độ dài ≤ `chunk_size` hoặc khi không còn separator nào để tiếp tục tách thì trả về nguyên đoạn. Sau khi tách, các đoạn được chia thành `good_chunks` (đủ nhỏ) và `bad_chunks` (quá dài), sau đó chỉ tiếp tục đệ quy trên các `bad_chunks`.

Sau đó, chỉ tiếp tục đệ quy trên các `bad_chunks`.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Nội dung trong tài liệu bói toán thường được tổ chức thành nhiều phần với tiêu đề rõ ràng, nhưng độ dài mỗi đoạn lại không đồng nhất. Nếu áp dụng cách chia cố định, rất dễ làm mất mạch ý hoặc khiến nội dung bị cắt rời. Do đó, việc tách dựa trên cấu trúc văn bản là phù hợp hơn. RecursiveChunker khai thác đặc điểm này bằng cách ưu tiên chia theo ranh giới ngữ nghĩa trước, sau đó mới áp dụng các quy tắc kỹ thuật. Nhờ vậy, các đoạn được tạo ra khi truy xuất sẽ dễ hiểu hơn và giữ được tính liên kết, phục vụ tốt hơn cho việc trả lời các câu hỏi nghiệp vụ.


### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| data.md | **best baseline: FixedSizeChunker** | 246 | 199.81 | Medium |
| data.md | **của tôi: RecursiveChunker** | 211 | 172,24 | High |


### So Sánh Với Thành Viên Khác


| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | RecursiveChunker | 8.0 | Tối ưu theo domain, cân bằng giữa context và size | Cần tinh chỉnh nhiều, khó triển khai hơn baseline |
| Dũng | SentenceChunker | 7.0 | Giữ ngữ cảnh tốt, xử lý văn bản phức tạp | Chunk có thể không đồng đều, khó kiểm soát kích thước |
| Huy | FixedSizeChunker | 7.5 | Đơn giản, ổn định, dễ kiểm soát kích thước chunk | Dễ cắt ngang ý, mất ngữ cảnh ở đoạn dài |
| Sơn | RecursiveChunker | 8.5 | Giữ ngữ cảnh tốt, phù hợp tài liệu có cấu trúc phức tạp | Logic phức tạp hơn, phụ thuộc separator |
| Đạt | RecursiveChunker | 8.0 | Tạo chunk tự nhiên, dễ đọc | Chunk có thể quá dài hoặc quá ngắn tùy nội dung |
| Đăng | RecursiveChunker | 8.0 | Giữ ngữ cảnh tốt, chunk rõ nghĩa | Phụ thuộc vào cấu trúc văn bản, cần chọn separator phù hợp |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> Với tập tài liệu hiện tại, RecursiveChunker là lựa chọn phù hợp vì nội dung có nhiều đoạn dài và được tổ chức thành các mục rõ ràng. Thay vì chia theo độ dài cố định, phương pháp này ưu tiên tách dựa trên ngữ nghĩa, giúp giữ được sự liền mạch của nội dung. Nhờ đó, các chunk tạo ra phục vụ retrieval thường ổn định và dễ khai thác hơn. Trong các thử nghiệm baseline, cách này cũng thể hiện sự cân bằng tốt giữa kích thước chunk và độ liên kết thông tin.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Sử dụng regex để tách câu dựa trên các dấu câu như `.`, `!`, `?` kết hợp với khoảng trắng và chữ hoa đầu câu. Xử lý edge case như dấu chấm trong từ viết tắt hoặc số liệu để tránh tách nhầm.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Thuật toán chia đoạn văn bản dựa trên các dấu hiệu phân tách lớn trước (ví dụ: đoạn, tiêu đề), nếu không đủ nhỏ thì tiếp tục chia nhỏ hơn (câu, từ). Base case là khi đoạn nhỏ hơn hoặc bằng chunk_size hoặc không thể chia nhỏ hơn nữa.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> Lưu trữ embedding của từng chunk cùng metadata. Khi search, tính cosine similarity giữa embedding query và các embedding đã lưu, trả về các chunk có similarity cao nhất.

**`search_with_filter` + `delete_document`** — approach:
> Áp dụng filter lên metadata trước khi tính similarity để giảm số lượng so sánh. Xóa document bằng cách loại bỏ embedding và metadata tương ứng khỏi store.

### KnowledgeBaseAgent

**`answer`** — approach:
> Tạo prompt gồm câu hỏi người dùng và các context chunk liên quan (retrieved). Inject context vào prompt theo định dạng: [context]\n\n[question], giúp model trả lời sát ngữ cảnh.

### Test Results

```
=============================================================================== test session starts ===============================================================================
platform win32 -- Python 3.14.3, pytest-9.0.3, pluggy-1.6.0 -- C:\Users\nak11\python\aithucchien\2A202600113-LuongAnhTuan-Day07\.venv\Scripts\python.exe
cachedir: .pytest_cache
rootdir: C:\Users\nak11\python\aithucchien\2A202600113-LuongAnhTuan-Day07
plugins: anyio-4.13.0
collected 42 items                                                                                                                                                                 

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED                                                                                        [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED                                                                                                 [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED                                                                                          [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED                                                                                           [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED                                                                                                [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED                                                                                [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED                                                                                      [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED                                                                                       [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED                                                                                     [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED                                                                                                       [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED                                                                                       [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED                                                                                                  [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED                                                                                              [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED                                                                                                        [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED                                                                               [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED                                                                                   [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED                                                                             [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED                                                                                   [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED                                                                                                       [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED                                                                                         [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED                                                                                           [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED                                                                                                 [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED                                                                                      [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED                                                                                        [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED                                                                            [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED                                                                                         [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED                                                                                                  [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED                                                                                                 [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED                                                                                            [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED                                                                                        [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED                                                                                   [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED                                                                                       [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED                                                                                             [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED                                                                                       [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED                                                                    [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED                                                                                  [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED                                                                                 [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED                                                                     [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED                                                                                [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED                                                                         [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED                                                               [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED                                                                   [100%]

================================================================================ warnings summary =================================================================================
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size
  C:\Users\nak11\python\aithucchien\2A202600113-LuongAnhTuan-Day07\.venv\Lib\site-packages\chromadb\telemetry\opentelemetry\__init__.py:128: DeprecationWarning: 'asyncio.iscoroutinefunction' is deprecated and slated for removal in Python 3.16; use inspect.iscoroutinefunction() instead
    if asyncio.iscoroutinefunction(f):

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
========================================================================== 42 passed, 1 warning in 0.94s ==========================================================================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
|    1 | I am learning new skills.        | I am improving myself every day.  | High      |           0.9  | Đúng |
|    2 | I love playing chess.            | I am not a fan of board games.    | High      |           0.2  | Sai |
|    3 | My father is very tall.          | My mother is short.               | High      |           0.4  | Sai |
|    4 | I like the beach.                | I prefer the mountains.           | High      |           0.5  | Đúng |
|    5 | I enjoy photography.             | I am a professional photographer. | High      |           0.85 | Đúng |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Bất ngờ nhất là pair 2 vì hai câu trái nghĩa nhưng vẫn bị dự đoán giống nhau. Điều này cho thấy embeddings chủ yếu hiểu cùng chủ đề, chưa phân biệt tốt phủ định/mâu thuẫn.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Trong lý giải về Ngũ hành, quy luật Tương sinh diễn ra như thế nào|Quy luật Tương sinh giữa các hành bao gồm: *Kim sinh Thủy, Thủy sinh Mộc, Mộc sinh Hỏa, Hỏa sinh Thổ, và Thổ sinh Kim*. Ngược lại, quy luật Tương khắc là: Kim khắc Mộc, Mộc khắc Thổ, Thổ khắc Thủy, Thủy khắc Hỏa, và Hỏa khắc Kim |
| 2 |Làm thế nào để tìm được Bản Mệnh thuộc hành nào trong Ngũ hành? |Để tìm Bản Mệnh, người xem số cần rõ tuổi của mình ở hai hàng *Can và Chi*, sau đó tra bảng để xác định mình thuộc hành nào trong Ngũ hành (Kim, Mộc, Thủy, Hỏa, Thổ). Có tất cả *Thập Thiên Can* (Giáp, Ất, Bính, Đinh, Mậu, Kỷ, Canh, Tân, Nhâm, Qúy) phối hợp với các Địa chi. |
| 3 |Quy tắc đổi giờ đồng hồ sang giờ hàng Chi trong Tử Vi là gì? |Một ngày có 24 giờ đồng hồ và cứ *hai giờ đồng hồ tương ứng với một giờ hàng Chi*. Ví dụ: giờ Tý bắt đầu từ 23 giờ đến 1 giờ sáng, giờ Sửu từ 1 giờ đến 3 giờ sáng, và tiếp tục như vậy cho đến hết 12 ch |
| 4 |Chùm sao thuộc Tử Vi tinh hệ bao gồm những sao nào? |Chùm sao này gồm có 5 sao: *Tử Vi, Liêm Trinh, Thiên Đồng, Vũ Khúc và Thiên Cơ*. Việc an các sao này dựa trên Cục và ngày sinh của mỗi ngườ |
| 5 |Một lá số Tử Vi được chia làm bao nhiêu ô và tên gọi của các ô này dựa trên quy tắc nào? |Lá số được chia làm *12 ô*, mỗi ô gọi là một cung. Tên riêng của mỗi cung được gọi theo *Thập Nhị Địa Chi*, bao gồm: Tý, Sửu, Dần, Mão, Thìn, Tỵ, Ngọ, Mùi, Thân, Dậu, Tuất, Hợi. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|---|---|---:|---|---|
| 1 | Trong lý giải về Ngũ hành, quy luật Tương sinh diễn ra như thế nào | Chunk mô tả đầy đủ vòng Tương sinh (Kim → Thủy → Mộc → Hỏa → Thổ → Kim) và có nhắc thêm mối quan hệ Tương khắc | 0.92 | Có | Agent đã trình bày chính xác chuỗi Tương sinh và đồng thời bổ sung thêm phần Tương khắc một cách phù hợp. |
| 2 | Làm thế nào để tìm được Bản Mệnh thuộc hành nào trong Ngũ hành? | Chunk trình bày cách xác định Bản Mệnh thông qua Can–Chi và bảng tra tương ứng với ngũ hành | 0.86 | Có | Agent đã mô tả đúng quy trình: trước hết xác định Can–Chi, sau đó suy ra hành tương ứng. |
| 3 | Quy tắc đổi giờ đồng hồ sang giờ hàng Chi trong Tử Vi là gì? | Chunk giải thích quy ước 2 giờ dương lịch tương ứng 1 giờ Chi, kèm ví dụ minh họa cụ thể | 0.95 | Có | Agent đã vận dụng đúng quy tắc và cung cấp ví dụ chính xác. |
| 4 | Chùm sao thuộc Tử Vi tinh hệ bao gồm những sao nào? | Chunk liệt kê trọn vẹn 5 sao trong tinh hệ Tử Vi | 0.87 | Có | Agent đã đưa ra câu trả lời đầy đủ và nêu đúng tên các sao. |
| 5 | Một lá số Tử Vi được chia làm bao nhiêu ô và tên gọi của các ô này dựa trên quy tắc nào? | Chunk nêu rõ lá số gồm 12 cung và cách đặt tên dựa trên Thập Nhị Địa Chi | 0.88 | Có | Agent đã trả lời chính xác về số lượng cung và tuân thủ đúng nguyên tắc đặt tên. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
>  Tôi nhận ra tầm quan trọng của việc gắn metadata (như source, category…) ngay từ giai đoạn đầu, giúp hệ thống retrieval ổn định hơn khi quy mô dữ liệu tăng. Bên cạnh đó, việc đánh giá chunk dựa trên mức độ “đủ ý” thay vì chỉ dựa vào độ dài cũng rất hữu ích. Nhờ vậy, câu trả lời của agent giữ được ngữ cảnh tốt hơn, giúp tôi hiểu rõ hơn mối liên hệ giữa chunking và chất lượng đầu ra.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Nhóm khác xây dựng quy trình đánh giá khá rõ ràng với bộ câu hỏi bám sát bài toán và đáp án ngắn gọn, dễ đối chiếu. Họ cũng thử nghiệm nhiều cấu hình như chunk_size và overlap trên cùng một tập câu hỏi, từ đó làm nổi bật ưu và nhược điểm của từng phương án. Điều này cho thấy việc đánh giá nên kết hợp cả số liệu định lượng và ví dụ minh họa cụ thể.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> Nếu có cơ hội làm lại, tôi sẽ tập trung chuẩn bị dữ liệu kỹ lưỡng hơn ngay từ đầu: làm sạch, chuẩn hóa và gắn metadata trước khi thực hiện embedding. Ngoài ra, tôi sẽ thiết kế bộ câu hỏi theo nhiều mức độ khó để dễ dàng выяв điểm yếu của từng phương pháp chunking. Cuối cùng, tôi sẽ lưu lại log retrieval theo từng phiên bản dữ liệu nhằm theo dõi và so sánh hiệu quả cải thiện một cách khách quan.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 15 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 10 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **90 / 90** |

