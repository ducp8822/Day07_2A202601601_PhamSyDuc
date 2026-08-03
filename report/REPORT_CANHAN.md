# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Phạm Sỹ Đức
**Nhóm:** C52
**Ngày:** 03/08/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**
> Độ tương tự cosine cao nghĩa là hai vector embedding có hướng gần giống nhau. Trong bài toán văn bản, điều này thường cho thấy hai câu hoặc tài liệu có nội dung ngữ nghĩa tương đồng.

**Ví dụ có độ tương tự CAO:**
- Câu A: Sinh viên có thể đăng ký môn học trực tuyến.
- Câu B: Người học được phép ghi danh học phần qua hệ thống online.
- Tại sao tương đồng: Hai câu dùng từ khác nhau nhưng đều nói về việc sinh viên đăng ký học phần trực tuyến.

**Ví dụ có độ tương tự THẤP:**
- Câu A: Sinh viên cần đóng học phí trước ngày quy định.
- Câu B: Thư viện mở cửa từ 8 giờ sáng.
- Tại sao khác: Một câu nói về học phí, câu còn lại nói về thời gian hoạt động của thư viện.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**
> Cosine similarity tập trung vào hướng của vector nên ít bị ảnh hưởng bởi độ lớn của vector. Điều này phù hợp với text embeddings vì hướng thường thể hiện ngữ nghĩa quan trọng hơn độ dài tuyệt đối.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> Bước dịch giữa hai chunk là `500 - 50 = 450`. Số chunk là `ceil((10,000 - 50) / 450) = 23`.
> **Đáp án: 23 chunks.**

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**
> Khi overlap tăng lên 100, bước dịch còn `500 - 100 = 400`, nên số chunk là `ceil((10,000 - 100) / 400) = 25`. Overlap lớn hơn giúp giữ ngữ cảnh ở ranh giới chunk tốt hơn, nhưng làm tăng số vector cần lưu và tìm kiếm.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi lập trình (implement) các phần chính trong gói `src`.

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk`** — hướng tiếp cận:
> Tôi dùng regex `(?<=[.!?])\s+|\n+` để tách văn bản sau dấu kết thúc câu hoặc tại ký tự xuống dòng. Các phần rỗng được loại bỏ, sau đó các câu được gom theo `max_sentences_per_chunk`; văn bản rỗng trả về danh sách rỗng.

**`RecursiveChunker.chunk` / `_split`** — hướng tiếp cận:
> Thuật toán thử lần lượt các separator theo mức ưu tiên: đoạn văn, dòng, câu, khoảng trắng và cuối cùng là tách trực tiếp. Base case xảy ra khi đoạn hiện tại không vượt quá `chunk_size`; nếu không còn separator phù hợp, văn bản được cắt theo kích thước cố định.

### Lớp EmbeddingStore

**`add_documents` + `search`** — hướng tiếp cận:
> Mỗi `Document` được chuyển thành một record gồm ID, nội dung, metadata và embedding rồi lưu trong danh sách bộ nhớ. Khi tìm kiếm, query được embedding và tính dot product với từng record; kết quả được sắp xếp theo score giảm dần và lấy `top_k`.

**`search_with_filter` + `delete_document`** — hướng tiếp cận:
> `search_with_filter` lọc record theo tất cả cặp khóa-giá trị metadata trước khi tính điểm similarity, giúp giảm tập ứng viên. `delete_document` tạo lại danh sách và loại các record có `metadata.doc_id` hoặc ID tương ứng, sau đó trả về việc kích thước store có giảm hay không.

### Tác tử KnowledgeBaseAgent

**`answer`** — hướng tiếp cận:
> `answer` lấy các chunk liên quan bằng `store.search`, ghép nội dung thành context rồi đưa vào prompt cùng câu hỏi. Prompt yêu cầu mô hình chỉ dựa trên context và thông báo không có thông tin nếu ngữ cảnh không chứa câu trả lời.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```
42 tests collected
42 passed, 0 failed
```

**Số lượng bài test vượt qua (pass):** 42 / 42

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

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

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Starting in Summer 2026, which portal must students use for course registration, and what checks confirm that registration is complete? | `summer-2026-new-student-portal`: đăng ký từ Summer 2026 được thực hiện trên VinUniDigi Student Portal. | 0.792193 | Có | Xác định đúng portal nhưng context top-3 chưa chứa đầy đủ các bước kiểm tra và trạng thái `Registered`. |
| 2 | What was the Summer 2026 course registration period, and what was the final add/drop deadline? | `summer-2026-registration`: thời gian đăng ký từ 29/6 đến 4/7/2026. | 0.866933 | Có | Trả lời được thời gian đăng ký và hạn add/drop cuối cùng là 11/7/2026 từ các chunk top-2. |
| 3 | After the add/drop period, how is a course withdrawal recorded, by what point must it occur, and what is the program-wide withdrawal credit limit? | `spring-2026-important-notes`: withdrawal phải trước khi hoàn thành quá 30% thời lượng môn học. | 0.736959 | Có | Context trả lời được điểm `W` và mốc 30%, nhưng top-3 chưa chứa giới hạn 18 tín chỉ. |
| 4 | What do Full and Conflict mean during course registration, and what happens when prerequisite requirements have not been satisfied? | `summer-2026-new-student-portal`: giải thích đầy đủ trạng thái Full, Conflict và prerequisite. | 0.664962 | Có | Trả lời đầy đủ: Full là hết chỗ, Conflict là trùng lịch và hệ thống chặn khi chưa đạt prerequisite. |
| 5 | How should students request a course retake, audit or individual study, and how should they request withdrawal after the add/drop period? | `undergraduate-academic-regulations`: quy định chung về withdrawal sau add/drop. | 0.739508 | Không | Không truy xuất được `forms-and-petitions`, nên thiếu quy trình gửi email, Registrar và phê duyệt của giảng viên. |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 4 / 5

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**
> Việc so sánh kết quả cho thấy chunking giữ đúng ranh giới mục giúp câu hỏi cụ thể truy xuất chính xác hơn. Một truy vấn ghép nhiều ý như câu 5 có thể ưu tiên đoạn quy định chung thay vì trang hướng dẫn thủ tục, vì vậy cần thử section-based chunking hoặc lọc thêm theo `category`.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 6 / 10 |
| **Tổng phần cá nhân** | **56 / 60** |
