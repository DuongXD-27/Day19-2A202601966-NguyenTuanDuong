# Failure Analysis Report - Lab 19

**Tác giả:** Nguyễn Tuấn Dương

---

## 1. Phân Tích Thực Nghiệm: GraphRAG vs Flat RAG

Dựa trên kết quả đánh giá LLM-as-a-Judge, dưới đây là phân tích ưu nhược điểm của từng hệ thống trong các kịch bản thực tế:

### Ca Lỗi Điển Hình của Flat RAG (GraphRAG Thành Công)
- **Câu hỏi (G02 - Multi-hop):** *"Which company partnered with Sineng Electric and what silicon carbide technology did they develop?"*
- **Sự cố của Flat RAG:** Flat RAG hoạt động dựa trên vector similarity của toàn bộ đoạn văn. Do đó, hệ thống dễ bị đánh lừa bởi các đoạn văn bản chứa nhiều từ khóa *"solar inverters"* hoặc *"semiconductor"* nhưng lại không chứa đúng liên kết nối tiếp trực tiếp giữa `Sineng Electric` và `onsemi`.
- **Cách GraphRAG giải quyết:** GraphRAG sử dụng Node Seed để xác định chính xác điểm khởi đầu `Sineng Electric`, sau đó đi bộ qua đồ thị: `Sineng Electric` -> `PARTNERED_WITH` -> `onsemi` -> `DEVELOPED` -> `EliteSiC power modules`. Do đường dẫn được thiết lập minh bạch 100%, điểm Faithfulness và Multi-hop Reasoning của GraphRAG đạt tối đa (5/5).

### Ca Khó Khăn của GraphRAG (Flat RAG Tỏ Ra Ưu Thế Hơn)
- **Câu hỏi (G05 - Cross-doc Aggregation):** *"Summarize how semiconductor companies develop specialized hardware modules for energy and enterprise systems across multiple articles."*
- **Hạn chế của GraphRAG:** Đây là dạng câu hỏi yêu cầu cái nhìn tổng quan toàn cầu (Global Aggregation). Thuật toán của GraphRAG hiện tại dựa trên Local Traversal (BFS từ các Seed Nodes cụ thể). Khi không xác định được 1-2 Seed Nodes mang tính đại diện cốt lõi, Subgraph được tạo ra sẽ quá hẹp, dẫn đến việc thiếu thông tin bao quát. Flat RAG bù lại có khả năng quét nhiều chunks rời rạc chứa ngữ nghĩa chung để tổng hợp dễ dàng hơn.
- **Đề xuất nâng cấp:** Để khắc phục nhược điểm này, hệ thống cần tích hợp cơ chế **Global Community Summarization** (ví dụ: Thuật toán Leiden/Louvain như trong Microsoft GraphRAG) hoặc triển khai **Hybrid RAG** (lấy song song Top-K Chunks bằng Vector và Subgraph bằng Cypher).

---

## 2. Quá Trình Debugging Trong Dự Án

### Lỗi 1: Model Unavailable (Lỗi 404 từ Groq)
- **Sự cố:** Trong bước trích xuất NER/RE, script cố gọi `llama-3.3-70b-versatile` qua Groq nhưng bị trả về mã lỗi HTTP 404 (Not Found).
- **Phân tích Root Cause:** Provider Groq đã ngừng hỗ trợ hoặc giới hạn quyền truy cập mô hình này trên account.
- **Giải pháp:** Cập nhật file cấu hình `.env` để chuyển sang `openai/gpt-oss-20b` (cho trích xuất) và `openai/gpt-oss-120b` (cho Judge). Tái cấu trúc codebase để tự động linh hoạt inject thông số từ `.env` nhằm tránh hardcode.

### Lỗi 2: KeyError khi xử lý DataFrame Rỗng
- **Sự cố:** Gặp lỗi `AttributeError` / `KeyError` tại khâu chuẩn hóa do batch nạp dữ liệu bị rỗng (không trích xuất được Triples nào).
- **Phân tích Root Cause:** Lỗ hổng trong quy trình xử lý dữ liệu mảng khi không lường trước trường hợp API LLM trả về rỗng vì rate limit hoặc fail parser.
- **Giải pháp:** Cài đặt các bộ bảo vệ (Safeguards) trong hàm xử lý mảng, đảm bảo DataFrame luôn duy trì cấu trúc cột rỗng hợp lệ (schema-compliant) trước khi thực thi lệnh Merge/Join.

### Lỗi 3: Windows Socket Exhaustion (WinError 10055)
- **Sự cố:** Quá trình chạy Jupyter (`nbconvert`) bị sập với lỗi `[WinError 10055] An operation on a socket could not be performed`.
- **Phân tích Root Cause:** Do số lượng HTTP request gửi đến API đồng thời quá lớn, kết hợp với các tiến trình zombie bị kẹt trước đó giữ nguyên trạng thái `TIME_WAIT` trên cổng TCP, dẫn đến việc cạn kiệt cổng mạng khả dụng.
- **Giải pháp:** Xóa sổ các tiến trình `nbconvert` thừa mứa trên hệ điều hành, đợi cổng mạng timeout (release) và thiết lập cơ chế Exponential Backoff & Connection Pooling chặt chẽ hơn.
