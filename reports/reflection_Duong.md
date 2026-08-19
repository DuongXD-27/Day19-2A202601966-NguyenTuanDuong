# Điểm Đánh Giá & Suy Ngẫm (Reflection) - Lab 19

**Học viên:** Nguyễn Tuấn Dương  
**Track:** 3 (GraphRAG)

---

## 1. Tự Đánh Giá Chấm Điểm
Dựa theo Rubric của Lab 19, tôi tự đánh giá các tiêu chí như sau:

| Tiêu chí | Điểm tự chấm (1–5) | Lý do chi tiết |
|:---|:---:|:---|
| **Hiểu sâu bài giảng GraphRAG** | **5/5** | Nắm vững sự cần thiết của Data Preprocessing (Coreference, Normalization) đối với chất lượng của Graph, và hiểu cơ chế Local Traversal / Hybrid Retrieval của GraphRAG. |
| **Kiểm soát AI Coding Agent** | **5/5** | Khi AI Agent gặp lỗi Model Unavailable (Groq HTTP 404) và Socket Exhaustion (WinError 10055), tôi đã định hướng AI thay đổi cấu hình `.env`, diệt các tiến trình lỗi và không ép hệ thống chạy lại toàn bộ một cách lãng phí (chỉ retry các component cần thiết). Ngoài ra, tôi đã ngăn chặn AI đưa ra thuật toán O(N^2) ngây thơ trong khâu Entity Resolution và thay bằng FAISS Index kết hợp Union-Find. |
| **Xây dựng Đồ thị chất lượng** | **5/5** | Sử dụng Neo4j AuraDB nạp thành công các entity và relationship với 0 lỗi provenance (`invalid_provenance_edges = 0`). Bảng đồ thị hoạt động mượt mà khi query. |
| **Phân tích Benchmarks & Lỗi** | **5/5** | Báo cáo chi tiết nguyên nhân Flat RAG thất bại ở Multi-hop (G02) và phân tích sâu vì sao GraphRAG đuối thế ở câu hỏi Global Aggregation (G05). Đề xuất biện pháp xử lý bằng Leiden Community Summarization hợp lý. |

*Kết luận: Tôi tự đánh giá bài nộp đạt 100/100 điểm cho Track 3 theo tiêu chuẩn Rubric.*

---

## 2. Bài Học Kinh Nghiệm & Đánh Đổi Hiệu Năng (Trade-offs)

### Đánh Đổi (Latency vs Quality vs Tokens)
- **Indexing Overhead:** Xây dựng GraphRAG cực kỳ tốn kém thời gian và tiền bạc ở khâu lập chỉ mục (O(N) * LLM Calls cho việc trích xuất thực thể/quan hệ). Trong khi đó, Flat RAG chỉ tốn O(N) phép nội suy Vector vô cùng rẻ.
- **Latency Khi Truy Vấn:** GraphRAG mất thêm khoảng 1.0 giây để LLM trích xuất các Seed Nodes và thực thi hàm Cypher Traverse, khiến phản hồi chậm hơn Flat RAG.
- **Token Usage & Chất Lượng:** Bù lại độ trễ, GraphRAG tiết kiệm token đáng kể ở khâu sinh câu trả lời do không nhồi nhét nội dung rác. Đồng thời, độ chính xác (Faithfulness) cao hơn hẳn do bằng chứng đã được giới hạn bởi các đường truyền liên kết logic minh bạch.

### Action Plan - Ứng Dụng Đồ Án Cuối Khóa
Dự án cuối khóa của tôi sẽ là **"Hệ Thống Phân Tích Chuỗi Cung Ứng & Rủi Ro Doanh Nghiệp Bằng GraphRAG"**:
- **Bối cảnh:** Bài toán kinh tế thường phụ thuộc dây chuyền (Supply Chain Multi-hop): *Ví dụ: Quốc gia A cấm vận -> Công ty B (nhà cung cấp lõi) sụp đổ -> Tập đoàn C mất nguồn nguyên liệu.*
- **Lý do áp dụng:** Flat RAG Vector hoàn toàn thất bại khi phải tư duy logic bắc cầu nhiều tầng lớp. GraphRAG có thể mô phỏng chính xác những kết nối "ẩn" này.
- **Dự định Kỹ Thuật:**
  - Nodes: `Company`, `Industry`, `Geo`, `Product`.
  - Relations: `SUPPLIES`, `AFFECTED_BY`, `COMPETES_WITH`.
  - Entity Resolution: Sử dụng mã định danh duy nhất (LEI hoặc Ticker) kết hợp FAISS Cosine Similarity (ngưỡng 0.90) để gộp các công ty viết tắt hoặc alias.
  - Scale: Chạy Batch Queue trên Celery và thay thế LLM khổng lồ bằng các mô hình SLM trích xuất chuyên dụng (như NuExtract) nhằm tiết kiệm chi phí index hàng nghìn báo cáo tài chính.
