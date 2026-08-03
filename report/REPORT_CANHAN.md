# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Phạm Sỹ Đức
**Nhóm:** C52
**Ngày:** 03/08/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

### Tóm tắt kết quả

- Hoàn thành toàn bộ phần lập trình cốt lõi; kết quả kiểm thử ghi nhận `42/42` test đạt.
- Sử dụng local embedding `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` thay cho mock embedding khi đánh giá retrieval.
- Đánh giá trên 8 tài liệu công khai của VinUni và 5 truy vấn benchmark chung của nhóm.
- Chiến lược cuối cùng dùng `RecursiveChunker(chunk_size=1600)`, `top_k=3` và metadata pre-filter theo ý định truy vấn; kết quả đạt `5/5` top-3 hit, đồng thời bao phủ đủ `19/19` mục bằng chứng bắt buộc.

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (high cosine similarity) nghĩa là gì?**

Độ tương tự cosine đo mức độ cùng hướng của hai vector theo công thức `cos(a,b) = (a·b) / (||a|| ||b||)`. Giá trị gần `1` cho biết hai vector có hướng gần nhau và, trong cùng một không gian embedding, thường biểu diễn nội dung tương đồng. Giá trị gần `0` cho biết hai vector gần vuông góc; giá trị âm cho biết góc giữa chúng lớn hơn 90°. Giá trị âm không tự động có nghĩa hai câu là từ trái nghĩa, vì cách diễn giải còn phụ thuộc mô hình và dữ liệu huấn luyện.

**Ví dụ có độ tương tự cao:**

- Câu A: Sinh viên có thể đăng ký môn học trực tuyến.
- Câu B: Người học được phép ghi danh học phần qua hệ thống online.
- Tại sao tương đồng: Hai câu dùng từ khác nhau nhưng đều nói về việc sinh viên đăng ký học phần trực tuyến.

**Ví dụ có độ tương tự thấp:**

- Câu A: Sinh viên cần đóng học phí trước ngày quy định.
- Câu B: Thư viện mở cửa từ 8 giờ sáng.
- Tại sao khác: Một câu nói về học phí, câu còn lại nói về thời gian hoạt động của thư viện.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**

Cosine similarity tập trung vào hướng của vector nên không bị độ lớn của vector chi phối. Điều này phù hợp khi mục tiêu là so sánh khuynh hướng ngữ nghĩa; trong khi đó Euclidean distance chịu ảnh hưởng bởi cả hướng lẫn độ lớn. Với các vector đã chuẩn hóa L2 như đầu ra của `LocalEmbedder`, dot product bằng cosine similarity, nhờ đó store có thể xếp hạng bằng phép nhân vô hướng đơn giản. Cosine không phải lúc nào cũng tốt hơn Euclidean; lựa chọn metric vẫn phải phù hợp với cách mô hình embedding được huấn luyện và chuẩn hóa.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**

Áp dụng công thức `N = ceil((L - overlap) / (chunk_size - overlap))`:

- Bước dịch: `step = 500 - 50 = 450` ký tự.
- Số chunk: `N = ceil((10,000 - 50) / 450) = ceil(22.11...) = 23`.

**Đáp án: 23 chunks.**

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**

Khi overlap tăng lên 100, bước dịch còn `500 - 100 = 400`, nên `N = ceil((10,000 - 100) / 400) = ceil(24.75) = 25` chunks. Overlap lớn hơn giúp giữ ngữ cảnh ở ranh giới chunk và giảm nguy cơ tách mất ý. Đánh đổi là nội dung bị lặp nhiều hơn, số vector tăng, đồng thời tốn thêm bộ nhớ và thời gian truy vấn.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — Tôi kiểm tra đầu vào rỗng trước để trả về `[]`, sau đó dùng regex `(?<=[.!?])\s+|\n+` để tách tại khoảng trắng sau dấu kết thúc câu hoặc tại ký tự xuống dòng. Các phần rỗng và khoảng trắng thừa được loại bỏ. Cuối cùng, tôi duyệt danh sách câu theo bước `max_sentences_per_chunk`, ghép từng nhóm thành một chuỗi và bảo đảm mỗi chunk không vượt quá số câu cấu hình.

**`RecursiveChunker.chunk` / `_split`** — Thuật toán thử lần lượt các separator `"\n\n"`, `"\n"`, `". "`, `" "` và `""`, ưu tiên giữ nguyên đoạn văn và câu trước khi phải cắt theo ký tự. Base case xảy ra khi đoạn hiện tại có độ dài không vượt quá `chunk_size`, lúc đó đoạn được trả về ngay. Nếu separator hiện tại không xuất hiện, hàm chuyển sang separator tiếp theo; nếu không còn separator hoặc gặp separator rỗng, hàm fallback sang cắt cố định để bảo đảm không lặp vô hạn và luôn tạo được kết quả.

Các phần nhỏ được ghép lại khi tổng độ dài vẫn nằm trong giới hạn. Phần vượt quá kích thước được xử lý đệ quy bằng separator có mức ưu tiên thấp hơn. Cách làm này giúp chunk giữ cấu trúc tự nhiên tốt hơn fixed-size chunking nhưng vẫn xử lý được văn bản dài không có dấu phân cách.

**`compute_similarity`** — Tôi tính dot product và chuẩn L2 của hai vector, sau đó chia dot product cho tích hai chuẩn. Nếu một trong hai vector có độ lớn bằng 0, hàm trả về `0.0` để tránh phép chia cho 0. Cách triển khai này bao phủ được các trường hợp vector giống nhau, trực giao, đối hướng và zero vector.

**`ChunkingStrategyComparator.compare`** — Tôi chạy cùng một văn bản qua ba chiến lược `fixed_size`, `by_sentences` và `recursive`, rồi trả về cho từng chiến lược: số chunk, độ dài trung bình và danh sách chunk. Kết quả này tạo một baseline chung để so sánh định lượng, đồng thời cho phép kiểm tra định tính mức độ mạch lạc của từng chunk.

### Lớp EmbeddingStore

**`add_documents` + `search`** — Tôi triển khai store trong bộ nhớ để đáp ứng phần bắt buộc của lab. Mỗi `Document` được chuẩn hóa thành record gồm ID duy nhất, nội dung, bản sao metadata và embedding; `doc_id` gốc được giữ trong metadata để truy vết và xóa tất cả chunk cùng tài liệu. Cách sao chép metadata tránh làm thay đổi đối tượng đầu vào.

Khi tìm kiếm, query được chuyển thành embedding bằng cùng backend với tài liệu. Store tính dot product giữa query embedding và từng record, tạo kết quả gồm `id`, `content`, `metadata`, `score`, sau đó sắp xếp score giảm dần và trả về tối đa `top_k`. Local embedder chuẩn hóa vector nên dot product bằng cosine similarity.

**`search_with_filter` + `delete_document`** — `search_with_filter` thực hiện pre-filter: một record chỉ được giữ khi tất cả cặp khóa–giá trị trong `metadata_filter` khớp. Sau đó hàm embedding query và xếp hạng tập ứng viên đã lọc. Cách này đặc biệt hữu ích để giới hạn đúng `audience`, `category` hoặc học kỳ, đồng thời tránh để tài liệu gần nghĩa nhưng sai đối tượng lấn át kết quả.

`delete_document` ghi nhận kích thước ban đầu rồi tạo lại `_store`, loại mọi record có `metadata["doc_id"]` trùng với tài liệu cần xóa; điều kiện ID trực tiếp được giữ như một fallback. Hàm trả về `True` khi số record giảm và `False` nếu không tìm thấy tài liệu.

### Tác tử KnowledgeBaseAgent

**`answer`** — Phương thức này triển khai ba bước của RAG: retrieve các chunk liên quan, ghép nội dung thành context, rồi gọi `llm_fn`. Prompt tách rõ phần hướng dẫn, `Context`, `Question` và vị trí `Answer`, đồng thời yêu cầu mô hình chỉ dùng dữ liệu được cung cấp và báo rằng thông tin không có sẵn nếu context không chứa câu trả lời. Việc truyền `llm_fn` từ bên ngoài giúp agent dễ kiểm thử và không phụ thuộc vào một nhà cung cấp LLM cụ thể.

### Các quyết định về độ tin cậy

- Các embedding tài liệu và truy vấn luôn được tạo bằng cùng một backend.
- Metadata được sao chép trước khi bổ sung `doc_id`, tránh side effect lên `Document` đầu vào.
- Các trường hợp rỗng, `top_k <= 0`, zero vector và separator không khả dụng đều có đường xử lý an toàn.
- Mock embedding chỉ phục vụ unit test; benchmark retrieval dùng local multilingual embedding có chuẩn hóa L2.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

### Kết quả kiểm thử

```text
collected 42 items
42 passed
```

**Kết quả:** `42/42` test đạt, `0` test thất bại.

Lệnh kiểm thử:

```bash
pytest tests/ -v
```

Các nhóm hành vi đã được kiểm tra gồm:

- Cấu trúc project và các interface bắt buộc.
- Fixed-size, sentence-based và recursive chunking, bao gồm input rỗng và fallback separator.
- Cosine similarity cho vector giống nhau, trực giao, đối hướng và zero vector.
- Thêm, đếm, tìm kiếm, sắp xếp score, lọc metadata và xóa tài liệu trong `EmbeddingStore`.
- Luồng trả lời của `KnowledgeBaseAgent`.

Ngoài kết quả chạy trên, mình đã đối chiếu file `tests/test_solution.py`: bộ kiểm thử có đúng 42 test case, bao phủ các nhóm hành vi nêu trên.

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

Tôi ghi lại dự đoán **trước khi** đo, dựa trên mức độ tương đồng ngữ nghĩa. Sau đó, tôi dùng `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` để tạo vector và gọi `compute_similarity()` để tính cosine similarity. Quy ước phân loại cho thí nghiệm nhỏ này là: score `>= 0.60` được xem là **cao**, score `< 0.60` được xem là **thấp**; đây là ngưỡng thực nghiệm của bài, không phải ngưỡng phổ quát cho mọi embedding model.

| Cặp | Câu A | Câu B | Dự đoán | Cosine score | Phân loại theo ngưỡng | Dự đoán đúng? |
|---:|---|---|:---:|---:|:---:|:---:|
| 1 | Students register for courses through the online portal. | Course enrollment is completed using the student portal. | Cao | 0.760827 | Cao | Có |
| 2 | The final add/drop deadline is July 11, 2026. | Students may change their Summer 2026 courses until July 11, 2026. | Cao | 0.654121 | Cao | Có |
| 3 | A withdrawn course receives a W grade. | The cafeteria serves lunch from Monday to Friday. | Thấp | 0.082864 | Thấp | Có |
| 4 | The library opens at eight in the morning. | Unmet prerequisites prevent course registration. | Thấp | 0.058327 | Thấp | Có |
| 5 | Students must pay tuition before the stated deadline. | Tuition fees are due by the announced payment date. | Cao | 0.607986 | Cao | Có |

**Độ chính xác dự đoán:** `5/5 = 100%` theo ngưỡng đã công bố.

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**

Cặp 5 bất ngờ nhất: hai câu diễn đạt gần như cùng một yêu cầu về hạn thanh toán học phí nhưng score chỉ vừa vượt ngưỡng (`0.607986`), thấp hơn cặp 1 (`0.760827`). Kết quả cho thấy cosine score không phải xác suất hai câu cùng nghĩa và không nên dùng một ngưỡng cố định cho mọi miền dữ liệu. Cách diễn đạt, từ vựng, cấu trúc câu và đặc tính huấn luyện của model đều có thể ảnh hưởng vị trí vector; vì vậy ngưỡng cần được hiệu chỉnh bằng dữ liệu đánh giá thực tế.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

### Thiết lập và giao thức đánh giá

- Corpus: 8 tài liệu công khai về đăng ký học phần và quy định học vụ VinUni.
- Embedding backend: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`.
- Chiến lược chunking: `RecursiveChunker(chunk_size=1600)`.
- Số kết quả: `top_k=3`.
- Bộ câu hỏi và gold answer: `benchmarks/vinuni_course_registration.json`.
- Tiêu chí hit: top-3 chứa ít nhất một chunk có `doc_id` thuộc `relevant_doc_ids` của câu hỏi.
- Tiêu chí đủ bằng chứng: context truy xuất chứa toàn bộ các mục trong `required_evidence`.
- Q1 lọc `{"audience": "student"}` để đáp ứng yêu cầu K3.
- Q4 lọc `{"category": "registration-system-guide"}` vì câu hỏi hỏi trạng thái trên hệ thống đăng ký.
- Q5 lọc `{"category": "academic-request-process"}` vì câu hỏi hỏi quy trình gửi yêu cầu học vụ.
- Q2 và Q3 không lọc metadata để giữ recall trên nhiều thông báo/quy định liên quan.

Metadata filter là một phần trong chiến lược retrieval cá nhân; năm câu hỏi, gold answers, `relevant_doc_ids` và `required_evidence` vẫn giữ nguyên theo benchmark chung của nhóm. Các score dưới đây là dot product trên vector đã chuẩn hóa L2, vì vậy tương đương cosine similarity.

| # | Top-1 `doc_id` | Score | Hit@3 | Evidence | Câu trả lời của agent (tóm tắt) |
|---:|---|---:|:---:|:---:|---|
| 1 | `summer-2026-new-student-portal` | 0.780154 | Có | 4/4 | Từ Summer 2026, sinh viên dùng VinUniDigi Student Portal; cần chọn đúng kỳ, kiểm tra prerequisite, chỗ trống và trùng lịch, nhấn CONFIRM, kiểm tra trạng thái Registered và xem trước thời khóa biểu. |
| 2 | `summer-2026-registration` | 0.778531 | Có | 4/4 | Thời gian đăng ký là 29/6–4/7/2026; hạn add/drop cuối cùng là 11/7/2026. |
| 3 | `spring-2026-important-notes` | 0.754785 | Có | 3/3 | Withdrawal sau add/drop được ghi điểm W, phải thực hiện trước khi hoàn thành quá 30% thời lượng môn học và tổng giới hạn là 18 tín chỉ trong toàn chương trình. |
| 4 | `summer-2026-new-student-portal` | 0.486058 | Có | 4/4 | Full nghĩa là không còn chỗ; Conflict nghĩa là lớp trùng với lớp đã đăng ký; hệ thống không cho đăng ký nếu chưa đạt prerequisite hoặc pre-study requirement. |
| 5 | `forms-and-petitions` | 0.527089 | Có | 4/4 | Retake, audit và individual study được gửi email tới Registrar's Office; withdrawal sau add/drop cũng gửi qua email và cần giảng viên môn học phê duyệt. |

**Kết quả tổng hợp:** `Hit@3 = 5/5 (100%)`; độ bao phủ bằng chứng `19/19 (100%)`; cả 5 câu trả lời tóm tắt đều khớp gold answer.

### Phân tích và cải thiện

Ở cấu hình ban đầu `RecursiveChunker(chunk_size=500)` không bổ sung category filter, hệ thống đạt `4/5` top-3 hits. Q5 bị MISS vì các đoạn nói chung về withdrawal trong `undergraduate-academic-regulations` có similarity cao hơn trang thủ tục `forms-and-petitions`. Q1 và Q3 cũng bị chia evidence qua nhiều chunk nhỏ, khiến context chưa đủ để trả lời trọn vẹn.

Tôi thử nhiều cấu hình fixed-size, sentence và recursive với các kích thước khác nhau. `RecursiveChunker(chunk_size=1600)` giúp giữ các mục liên quan trong chunk lớn hơn; sau đó pre-filter theo `category` cho Q4 và Q5 loại các tài liệu đúng chủ đề rộng nhưng sai loại nội dung. Cấu hình cuối đạt `5/5` top-3 hits và lần lượt đủ `4/4`, `4/4`, `3/3`, `4/4`, `4/4` required evidence cho Q1–Q5.

| Cấu hình | Hit@3 | Tỷ lệ hit | Nhận xét chính |
|---|---:|---:|---|
| Baseline: Recursive 500, không category filter | 4/5 | 80% | Q5 lấy tài liệu đúng chủ đề nhưng sai mục đích; Q1 và Q3 bị phân tán bằng chứng. |
| Cuối: Recursive 1600 + metadata pre-filter | 5/5 | 100% | Tăng `20` điểm phần trăm và bao phủ đủ `19/19` mục bằng chứng. |

Điểm similarity của Q4 và Q5 thấp hơn một số ứng viên không lọc nhưng kết quả được chọn lại đúng hơn. Pre-filter không làm thay đổi cosine score của một cặp query–chunk; nó loại các ứng viên sai loại trước khi xếp hạng. Vì vậy, score cao nhất chưa chắc phù hợp nhất với ý định nghiệp vụ nếu metadata về `audience` hoặc `category` chưa được xét đến.

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**

Retrieval tốt phụ thuộc đồng thời vào chunking, metadata và cách đặt truy vấn, không chỉ phụ thuộc embedding model. Ví dụ rõ nhất là Q5: với vector search thuần và `chunk_size=500`, đoạn “Course Withdrawal” trong `undergraduate-academic-regulations` được xếp trên `forms-and-petitions`, dù đoạn đầu chỉ giải thích quy định còn câu hỏi cần hướng dẫn thủ tục. Kết hợp chunk đủ ngữ cảnh với pre-filter theo `category` giúp hệ thống chọn đúng loại tài liệu. Bài học rút ra là embedding giải quyết độ gần ngữ nghĩa, còn metadata bổ sung tín hiệu về đối tượng và mục đích sử dụng.

### Hạn chế và hướng phát triển

- Benchmark chỉ có 5 câu hỏi trong một miền hẹp, vì vậy kết quả `100%` chưa chứng minh khả năng khái quát trên các dịch vụ đại học khác.
- Quy tắc filter hiện được ánh xạ thủ công theo ý định query; bước tiếp theo là xây dựng query router nhất quán và đo cả trường hợp filter quá chặt làm giảm recall.
- In-memory store phù hợp với lab nhưng chưa có persistence hay chỉ mục gần đúng; khi corpus lớn hơn cần đánh giá thêm latency, bộ nhớ và vector database chuyên dụng.
- Nên cố định phiên bản model/dependency và lưu kết quả từng lần chạy để benchmark có thể tái lập hoàn toàn.

---

## Tự đánh giá phần cá nhân

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 10 / 10 |
| **Tổng phần cá nhân** | **60 / 60** |

## Minh chứng trong repository

- Mã nguồn triển khai: `src/chunking.py`, `src/embeddings.py`, `src/store.py`, `src/agent.py`.
- Bộ kiểm thử: `tests/test_solution.py`.
- Bộ benchmark chung: `benchmarks/vinuni_course_registration.json`.
- Corpus đánh giá: `data/vinuni_course_registration/`.
- Báo cáo nhóm dùng cùng corpus và benchmark: `report/REPORT_NHOM.md`.
