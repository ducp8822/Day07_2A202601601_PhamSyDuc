# Báo Cáo Nhóm — Lab 7: Embedding & Vector Store

**Nhóm:** C52
**Thành viên:** Phạm Sỹ Đức *(bổ sung các thành viên còn lại trước khi nộp)*
**Ngày:** 03/08/2026

> **Nộp 1 bản / nhóm.** Phần cá nhân (hướng tiếp cận, kết quả riêng, dự đoán…) mỗi thành viên nộp riêng trong `REPORT_CANHAN.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần nhóm: 40** = Lựa chọn tài liệu (10) + Thiết kế chiến lược (15) + Chất lượng truy xuất (10) + Thuyết trình (5).

---

## 1. Lựa chọn tài liệu (Document Set Quality) — Nhóm (10 điểm)

### Phạm vi bộ tài liệu (Scope)

**Chủ đề (cố định theo lớp K3):** Dịch vụ / quy định đại học (đăng ký môn, học phí, học bổng, thư viện, ký túc xá…).

**Phạm vi cụ thể nhóm tập trung:**
> Quy định, lịch và quy trình đăng ký học phần dành cho sinh viên VinUni, bao gồm add/drop, withdrawal và các yêu cầu học vụ liên quan.

### Danh sách tài liệu (Data Inventory)

| # | Tên tài liệu | Nguồn (Source URL) | Ngày lấy / Phiên bản | Số ký tự | Metadata đã gán |
|---|--------------|------------|--------------------|----------|-----------------|
| 1 | Forms and Petitions | https://registrar.vinuni.edu.vn/academics/forms-petitions/ | 2026-08-03 / not-stated | 1,625 | audience, department, category, language, temporal_scope |
| 2 | Registrar FAQs | https://registrar.vinuni.edu.vn/faqs/ | 2026-08-03 / not-stated | 1,497 | audience, department, category, language, temporal_scope |
| 3 | Class Schedule and Course Registration | https://registrar.vinuni.edu.vn/academics/class-schedule-course-registration/ | 2026-08-03 / not-stated | 2,337 | audience, department, category, language, temporal_scope |
| 4 | Important Announcement for Spring 2026 Semester | https://registrar.vinuni.edu.vn/2026/01/28/important-announcement-for-spring-2026-semester/ | 2026-08-03 / 2026-01-28 | 2,476 | audience, department, category, semester, temporal_scope |
| 5 | Spring 2026 Course Registration Announcement | https://registrar.vinuni.edu.vn/2025/12/15/official-announcement-spring-2026-course-registration/ | 2026-08-03 / 2025-12-15 | 2,289 | audience, department, category, semester, temporal_scope |
| 6 | New Student Portal for Summer 2026 | https://registrar.vinuni.edu.vn/2026/06/29/announcement-launch-of-the-new-student-portal-for-summer-2026-course-registration/ | 2026-08-03 / 2026-06-29 | 2,246 | audience, department, category, semester, temporal_scope |
| 7 | Summer 2026 Course Registration Announcement | https://registrar.vinuni.edu.vn/2026/05/22/official-announcement-summer-2026-course-registration/ | 2026-08-03 / 2026-05-22 | 2,397 | audience, department, category, semester, temporal_scope |
| 8 | Academic Regulations for Full-Time Undergraduate Programs | https://policy.vinuni.edu.vn/all-policies/academic-regulations-for-full-time-undergraduate-programs/ | 2026-08-03 / 8.1-2024-10-30 | 4,821 | audience, department, category, language, temporal_scope |

**Danh sách kiểm tra quản trị dữ liệu (Data governance checklist):**
- [x] Tập tài liệu (Corpus) chỉ chứa nguồn công khai/được phép dùng và không chứa dữ liệu cá nhân, thông tin đăng nhập hoặc tài liệu nội bộ.
- [x] Mỗi tài liệu có `source_url`, `retrieved_at`, `document_version` (hoặc ngày hiệu lực) trong metadata.

### Cấu trúc Metadata (Metadata Schema)

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho truy xuất (retrieval)? |
|----------------|------|---------------|-------------------------------|
| `doc_id` | string | `summer-2026-registration` | Đối chiếu kết quả với tài liệu chuẩn và truy vết từng chunk. |
| `audience` | string | `student` | Lọc tài liệu đúng nhóm người dùng. |
| `department` | string | `registrar` | Giới hạn kết quả theo đơn vị phụ trách. |
| `category` | string | `semester-registration-announcement` | Phân biệt quy định, thông báo và hướng dẫn thủ tục. |
| `semester` | string | `summer-2026` | Tránh trộn lịch giữa các học kỳ. |
| `source_url`, `retrieved_at`, `document_version` | string | URL / ngày / phiên bản | Kiểm chứng nguồn và độ cập nhật của thông tin. |

---

## 2. Thiết kế chiến lược (Strategy Design) — Nhóm (15 điểm)

> Mỗi thành viên thử **một chiến lược khác nhau** trên cùng bộ tài liệu; nhóm tổng hợp và so sánh ở đây.

### Phân tích đường cơ sở (Baseline Analysis)

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Chiến lược (Strategy) | Số lượng Chunk | Độ dài trung bình | Giữ được ngữ cảnh không? |
|-----------|----------|-------------|------------|-------------------|
| New Student Portal Summer 2026 | FixedSizeChunker (`fixed_size`) | 5 | 476.60 | Có thể cắt giữa câu/mục. |
| New Student Portal Summer 2026 | SentenceChunker (`by_sentences`) | 12 | 178.92 | Giữ câu tốt nhưng chunk khá nhỏ. |
| New Student Portal Summer 2026 | RecursiveChunker (`recursive`) | 6 | 362.17 | Giữ ranh giới đoạn và ngữ cảnh tốt. |
| Summer 2026 Registration | FixedSizeChunker (`fixed_size`) | 6 | 433.50 | Kích thước ổn định nhưng có thể cắt ý. |
| Summer 2026 Registration | SentenceChunker (`by_sentences`) | 9 | 258.11 | Mạch lạc ở cấp câu. |
| Summer 2026 Registration | RecursiveChunker (`recursive`) | 6 | 390.17 | Cân bằng độ dài và tính hoàn chỉnh. |
| Undergraduate Academic Regulations | FixedSizeChunker (`fixed_size`) | 11 | 478.82 | Có nguy cơ trộn/cắt các điều khoản. |
| Undergraduate Academic Regulations | SentenceChunker (`by_sentences`) | 16 | 295.44 | Nhiều chunk, đôi khi thiếu ngữ cảnh điều khoản. |
| Undergraduate Academic Regulations | RecursiveChunker (`recursive`) | 13 | 364.85 | Bảo toàn đoạn và tiêu đề tốt hơn. |

### Chiến lược của từng thành viên

> Mỗi thành viên điền một khối dưới đây (copy thêm nếu nhóm có nhiều hơn 3 người).

**Thành viên 1 — Phạm Sỹ Đức**
- **Loại chiến lược:** Recursive, `chunk_size=1600`, kết hợp metadata pre-filter theo ý định truy vấn
- **Mô tả & lý do chọn cho chủ đề này:** Chiến lược ưu tiên ranh giới đoạn, dòng, câu rồi khoảng trắng nên phù hợp với thông báo và quy định có cấu trúc mục. Kích thước 1600 giữ các evidence liên quan trong cùng context; metadata `audience` và `category` loại tài liệu đúng chủ đề rộng nhưng sai mục đích sử dụng.
- **Code snippet (nếu custom):**
```python
# Sử dụng RecursiveChunker(chunk_size=1600) và EmbeddingStore.search_with_filter()
```

**Thành viên 2 — [Tên]**
- **Loại chiến lược:**
- **Mô tả & lý do chọn:**
- **Code snippet (nếu custom):**

**Thành viên 3 — [Tên]**
- **Loại chiến lược:**
- **Mô tả & lý do chọn:**
- **Code snippet (nếu custom):**

### So Sánh Giữa Các Thành Viên

| Thành viên | Chiến lược (Strategy) | Điểm truy xuất (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Phạm Sỹ Đức | Recursive 1600 + metadata filter | 10/10 | Cả 5 câu có tài liệu đúng trong top-3 và đủ required evidence. | Cần thiết kế quy tắc filter phù hợp với ý định truy vấn. |
| | | | | |
| | | | | |

**Chiến lược nào tốt nhất cho chủ đề này? Tại sao?**
> Trong lần chạy của Phạm Sỹ Đức, RecursiveChunker với `chunk_size=1600` kết hợp metadata pre-filter cho kết quả tốt nhất, đạt 5/5 câu và đủ toàn bộ evidence. Chunk lớn hơn giữ các mục liên quan trong context, còn filter theo `category` giải quyết trường hợp q5 bị quy định withdrawal chung lấn át trang hướng dẫn thủ tục; kết luận chung vẫn cần bổ sung kết quả của các thành viên khác.

---

## 3. Câu hỏi đánh giá & Chất lượng truy xuất (Retrieval Quality) — Nhóm (10 điểm)

### Câu hỏi đánh giá & Câu trả lời chuẩn (nhóm thống nhất)

> **Đúng 5 câu hỏi**, đa dạng, có thể kiểm chứng; **ít nhất 1 câu** cần lọc metadata mới trả lời tốt. Đây là bộ câu hỏi chung cho mọi thành viên chạy.

| # | Câu hỏi (Query) | Câu trả lời chuẩn (Gold Answer) | Chunk nào chứa thông tin? |
|---|-------|-------------------------------|--------------------------|
| 1 | Starting in Summer 2026, which portal must students use for course registration, and what checks confirm that registration is complete? | Registration uses the VinUniDigi Student Portal. Students must choose the correct term, check prerequisites, availability and timetable conflicts, click CONFIRM, verify the Registered status and preview the timetable. | `summer-2026-new-student-portal` |
| 2 | What was the Summer 2026 course registration period, and what was the final add/drop deadline? | Registration ran from June 29 to July 4, 2026; the final add/drop deadline was July 11, 2026. | `summer-2026-registration` |
| 3 | After the add/drop period, how is a course withdrawal recorded, by what point must it occur, and what is the program-wide withdrawal credit limit? | A withdrawal receives a W grade, must occur before more than 30% of course study time is completed, and is limited to 18 credits across the program. | `undergraduate-academic-regulations`, `spring-2026-important-notes` |
| 4 | What do Full and Conflict mean during course registration, and what happens when prerequisite requirements have not been satisfied? | Full means no seats are available; Conflict means the class overlaps another registered class; unmet prerequisites prevent registration. | `summer-2026-new-student-portal`, `registration-hub` |
| 5 | How should students request a course retake, audit or individual study, and how should they request withdrawal after the add/drop period? | Requests are emailed to the Registrar's Office. Withdrawal after add/drop is also requested by email and requires the instructor's approval. | `forms-and-petitions` |

### Tổng hợp chất lượng truy xuất của nhóm

> Cách chấm (theo `docs/SCORING.md`): **2 điểm/câu** — top-3 chứa chunk liên quan + agent trả lời đúng (2), có liên quan nhưng thiếu/không ở top-1 (1), không có trong top-3 (0).

| # | Câu hỏi | Chiến lược tốt nhất cho câu này | Có chunk liên quan trong top-3? | Ghi chú |
|---|---------|-------------------------------|-------------------------------|---------|
| 1 | New Student Portal và các bước xác nhận đăng ký | Recursive 1600 + audience filter | Có | Top-1 đúng, score 0.780154, đủ 4/4 evidence. |
| 2 | Thời gian đăng ký và hạn add/drop Summer 2026 | Recursive 1600 | Có | Top-1 đúng, score 0.778531, đủ 4/4 evidence. |
| 3 | Quy định withdrawal, điểm W, mốc 30% và giới hạn 18 tín chỉ | Recursive 1600 | Có | Top-1 đúng, score 0.754785, đủ 3/3 evidence. |
| 4 | Ý nghĩa Full, Conflict và prerequisite | Recursive 1600 + category filter | Có | Top-1 đúng, score 0.486058, đủ 4/4 evidence. |
| 5 | Quy trình gửi yêu cầu retake/audit/individual study/withdrawal | Recursive 1600 + category filter | Có | Top-1 là `forms-and-petitions`, score 0.527089, đủ 4/4 evidence. |

**Lọc bằng metadata có giúp ích không? Ở câu hỏi nào?**
> Q1 sử dụng `metadata_filter={"audience": "student"}` đúng yêu cầu K3. Q4 lọc `category=registration-system-guide` để ưu tiên tài liệu giải thích trạng thái hệ thống, còn q5 lọc `category=academic-request-process` để lấy đúng trang Forms and Petitions. Q4 và q5 cho thấy metadata filter cải thiện precision rõ rệt dù điểm similarity tuyệt đối có thể thấp hơn.

---

## 4. Thuyết trình (Demo) & Bài học nhóm — Nhóm (5 điểm)

**Những phân tích (insights) hay nhất nhóm sẽ trình bày:**
> Local multilingual embedding cho kết quả ngữ nghĩa rõ ràng hơn mock embedding. Baseline Recursive 500 đạt 4/5, còn Recursive 1600 kết hợp metadata pre-filter đạt 5/5 và đủ evidence. Q5 minh họa rõ việc vector search thuần túy có thể ưu tiên quy định chung, trong khi metadata giúp chọn đúng loại tài liệu thủ tục.

**Bài học rút ra khi so sánh trong nhóm:**
> Fixed-size tạo chunk ổn định nhưng có thể cắt giữa ý; sentence chunking giữ câu tốt nhưng tạo nhiều đoạn nhỏ. Recursive chunking giữ được cấu trúc đoạn tốt hơn, dù vẫn cần chunk theo heading cho các trang chứa nhiều loại thủ tục.

**Nếu làm lại, nhóm sẽ thay đổi gì trong chiến lược dữ liệu (data strategy)?**
> Nhóm sẽ chuẩn hóa metadata `category` ngay khi thu thập tài liệu và xây dựng quy tắc query routing để chọn filter nhất quán. Ngoài Recursive 1600, nhóm sẽ thử section-based chunking để giữ tiêu đề `Course Retake`, `Audit`, `Individual Study` và `Withdrawal` cùng nội dung tương ứng.

---

## Tự Đánh Giá (Phần Nhóm)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Lựa chọn tài liệu (Document Set Quality) | / 10 |
| Thiết kế chiến lược (Strategy Design) | / 15 |
| Chất lượng truy xuất (Retrieval Quality) | / 10 |
| Thuyết trình (Demo) | / 5 |
| **Tổng phần nhóm** | **/ 40** |
