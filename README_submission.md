# Báo cáo kết quả bài Lab 17 - Multi-Memory Agent với Zep

## 1. Trả lời câu hỏi lý thuyết thực hành
* **Layer quan trọng nhất trong bộ test này:** **Long-term Memory** (bộ nhớ dài hạn). Trong 11 case luyện tập, Long-term chịu trách nhiệm trực tiếp cho 4 case cốt lõi (E02, E03, E08, E09) liên quan đến sở thích cá nhân, xung đột Stack dự án và các open loops kéo dài qua nhiều session của các người dùng khác nhau.
* **Trade-off giữa Context Block (Zep) và Redis+Qdrant tự build:**
  * **Zep (Context Block / Graph Search):** Tự động phân tích, trích xuất và cập nhật các facts/entities theo thời gian thực dựa trên đồ thị tri thức của người dùng. Có cơ chế tự nén thông tin thông minh và phân tách namespace bảo mật. Nhược điểm: Latency trung bình cao (~2.2 giây) và phụ thuộc kết nối Internet Cloud.
  * **Redis+Qdrant tự build:** Tốc độ phản hồi cực nhanh (<10ms) và chạy hoàn toàn offline bảo mật. Nhược điểm: Lập trình viên phải tự viết mã phức tạp để quản lý cập nhật đồ thị, nén ngữ cảnh và phân tách người dùng thủ công.
* **Biện pháp chống Memory Poisoning (nạp thông tin bẩn):**
  * Thiết lập một bộ phân loại đầu vào (Input Classifier Model) bằng LLM nhỏ hoặc Regex để đánh giá độ tin cậy của thông tin trước khi đẩy vào bộ nhớ dài hạn.
  * Chỉ cho phép cập nhật `preference`/`constraint` bền vững khi có sự xác nhận rõ ràng từ phía người dùng (Explicit User Confirmation).
  * Giới hạn quyền chỉnh sửa các note dài hạn bằng cơ chế phân quyền (Role-based access) và lưu vết nguồn gốc (Provenance tracking) để rollback nếu phát hiện nhiễm độc.

## 2. Phân tích kết quả Benchmark
* **Layer có hit rate thấp nhất trong baseline no-memory:** **Long-term, Episodic và Semantic** đều đạt **0%** (0/11 case), do baseline không giữ lại bất kỳ thông tin nào ngoài session hiện tại. Trong khi đó, student-memory khôi phục thành công **90.9% Hit Rate (10/11 PASS)**.
* **Query retrieve nhiều token nhất:** **E02** và **E03** do Zep Context Block kéo theo toàn bộ thông tin Fact-sheet thu thập được của Minh.
* **Case mixed E07:** Kết hợp **Long-term** (sở thích Python của Minh) và **Semantic** (Quy tắc retry payment API chung). Hai bằng chứng bắt buộc xuất hiện trong kết quả truy xuất là `Python` và `Idempotency-Key`.
* **Mối quan hệ Token Reduction & Hit Rate:** Student-memory giảm 15.6% token với **90.9% Hit Rate**. Baseline no-memory giảm đến **81.8% token** nhưng **Hit Rate chỉ còn 18.2%** (chỉ pass các case short-term cục bộ). Token reduction cao của no-memory vô nghĩa vì nó không truy xuất được gì, làm hỏng chức năng Agent.

## 3. Phân tích E08 Recency và E10 Compaction
* **E08 Recency (CHƯA ĐẠT):** Minh họa quy luật "Recency Wins" nhưng case này **chưa pass**. Dù ban đầu Minh khai báo dùng Python cho demo cá nhân, sau đó ở session tiếp theo đã quy định dự án công ty `BLUEBIRD-42` bắt buộc dùng TypeScript. Khi truy vấn về BLUEBIRD-42, hệ thống **chưa trả về đủ** thông tin NestJS + TypeScript (missing: BLUEBIRD-42, TypeScript, NestJS).
* **E10 Compaction (ĐẠT):** Khi số lượt hội thoại vượt quá ngưỡng, cơ chế sliding compaction tự động nén các lượt cũ thành `SESSION_SUMMARY`, đồng thời trích xuất các ràng buộc đặc biệt chứa mã định danh (như `REVIEW-DEADLINE-1600`) đưa vào `DURABLE_NOTES` bảo toàn vĩnh viễn không bị xóa trôi đi.

---

## 4. Minh chứng kết quả chạy hệ thống (Logs)

### A. Kết quả Student Benchmark (PASS 10/11)
```text
E01 short_term: Ten du an ca nhan toi vua nhac la gi? -> PASS
E06 semantic: Quy tac retry POST payment la gi? -> PASS
E09 long_term: Lan uu tien stack backend nao cho LOTUS-88? -> PASS
E10 short_term: Deadline review cu la khi nao? -> PASS
E02 long_term: Voi demo ca nhan cua Minh, ngon ngu uu tien la gi? -> PASS
E03 long_term: Minh con open loop hay deadline nao chua hoan thanh? -> PASS
E04 episodic: Lan truoc Minh fix async HTTP timeout bang cach nao? -> PASS
E05 episodic: Reflection cua su co async la gi, tang timeout co phai root fix khong? -> PASS
E07 mixed: Hay chon huong dan code retry payment phu hop voi preference ca nhan cua Minh. -> PASS
E11 semantic: Theo incident playbook, truoc khi tang timeout can kiem tra gi? -> PASS
E08 long_term: Backend cua BLUEBIRD-42 bat buoc dung stack gi? -> FAIL (missing: BLUEBIRD-42, TypeScript, NestJS)
```

### B. Bảng So Sánh Hiệu Năng (Comparison Matrix)
```text
| Metric                     | Memory-enabled | No-memory | Delta   |
| -------------------------- | -------------: | --------: | ------: |
| Evidence hit rate          |          90.9% |     18.2% |  +72.7% |
| Passed cases               |          10/11 |      2/11 |      +8 |
| Avg retrieval latency (ms) |          949.4 |       0.1 | +949.3 |
| Avg token reduction        |          15.6% |     81.8% |  -66.2% |
```

### C. Kết quả Diễn tập Quyền riêng tư (Privacy Forget Verification)
```text
Deleting user-scoped memory for 'minh-lab17'...
Redis keys deleted: 0
Zep user absent: True
Redis user keys remaining: 0
Shared semantic KB remains intact because it stores domain knowledge, not user PII.
```

### D. Kết quả Đánh giá Golden Set (PASS 17/20)
```text
Golden 17/20. 
            Benchmark summary            
┏━━━━━━┳━━━━━━━━━━━━┳━━━━━━┳━━━━━━━━━━━━┓
┃ Case ┃ Layer      ┃ Pass ┃ Latency ms ┃
┡━━━━━━╇━━━━━━━━━━━━╇━━━━━━╇━━━━━━━━━━━━┩
│ G01  │ short_term │ yes  │        0.4 │
│ G02  │ short_term │ yes  │        0.1 │
│ G08  │ long_term  │ yes  │     2377.5 │
│ G09  │ long_term  │ yes  │     1859.1 │
│ G12  │ semantic   │ yes  │      519.6 │
│ G14  │ semantic   │ yes  │      502.3 │
│ G15  │ semantic   │ yes  │      527.1 │
│ G19  │ mixed      │ yes  │     2314.6 │
│ G03  │ long_term  │ yes  │     1806.9 │
│ G04  │ long_term  │ yes  │     2087.9 │
│ G05  │ long_term  │ yes  │     2399.7 │
│ G10  │ episodic   │ yes  │      767.7 │
│ G11  │ episodic   │ yes  │      781.6 │
│ G13  │ semantic   │ yes  │      508.7 │
│ G16  │ mixed      │ yes  │     2416.9 │
│ G18  │ mixed      │ yes  │     1479.7 │
│ G20  │ mixed      │ yes  │     3950.0 │
│ G06  │ long_term  │ NO   │     2038.7 │ (missing: TypeScript, NestJS)
│ G07  │ long_term  │ NO   │     1918.2 │ (missing: TypeScript, NestJS)
│ G17  │ mixed      │ NO   │     2340.3 │ (missing: TypeScript, NestJS)
└──────┴────────────┴──────┴────────────┘
```
