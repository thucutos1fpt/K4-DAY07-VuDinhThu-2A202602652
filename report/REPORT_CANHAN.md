# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Vũ Đình Thư
**Mã học viên:** 2A202602652
**Nhóm:** Thiên An
**Ngày:** 20/09/2026

## 1. Khởi động (Warm-up) — Cá nhân

### Độ tương tự Cosine (Cosine Similarity)

**Độ tương tự cosine cao nghĩa là gì?**

Độ tương tự cosine cao nghĩa là hai vector có hướng gần giống nhau. Với text embedding, điều này thường cho thấy hai câu có nội dung hoặc ý nghĩa liên quan.

**Ví dụ có độ tương tự CAO:**

* Câu A: Sinh viên đăng ký học phần trên cổng thông tin.
* Câu B: Người học dùng hệ thống trực tuyến để đăng ký môn học.
* Tại sao tương đồng: Cả hai đều nói về hành động đăng ký môn học bằng hệ thống trực tuyến.

**Ví dụ có độ tương tự THẤP:**

* Câu A: Chính sách giao hàng áp dụng cho đơn nội thành.
* Câu B: Món ăn này có vị cay và đậm đà.
* Tại sao khác: Hai câu thuộc hai chủ đề khác nhau: vận chuyển hàng hóa và mô tả đồ ăn.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**

Cosine similarity đo hướng của vector nên tập trung vào mức độ giống nhau về ý nghĩa, ít bị ảnh hưởng bởi độ dài vector. Trong text embedding, hai câu có độ dài khác nhau vẫn có thể cùng ý nghĩa, nên cosine similarity thường phù hợp hơn khoảng cách Euclid.

### Bài toán tính toán Chunking

**Tài liệu 10.000 ký tự, `chunk_size = 500`, `overlap = 50`. Bao nhiêu chunks?**

* Bước dịch giữa hai chunk: `500 - 50 = 450` ký tự.
* Công thức: `ceil((10000 - 500) / 450) + 1`
* `= ceil(9500 / 450) + 1`
* `= 22 + 1 = 23 chunks`.

**Đáp án:** 23 chunks.

**Nếu overlap tăng lên 100 thì sao?**

Khi overlap là 100, bước dịch chỉ còn `500 - 100 = 400` ký tự nên số chunk tăng lên 25. Overlap lớn giúp thông tin ở ranh giới giữa hai chunk không bị mất ngữ cảnh, nhưng làm tăng số lượng dữ liệu cần lưu và tìm kiếm.

## 2. Hướng tiếp cận của tôi

### Các hàm chia nhỏ

**`SentenceChunker.chunk` — hướng tiếp cận**

Tôi dùng regex `(?<=[.!?])(?:\s+|$)` để nhận diện vị trí kết thúc câu sau dấu chấm, chấm hỏi hoặc chấm than. Sau đó, tôi loại bỏ khoảng trắng thừa và gom số câu theo `max_sentences_per_chunk`. Với chuỗi rỗng hoặc chỉ có khoảng trắng, hàm trả về danh sách rỗng.

**`RecursiveChunker.chunk` / `_split` — hướng tiếp cận**

Thuật toán ưu tiên chia văn bản theo thứ tự: đoạn trống, xuống dòng, dấu chấm, khoảng trắng và cuối cùng là từng ký tự. Nếu đoạn văn bản đã nhỏ hơn hoặc bằng `chunk_size` thì đây là base case và đoạn đó được giữ lại. Nếu không tìm thấy separator phù hợp, chương trình dùng separator tiếp theo; khi hết separator sẽ cắt theo kích thước cố định để tránh lỗi với từ quá dài.

### Lớp `EmbeddingStore`

**`add_documents` + `search` — hướng tiếp cận**

Khi thêm tài liệu, tôi tạo embedding từ `content`, lưu nội dung, metadata, ID và vector embedding thành một record. Khi tìm kiếm, hệ thống tạo embedding cho câu hỏi, tính dot product với embedding của các record và sắp xếp kết quả theo score giảm dần để lấy top-k tài liệu liên quan nhất.

**`search_with_filter` + `delete_document` — hướng tiếp cận**

Với `search_with_filter`, tôi lọc record theo metadata trước rồi mới tìm kiếm tương đồng trên tập tài liệu đã lọc. Cách này giúp tránh trả về thông tin không đúng đối tượng, ví dụ thông tin giảng viên khi câu hỏi thuộc về sinh viên. Với `delete_document`, tôi xóa các record có `metadata["doc_id"]` trùng với ID tài liệu cần xóa và trả về `True` nếu có dữ liệu bị xóa.

### Tác tử `KnowledgeBaseAgent`

**`answer` — hướng tiếp cận**

Agent lấy top-k chunk liên quan nhất từ `EmbeddingStore`, sau đó ghép các chunk thành phần `Context` có đánh số nguồn. Prompt yêu cầu mô hình chỉ trả lời dựa trên context đã cung cấp và nói rõ nếu context không có câu trả lời. Cuối cùng, prompt chứa context và question được gửi vào `llm_fn` để sinh câu trả lời.

## 3. Hoàn thiện code

### Kết quả kiểm thử

```text
====================== 42 passed in 0.20s ======================
```

**Số lượng bài test vượt qua:** **42 / 42**

Tôi đã hoàn thiện các phần `SentenceChunker`, `RecursiveChunker`, `compute_similarity`, `ChunkingStrategyComparator`, `EmbeddingStore` và `KnowledgeBaseAgent`. Các test về chunking, vector store, filter metadata, xóa tài liệu, cosine similarity và RAG agent đều đã pass.

## 4. Dự đoán độ tương tự

| Cặp | Câu A                                           | Câu B                                                  | Dự đoán         | Điểm thực tế | Đúng? |
| --- | ----------------------------------------------- | ------------------------------------------------------ | --------------- | -----------: | ----- |
| 1   | Quy định hoàn tiền trong 7 ngày.                | Khách hàng được nhận lại tiền sau tối đa 7 ngày.       | Cao             |       0.0264 | Không |
| 2   | Sinh viên đăng ký học phần trên cổng thông tin. | Người học dùng hệ thống trực tuyến để đăng ký môn học. | Cao             |       0.0157 | Không |
| 3   | Thư viện mở cửa từ thứ Hai đến thứ Sáu.         | Giờ hoạt động của thư viện là các ngày trong tuần.     | Cao             |       0.0171 | Không |
| 4   | Chính sách giao hàng áp dụng cho đơn nội thành. | Món ăn này có vị cay và đậm đà.                        | Thấp            |       0.0873 | Không |
| 5   | Giảng viên được mượn sách trong 180 ngày.       | Sinh viên được mượn sách trong 10 ngày.                | Trung bình/thấp |       0.0599 | Có    |

**Kết quả bất ngờ nhất**

Các câu có cùng ý nghĩa vẫn cho điểm thấp, trong khi một cặp khác chủ đề lại có điểm cao hơn. Nguyên nhân là bài lab hiện dùng `_mock_embed`, đây là embedding giả lập sinh vector theo hàm hash để test ổn định, không phải mô hình embedding ngữ nghĩa thật. Vì vậy, kết quả này cho thấy chất lượng retrieval phụ thuộc rất nhiều vào embedding backend; khi dùng Local, OpenAI hoặc Gemini embedder thật, các câu cùng nghĩa được kỳ vọng sẽ có score cao hơn.

## 5. Kết quả truy xuất của tôi

> Phần này sẽ được hoàn thiện sau khi nhóm thống nhất bộ tài liệu chung và 5 benchmark query. Tôi sẽ chạy cùng 5 câu hỏi đó trên chiến lược cá nhân của mình để kết quả có thể so sánh công bằng với các thành viên khác.

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? | Câu trả lời của Agent (tóm tắt) |
| - | --------------- | ------------------------------------ | ---------: | ------------------- | ------------------------------- |
| 1 | [Query nhóm 1]  | [Điền sau khi chạy]                  |    [Score] | [Có/Không]          | [Tóm tắt]                       |
| 2 | [Query nhóm 2]  | [Điền sau khi chạy]                  |    [Score] | [Có/Không]          | [Tóm tắt]                       |
| 3 | [Query nhóm 3]  | [Điền sau khi chạy]                  |    [Score] | [Có/Không]          | [Tóm tắt]                       |
| 4 | [Query nhóm 4]  | [Điền sau khi chạy]                  |    [Score] | [Có/Không]          | [Tóm tắt]                       |
| 5 | [Query nhóm 5]  | [Điền sau khi chạy]                  |    [Score] | [Có/Không]          | [Tóm tắt]                       |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** [Điền sau khi chạy] / 5

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác**

Tôi muốn quan sát cách các thành viên lựa chọn kích thước chunk, overlap và metadata filter. Tôi kỳ vọng chiến lược chunk theo heading hoặc mục quy định sẽ giữ ngữ cảnh tốt hơn với tài liệu có cấu trúc, trong khi `FixedSizeChunker` có thể đơn giản hơn nhưng dễ cắt giữa một ý quan trọng.

## Tự đánh giá

| Tiêu chí                      |         Điểm tự đánh giá |
| ----------------------------- | -----------------------: |
| Khởi động (Warm-up)           |                    5 / 5 |
| Hướng tiếp cận của tôi        |                  10 / 10 |
| Hoàn thiện code (42/42 tests) |                  30 / 30 |
| Dự đoán độ tương tự           |                    5 / 5 |
| Kết quả truy xuất của tôi     |          [Điền sau] / 10 |
| **Tổng phần cá nhân**         | **50 / 60 + phần mục 5** |
