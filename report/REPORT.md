# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Đức Hoàng Phúc
**Nhóm:** 02 - E402
**Ngày:** 10/04/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity

**High cosine similarity nghĩa là gì?**

> Cosine similarity cao cho thấy hai vector embedding có hướng gần giống nhau, tức là hai câu có ý nghĩa ngữ nghĩa tương đồng bất kể độ dài khác nhau.

---

**Ví dụ HIGH similarity:**

* Sentence A: I enjoy learning artificial intelligence.
* Sentence B: I like studying AI.
* → Cùng ý nghĩa về việc học AI.

---

**Ví dụ LOW similarity:**

* Sentence A: I enjoy learning artificial intelligence.
* Sentence B: The weather is very hot today.
* → Không liên quan về ngữ nghĩa.

---

**Tại sao dùng cosine thay vì Euclidean?**

> Cosine đo hướng (semantic), còn Euclidean bị ảnh hưởng bởi độ dài vector nên không phản ánh đúng similarity.

---

### Chunking Math

Step = 500 - 50 = 450

Chunks ≈ ceil((10000 - 50) / 450) = **23**

---

**Nếu overlap = 100:**
→ chunks ≈ 25 (tăng)

**Giải thích:**

> Overlap giúp giữ context giữa các chunk → tăng accuracy khi retrieval.

---

## 2. Document Selection — Nhóm

**Domain:** VinUni Policies

**Lý do chọn:**

> Tài liệu policy có cấu trúc rõ ràng, dài, và thông tin phân tán → phù hợp để test RAG multi-document retrieval.

### Data Inventory

| # | Tên tài liệu | Nguồn | Kích thước | Metadata đã gán |
|---|--------------|-------|------------|-----------------|
| 01 | Sexual_Misconduct_Response_Guideline | policy.vinuni.edu.vn | 26.7 KB | category=Guideline, lang=en, dept=SAM, topic=student_life |
| 02 | Admissions_Regulations_GME_Programs | policy.vinuni.edu.vn | 29.0 KB | category=Regulation, lang=en, dept=CHS, topic=academics |
| 03 | Cam_Ket_Chat_Luong_Dao_Tao | policy.vinuni.edu.vn | 43.3 KB | category=Report, lang=vi, dept=University, topic=academics |
| 04 | Chat_Luong_Dao_Tao_Thuc_Te | policy.vinuni.edu.vn | 18.4 KB | category=Report, lang=vi, dept=University, topic=academics |
| 05 | Doi_Ngu_Giang_Vien_Co_Huu | policy.vinuni.edu.vn | 9.9 KB | category=Report, lang=vi, dept=University, topic=academics |
| 06 | English_Language_Requirements | policy.vinuni.edu.vn | 14.0 KB | category=Policy, lang=en, dept=University, topic=academics |
| 07 | Lab_Management_Regulations | policy.vinuni.edu.vn | 46.1 KB | category=Regulation, lang=en, dept=Operations, topic=safety |
| 08 | Library_Access_Services_Policy | policy.vinuni.edu.vn | 3.5 KB | category=Policy, lang=en, dept=Library, topic=student_life |
| 09 | Student_Grade_Appeal_Procedures | policy.vinuni.edu.vn | 6.1 KB | category=SOP, lang=en, dept=AQA, topic=academics |
| 10 | Tieu_Chuan_ANAT_PCCN | policy.vinuni.edu.vn | 4.6 KB | category=Standard, lang=vi, dept=Operations, topic=safety |
| 11 | Quy_Dinh_Xu_Ly_Su_Co_Chay | policy.vinuni.edu.vn | 2.7 KB | category=Regulation, lang=vi, dept=Operations, topic=safety |
| 12 | Scholarship_Maintenance_Criteria | policy.vinuni.edu.vn | 5.7 KB | category=Guideline, lang=en, dept=SAM, topic=finance |
| 13 | Student_Academic_Integrity | policy.vinuni.edu.vn | 41.8 KB | category=Policy, lang=en, dept=AQA, topic=academics |
| 14 | Student_Award_Policy | policy.vinuni.edu.vn | 14.7 KB | category=Policy, lang=en, dept=SAM, topic=student_life |
| 15 | Student_Code_of_Conduct | policy.vinuni.edu.vn | 17.9 KB | category=Policy, lang=en, dept=SAM, topic=student_life |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `category` | string | Guideline / Policy / SOP / Standard / Report | Phân loại loại tài liệu — có thể filter_search theo type |
| `language` | string | `en` / `vi` | Biết ngôn ngữ gốc để chọn embedder phù hợp |
| `department` | string | SAM / AQA / CHS / Operations / Library | Filter theo phòng ban phụ trách |
| `topic` | string | academics / student_life / safety / finance | Filter theo chủ đề để thu hẹp search space |
| `source` | string | `data/12_Scholarship_Maintenance_Criteria.md` | Traceability — biết câu trả lời đến từ file nào |
| `chunk_index` | int | 0, 1, 2, ... | Xác định vị trí chunk trong tài liệu gốc |

---

## 3. Chunking Strategy — Cá nhân

### Baseline Analysis

| Strategy  | Chunk Count | Context       |
| --------- | ----------- | ------------- |
| Fixed     | cao         |  mất context |
| Sentence  | trung bình  |             |
| Recursive | thấp hơn    | tốt         |

---

### Strategy của tôi

**Loại:** RecursiveChunker

---

**Mô tả:**

> Recursive chunking chia text theo thứ tự: đoạn → dòng → câu → từ. Nếu chunk vẫn lớn hơn chunk_size thì tiếp tục chia nhỏ hơn. Base case là khi chunk đủ nhỏ.

---

**Lý do chọn:**

> Policy documents có cấu trúc rõ nên recursive chunking giúp giữ semantic context tốt hơn.

---

### So sánh

| Strategy  | Retrieval |
| --------- | --------- |
| Fixed     | kém       |
| Sentence  | ổn        |
| Recursive | tốt   |
| Semantic | tốt nhất  |

---

## 4. My Approach

### RecursiveChunker

> Sử dụng danh sách separator để split đệ quy. Khi không thể chia tiếp thì fallback sang cắt theo ký tự.

---

### EmbeddingStore

> Lưu vector embedding và tính cosine similarity để search.

---

### Agent

> Lấy top-k chunks → ghép thành context → đưa vào LLM để generate answer.

---

### Test

**42 / 42 tests pass**

---

## 5. Similarity Predictions

| Pair          | Dự đoán | Score |
| ------------- | ------- | ----- |
| AI vs AI      | high    | ~0.8  |
| AI vs weather | low     | ~0.2  |

---

**Insight:**

> Embedding hiểu semantic khá tốt nhưng vẫn có nhiễu trong cùng domain.

---

## 6. Results

### Benchmark Queries & Gold Answers (nhóm thống nhất)

> **Lưu ý:** Nhóm chọn cross-document queries — mỗi query yêu cầu tổng hợp thông tin từ 3–5 tài liệu để test khả năng multi-document retrieval.

| # | Query | Gold Answer (tóm tắt) | Tài liệu liên quan |
|---|-------|-----------------------|--------------------|
| 1 | What are all the conditions a student must maintain to stay in good academic standing at VinUni? | Duy trì GPA ≥ ngưỡng học bổng; không vi phạm academic integrity; tuân thủ code of conduct; đáp ứng tiêu chí xét giải thưởng | `12_Scholarship` + `13_Academic_Integrity` + `14_Award_Policy` + `15_Code_of_Conduct` |
| 2 | What safety and conduct regulations must students follow when using VinUni campus facilities? | Tuân thủ quy định phòng lab; quy trình xử lý sự cố cháy; quy tắc ứng xử chung; chính sách chống xâm hại tình dục | `07_Lab_Management` + `11_Fire_Safety` + `15_Code_of_Conduct` + `01_Sexual_Misconduct` |
| 3 | What are the admission and language requirements for students entering medical programs at VinUni? | Đáp ứng chuẩn tiếng Anh (IELTS/TOEFL); đạt điểm chuẩn tuyển sinh GME; đáp ứng tiêu chuẩn ANAT PCCN | `02_Admissions_GME` + `06_English_Language` + `10_Tieu_Chuan_ANAT` |
| 4 | What procedures and consequences apply when a student breaks university rules? | Quy trình khiếu nại/kháng nghị; hình thức kỷ luật theo mức độ vi phạm; xử lý gian lận học thuật; xử lý hành vi xâm hại | `01_Sexual_Misconduct` + `09_Grade_Appeal` + `13_Academic_Integrity` + `15_Code_of_Conduct` |
| 5 | How does VinUni evaluate and ensure the quality of its academic programs and teaching staff? | Cam kết chất lượng đào tạo; báo cáo chất lượng thực tế; tiêu chuẩn đội ngũ giảng viên; tiêu chí duy trì học bổng như thước đo kết quả | `03_Cam_Ket_Chat_Luong` + `04_Chat_Luong_Thuc_Te` + `05_Doi_Ngu_Giang_Vien` + `12_Scholarship` |

## 6.1 So Sánh Với Approach Của Nhóm

### Khác biệt chính

| Thành phần       | Approach của nhóm           | Approach của tôi               |
| ---------------- | --------------------------- | ------------------------------ |
| Chunking         | HeaderAwareChunker          | RecursiveChunker               |
| Embedding        | Voyage AI (multilingual-2)  | Local embedding (simple model) |
| Context handling | Theo section (policy-level) | Theo cấu trúc text             |
| Độ phức tạp      | Cao hơn                     | Đơn giản hơn                   |

---

### So sánh kết quả

| Tiêu chí                   | Nhóm                      | Cá nhân                 |
| -------------------------- | ------------------------- | ----------------------- |
| Retrieval accuracy (top-3) | 4/5 (~80%)                | 5/5 (100%)              |
| Semantic precision         | Cao (hiểu policy-level)   | Trung bình              |
| Generalization             | Tốt với structured docs   | Tốt với nhiều loại text |
| Robustness                 | Phụ thuộc format Markdown | Ít phụ thuộc format     |
| Triển khai                 | Phức tạp hơn              | Dễ implement            |

---

### Phân tích

> Approach của nhóm sử dụng Header-aware chunking giúp mỗi chunk tương ứng với một điều khoản hoàn chỉnh trong policy, từ đó cải thiện semantic precision khi truy vấn. Ngoài ra, việc sử dụng Voyage AI embedding giúp vector biểu diễn tốt hơn về mặt ngữ nghĩa, đặc biệt trong môi trường đa ngôn ngữ.

> Trong khi đó, approach cá nhân của tôi sử dụng Recursive chunking đơn giản hơn nhưng linh hoạt hơn khi áp dụng cho nhiều loại dữ liệu không có cấu trúc rõ ràng. Local embedding tuy kém hơn về semantic richness nhưng vẫn đạt hiệu quả tốt trong các truy vấn cơ bản.

---

### Kết luận

> Approach của nhóm phù hợp hơn cho các hệ thống production cần độ chính xác cao và dữ liệu có cấu trúc rõ ràng. Ngược lại, approach cá nhân phù hợp cho việc prototype nhanh, dễ triển khai và áp dụng cho nhiều domain khác nhau.

> Nếu kết hợp cả hai (Header-aware chunking + recursive fallback + embedding mạnh), hệ thống có thể đạt hiệu quả tốt hơn cả về độ chính xác và tính linh hoạt.

---

### Kết quả (local embedding)

| # | Top-1              | Score | Relevant |
| - | ------------------ | ----- | -------- |
| 1 | Scholarship        | 0.50  | ✔        |
| 2 | Lab safety         | 0.65  | ✔        |
| 3 | Admissions         | 0.57  | ✔        |
| 4 | Academic integrity | 0.52  | ✔        |
| 5 | Quality policy     | 0.50  | ✔        |

---

**Top-3 accuracy:** 5 / 5

---

### Phân tích nâng cao

* Retrieval đúng tài liệu chính trong tất cả queries
* Score trung bình (~0.5–0.65) → có thể cải thiện
* Query multi-document vẫn chưa combine tốt

---

### Limitation

* Chunking chưa semantic-aware hoàn toàn
* Embedding model nhỏ
* Chưa có reranking

---

### Future Work

* Hybrid search (BM25 + embedding)
* Dùng embedding model lớn hơn
* Thêm reranker

---

## 7. What I Learned

**Từ nhóm:**

> Chunking ảnh hưởng mạnh hơn LLM trong RAG pipeline.

---

**Từ nhóm khác:**

> Hybrid retrieval giúp tăng accuracy đáng kể.

---

**Nếu làm lại:**

> Sẽ clean data tốt hơn và tune chunk_size.

---

## Tự Đánh Giá

| Tiêu chí           | Điểm            |
| ------------------ | --------------- |
| Warm-up            | 5               |
| Document selection | 10              |
| Chunking           | 14              |
| My approach        | 10              |
| Similarity         | 5               |
| Results            | 10              |
| Implementation     | 30              |
| Demo               | 5               |
| **Tổng**           | **89 / 100** |