# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Phạm Sỹ Đức
**Nhóm:** C52
**Ngày:** 03/08/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

### Tóm tắt kết quả

- Hoàn thành toàn bộ phần lập trình cốt lõi và vượt qua `42/42` test.
- Sử dụng local embedding `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` thay cho mock embedding khi đánh giá retrieval.
- Đánh giá trên 8 tài liệu công khai của VinUni và 5 benchmark queries chung của nhóm.
- Chiến lược cuối cùng dùng `RecursiveChunker(chunk_size=1600)`, `top_k=3` và metadata filtering theo ý định truy vấn; kết quả đạt `5/5` top-3 hits và đủ toàn bộ required evidence.

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> Độ tương tự cosine đo góc giữa hai vector theo công thức `cos(a,b) = (a·b) / (||a|| ||b||)`. Giá trị gần `1` nghĩa là hai vector cùng hướng và thường biểu diễn nội dung có ngữ nghĩa tương đồng; gần `0` nghĩa là ít liên quan; giá trị âm cho thấy hai hướng biểu diễn đối lập hoặc rất khác nhau.

**Ví dụ có độ tương tự CAO:**
- Câu A: Sinh viên có thể đăng ký môn học trực tuyến.
- Câu B: Người học được phép ghi danh học phần qua hệ thống online.
- Tại sao tương đồng: Hai câu dùng từ khác nhau nhưng đều nói về việc sinh viên đăng ký học phần trực tuyến.

**Ví dụ có độ tương tự THẤP:**
- Câu A: Sinh viên cần đóng học phí trước ngày quy định.
- Câu B: Thư viện mở cửa từ 8 giờ sáng.
- Tại sao khác: Một câu nói về học phí, câu còn lại nói về thời gian hoạt động của thư viện.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> Cosine similarity tập trung vào hướng của vector nên ít bị ảnh hưởng bởi độ lớn của vector. Điều này phù hợp với text embeddings vì hai văn bản có thể có vector với độ lớn khác nhau nhưng vẫn cùng biểu diễn một ý nghĩa; trong khi đó Euclidean distance chịu ảnh hưởng trực tiếp bởi cả hướng lẫn độ lớn. Với embedding đã chuẩn hóa, dot product cũng tương đương cosine similarity và có thể tính nhanh hơn.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> Bước dịch giữa hai chunk là `step = chunk_size - overlap = 500 - 50 = 450`. Số chunk được tính bằng `ceil((L - overlap) / step)`, nên `ceil((10,000 - 50) / 450) = ceil(22.11) = 23`.
> **Đáp án: 23 chunks.**

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> Khi overlap tăng lên 100, bước dịch còn `500 - 100 = 400`, nên số chunk là `ceil((10,000 - 100) / 400) = 25`. Overlap lớn hơn giúp giữ ngữ cảnh ở ranh giới chunk tốt hơn, nhưng làm tăng số vector cần lưu và tìm kiếm.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
> Tôi kiểm tra đầu vào rỗng trước để trả về `[]`, sau đó dùng regex `(?<=[.!?])\s+|\n+` để tách tại khoảng trắng sau dấu kết thúc câu hoặc tại ký tự xuống dòng. Các phần rỗng và khoảng trắng thừa được loại bỏ. Cuối cùng, tôi duyệt danh sách câu theo bước `max_sentences_per_chunk`, ghép từng nhóm thành một chuỗi và bảo đảm mỗi chunk không vượt quá số câu cấu hình.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
> Thuật toán thử lần lượt các separator `"\n\n"`, `"\n"`, `". "`, `" "` và `""`, ưu tiên giữ nguyên đoạn văn và câu trước khi phải cắt theo ký tự. Base case xảy ra khi đoạn hiện tại có độ dài không vượt quá `chunk_size`, lúc đó đoạn được trả về ngay. Nếu separator hiện tại không xuất hiện, hàm chuyển sang separator tiếp theo; nếu không còn separator hoặc gặp separator rỗng, hàm fallback sang cắt cố định để bảo đảm không lặp vô hạn và luôn tạo được kết quả.

> Các phần nhỏ được ghép lại trong khi tổng độ dài vẫn nằm trong giới hạn. Phần vượt quá kích thước được xử lý đệ quy bằng separator có mức ưu tiên thấp hơn. Cách làm này giúp chunk giữ cấu trúc tự nhiên tốt hơn fixed-size chunking nhưng vẫn xử lý được văn bản dài không có dấu phân cách.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
> Tôi triển khai store trong bộ nhớ để đáp ứng phần bắt buộc của lab. Mỗi `Document` được chuẩn hóa thành record gồm ID duy nhất, nội dung, bản sao metadata và embedding; `doc_id` gốc được giữ trong metadata để truy vết và xóa tất cả chunk cùng tài liệu. Cách sao chép metadata tránh làm thay đổi đối tượng đầu vào.

> Khi tìm kiếm, query được chuyển thành embedding bằng cùng backend với tài liệu. Store tính dot product giữa query embedding và từng record, tạo kết quả gồm `id`, `content`, `metadata`, `score`, sau đó sắp xếp score giảm dần và trả về tối đa `top_k`. Local embedder chuẩn hóa vector nên dot product có ý nghĩa tương đương cosine similarity.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
> `search_with_filter` thực hiện pre-filter: một record chỉ được giữ khi tất cả cặp khóa-giá trị trong `metadata_filter` khớp. Sau đó hàm mới embedding query và xếp hạng tập ứng viên đã lọc. Cách này đặc biệt hữu ích để giới hạn đúng `audience`, `category` hoặc học kỳ, đồng thời tránh để tài liệu gần nghĩa nhưng sai đối tượng lấn át kết quả.

> `delete_document` ghi nhận kích thước ban đầu rồi tạo lại `_store`, loại mọi record có `metadata["doc_id"]` trùng với tài liệu cần xóa; điều kiện ID trực tiếp được giữ như một fallback. Hàm trả về `True` khi số record giảm và `False` nếu không tìm thấy tài liệu.

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
> `answer` triển khai ba bước của RAG: retrieve các chunk liên quan, ghép nội dung thành context, rồi gọi `llm_fn`. Prompt tách rõ phần hướng dẫn, `Context`, `Question` và vị trí `Answer`, đồng thời yêu cầu mô hình chỉ dùng dữ liệu được cung cấp và nói rằng thông tin không có sẵn nếu context không chứa câu trả lời. Việc truyền `llm_fn` từ bên ngoài giúp agent dễ kiểm thử và không phụ thuộc vào một nhà cung cấp LLM cụ thể.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
42 tests collected
42 passed, 0 failed
```

**Số lượng bài test vượt qua (pass):** 42 / 42

Các nhóm hành vi đã được kiểm tra gồm:

- Cấu trúc project và các interface bắt buộc.
- Fixed-size, sentence-based và recursive chunking, bao gồm input rỗng và fallback separator.
- Cosine similarity cho vector giống nhau, trực giao, đối hướng và zero vector.
- Thêm, đếm, tìm kiếm, sắp xếp score, lọc metadata và xóa tài liệu trong `EmbeddingStore`.
- Luồng trả lời của `KnowledgeBaseAgent`.

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

Tôi dự đoán trước mỗi cặp là cao hoặc thấp dựa trên mức độ tương đồng ngữ nghĩa, sau đó dùng cùng local multilingual embedding backend của thí nghiệm retrieval để lấy vector và tính cosine similarity. Trong bảng này, các cặp trên khoảng `0.60` được xem là tương đồng cao; các cặp gần `0` được xem là tương đồng thấp.

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Students register for courses through the online portal. | Course enrollment is completed using the student portal. | cao | 0.760827 | Có |
| 2 | The final add/drop deadline is July 11, 2026. | Students may change their Summer 2026 courses until July 11, 2026. | cao | 0.654121 | Có |
| 3 | A withdrawn course receives a W grade. | The cafeteria serves lunch from Monday to Friday. | thấp | 0.082864 | Có |
| 4 | The library opens at eight in the morning. | Unmet prerequisites prevent course registration. | thấp | 0.058327 | Có |
| 5 | Students must pay tuition before the stated deadline. | Tuition fees are due by the announced payment date. | cao | 0.607986 | Có |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**
> Cặp 5 có cùng ý nghĩa về hạn thanh toán học phí nhưng điểm chỉ đạt 0.607986, thấp hơn cặp 1. Điều này cho thấy embedding không chỉ dựa vào chủ đề chung mà còn chịu ảnh hưởng bởi cách diễn đạt và mức độ trùng khớp của các khái niệm cụ thể.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chạy **5 câu hỏi đánh giá của nhóm** trên mã nguồn cá nhân của bạn trong gói `src`. **5 câu hỏi này phải trùng với các thành viên cùng nhóm** (xem `REPORT_NHOM.md`).

### Thiết lập đánh giá

- Corpus: 8 tài liệu công khai về đăng ký học phần và quy định học vụ VinUni.
- Embedding backend: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`.
- Chiến lược chunking: `RecursiveChunker(chunk_size=1600)`.
- Số kết quả: `top_k=3`.
- Q1 lọc `{"audience": "student"}` để đáp ứng yêu cầu K3.
- Q4 lọc `{"category": "registration-system-guide"}` vì câu hỏi hỏi trạng thái trên hệ thống đăng ký.
- Q5 lọc `{"category": "academic-request-process"}` vì câu hỏi hỏi quy trình gửi yêu cầu học vụ.
- Q2 và Q3 không lọc metadata để giữ recall trên nhiều thông báo/quy định liên quan.

Metadata filter là một phần trong chiến lược retrieval cá nhân; năm câu hỏi và gold answers vẫn giữ nguyên theo benchmark chung của nhóm.

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Starting in Summer 2026, which portal must students use for course registration, and what checks confirm that registration is complete? | `summer-2026-new-student-portal`: giới thiệu VinUniDigi Student Portal và checklist hoàn tất đăng ký. | 0.780154 | Có | Từ Summer 2026, sinh viên dùng VinUniDigi Student Portal; cần chọn đúng kỳ, kiểm tra prerequisite, chỗ trống và trùng lịch, nhấn CONFIRM, kiểm tra trạng thái Registered và xem trước thời khóa biểu. |
| 2 | What was the Summer 2026 course registration period, and what was the final add/drop deadline? | `summer-2026-registration`: lịch đăng ký và mốc add/drop Summer 2026. | 0.778531 | Có | Thời gian đăng ký là 29/6–4/7/2026; hạn add/drop cuối cùng là 11/7/2026. |
| 3 | After the add/drop period, how is a course withdrawal recorded, by what point must it occur, and what is the program-wide withdrawal credit limit? | `spring-2026-important-notes`: điều kiện withdrawal và giới hạn tín chỉ. | 0.754785 | Có | Withdrawal sau add/drop được ghi điểm W, phải thực hiện trước khi hoàn thành quá 30% thời lượng môn học và tổng giới hạn là 18 tín chỉ trong toàn chương trình. |
| 4 | What do Full and Conflict mean during course registration, and what happens when prerequisite requirements have not been satisfied? | `summer-2026-new-student-portal`: mục Class Status và Prerequisite Requirements. | 0.486058 | Có | Full nghĩa là không còn chỗ; Conflict nghĩa là lớp trùng với lớp đã đăng ký; hệ thống không cho đăng ký nếu chưa đạt prerequisite hoặc pre-study requirement. |
| 5 | How should students request a course retake, audit or individual study, and how should they request withdrawal after the add/drop period? | `forms-and-petitions`: mục Academic Requests. | 0.527089 | Có | Retake, audit và individual study được gửi email tới Registrar's Office; withdrawal sau add/drop cũng gửi qua email và cần giảng viên môn học phê duyệt. |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 5 / 5

### Phân tích và cải thiện

Ở cấu hình ban đầu `RecursiveChunker(chunk_size=500)` không bổ sung category filter, hệ thống đạt `4/5` top-3 hits. Q5 bị MISS vì các đoạn nói chung về withdrawal trong `undergraduate-academic-regulations` có similarity cao hơn trang thủ tục `forms-and-petitions`. Q1 và Q3 cũng bị chia evidence qua nhiều chunk nhỏ, khiến context chưa đủ để trả lời trọn vẹn.

Tôi thử nhiều cấu hình fixed-size, sentence và recursive với các kích thước khác nhau. `RecursiveChunker(chunk_size=1600)` giúp giữ các mục liên quan trong chunk lớn hơn; sau đó pre-filter theo `category` cho Q4 và Q5 loại các tài liệu đúng chủ đề rộng nhưng sai loại nội dung. Cấu hình cuối đạt `5/5` top-3 hits và lần lượt đủ `4/4`, `4/4`, `3/3`, `4/4`, `4/4` required evidence cho Q1–Q5.

Điểm similarity của Q4 và Q5 thấp hơn một số kết quả ở cấu hình không lọc nhưng vẫn là kết quả đúng. Điều này cho thấy score chỉ nên được so sánh trong cùng tập ứng viên; metadata filter có thể tăng độ chính xác dù điểm tuyệt đối giảm.

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> Điều quan trọng nhất tôi rút ra là retrieval tốt phụ thuộc đồng thời vào chunking, metadata và cách đặt truy vấn, không chỉ phụ thuộc embedding model. Chunk quá nhỏ có thể làm mất evidence, còn tìm kiếm thuần vector có thể trả về tài liệu đúng chủ đề nhưng sai mục đích; kết hợp chunk đủ ngữ cảnh với metadata pre-filter giúp kết quả ổn định và dễ giải thích hơn.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 10 / 10 |
| **Tổng phần cá nhân** | **60 / 60** |
