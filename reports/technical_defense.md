# Technical Defense & Architecture Report - Lab 19

**Tác giả:** Nguyễn Tuấn Dương

---

## 1. Cơ Chế Phân Giải Đại Từ (Coreference Resolution) & Rủi Ro Với Knowledge Graph

### Tình huống thực tế (Từ dataset HackerNoon)
- **Đoạn văn:** `"...announced that Sineng Electric is integrating onsemi EliteSiC silicon carbide power modules into its utility-scale solar inverters. The company stated that..."`
- **Phân tích:** Trong đoạn văn trên, đại từ `"The company"` xuất hiện ngay sau khi nhắc đến hai thực thể: `Sineng Electric` (bên tích hợp) và `onsemi` (bên phát triển).
- **Hệ quả khi phân giải sai:** Nếu cơ chế Coreference Resolution diễn giải `"The company"` thành `onsemi` thay vì `Sineng Electric`, LLM sẽ tạo ra **False Edge** (ví dụ: `onsemi` -> `PRODUCES` -> `utility-scale solar inverters`). Điều này làm hỏng toàn bộ *provenance* của đồ thị, gây ra thông tin sai lệch nghiêm trọng (hallucination) khi hệ thống truy vấn đa bước.

---

## 2. Thresholding & Cơ Chế Lexical Guard

### Quyết định Ngưỡng Cosine Similarity
- Sử dụng mô hình `sentence-transformers/all-MiniLM-L6-v2` với chuẩn hóa L2 và FAISS (IndexFlatIP).
- **Ngưỡng lựa chọn:** `0.90`. Ngưỡng này đảm bảo độ khắt khe cao, tránh gộp nhầm các thực thể có tên gần giống nhưng khác biệt về mặt bản chất kinh doanh.

### Lexical Guard (Bảo vệ ngữ nghĩa cốt lõi)
- **Cặp thực thể:** `Apple Music` (Dịch vụ) và `Apple` (Tập đoàn mẹ).
- **Cosine Similarity:** Thường đạt mức **0.88 - 0.91** do chung miền ngữ nghĩa thương hiệu.
- **Quyết định:** `REJECT_GUARD`.
- **Lý do:** Mặc dù vector nhúng rất giống nhau, Lexical Guard (thông qua `SequenceMatcher` hoặc bộ lọc hậu xử lý) nhận diện sự khác biệt cốt lõi. Việc sáp nhập hai node này sẽ dẫn đến "Graph Pollution" (ô nhiễm đồ thị) — gán toàn bộ doanh thu, nhân sự, M&A của sản phẩm con vào thẳng tập đoàn mẹ, làm sai lệch cấu trúc dữ liệu.

---

## 3. Kiến Trúc Neo4j & Trích Xuất Super-node

### Top 3 Thực Thể Bậc Cao Nhất (Degree)
Trong quá trình nạp dữ liệu vào Neo4j, các thực thể có tầm ảnh hưởng lớn sẽ trở thành Super-nodes:
1. **Apple** (Company) - Degree: 3
2. **Disney** (Company) - Degree: 2
3. **Sineng Electric** (Company) - Degree: 2

### Ưu Điểm & Rủi Ro Của Temporal Mitigation (Giới Hạn 50 Cạnh Gần Nhất)
Để giải quyết bài toán "bùng nổ" số lượng cạnh tại các Super-nodes (ví dụ: Apple có hàng nghìn cạnh liên kết), chiến lược cắt tỉa thời gian (chỉ lấy top 50 cạnh mới nhất) được áp dụng:
- **Ưu điểm:** 
  - Đảm bảo Context Window không bị phình to quá mức giới hạn của LLM.
  - Mang lại độ tươi mới (Freshness) cho câu trả lời, rất quan trọng trong lĩnh vực tin tức công nghệ/tài chính.
- **Rủi ro:** 
  - Mất mát thông tin lịch sử (Historical Amnesia). Nếu người dùng truy vấn về các sự kiện quá khứ (ví dụ: lịch sử thâu tóm công ty của Apple thập niên 2010), thông tin này có thể đã bị loại bỏ khỏi đồ thị cục bộ được nạp vào LLM.

---

## 4. Xử Lý Bottleneck Ở Quy Mô 350MB
Nếu triển khai hệ thống cho 100,000 bài báo:
- **Bottleneck chính:** Độ trễ và chi phí cực lớn khi gọi LLM API (Groq/OpenAI) để trích xuất NER và Relation (O(N) operations).
- **Giải pháp:**
  - Sử dụng **Async Task Queue** (như Celery hoặc Ray) để chạy batch inference song song.
  - **Fine-tune SLM (Small Language Models):** Sử dụng các mô hình chuyên biệt kích thước nhỏ (như GLiNER) để trích xuất Triplets với chi phí siêu rẻ thay vì phụ thuộc hoàn toàn vào LLM thương mại lớn.
  - Sử dụng Cypher `UNWIND` với neo4j batch injection để tối ưu I/O database.
