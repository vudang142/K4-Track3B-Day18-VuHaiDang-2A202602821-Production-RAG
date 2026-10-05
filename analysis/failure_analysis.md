# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** VuHaiDang  
**MSSV:** 2A202602821  
**Khóa:** K4 - Track 3B

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.6523 | 0.7845 | +0.1322 |
| Answer Relevancy | 0.7012 | 0.7823 | +0.0811 |
| Context Precision | 0.5834 | 0.7345 | +0.1511 |
| Context Recall | 0.6234 | 0.7156 | +0.0922 |

---

## Bottom-5 Failures

### #1
- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Expected:** KHÔNG. Nhân viên thử việc KHÔNG được nghỉ phép năm. Nếu cần nghỉ, phải xin nghỉ không lương và được trưởng phòng phê duyệt.
- **Got:** Nhân viên thử việc được nghỉ phép theo quy định của công ty.
- **Worst metric:** Faithfulness
- **Error Tree:** Output sai → Context có phần liên quan nhưng thiếu thông tin quan trọng → Query OK nhưng retrieval trả về doc tổng quát về nghỉ phép → Chunking không tách được chi tiết thử việc
- **Root cause:** Chính sách nghỉ phép nằm ở file `thu_viec.md` nhưng retrieval không ưu tiên đúng file. Context có thông tin tổng quát về nghỉ phép nhưng không có phần loại trừ cho thử việc.
- **Suggested fix:** Thêm metadata filtering theo category="hr" + content_type="thu_viec" khi query. Hoặc dùng HyQA để generate questions như "Nhân viên thử việc có được nghỉ phép không?" để index.

### #2
- **Question:** Thông tin lương thuộc cấp độ phân loại dữ liệu nào?
- **Expected:** Theo quy chế chi trả lương, thông tin lương được phân loại là dữ liệu Bí mật, cấm chia sẻ với đồng nghiệp. Theo chính sách phân loại dữ liệu, dữ liệu Bí mật (cấp 3) phải mã hóa khi truyền và hạn chế truy cập theo need-to-know.
- **Got:** Thông tin lương thuộc cấp độ "Bí mật" theo chính sách phân loại dữ liệu.
- **Worst metric:** Context Recall
- **Error cause:** Retrieve được phần phân loại dữ liệu nhưng KHÔNG retrieve được phần quy chế chi trả lương → thiếu context về "cấm chia sẻ với đồng nghiệp"
- **Error Tree:** Output thiếu thông tin → Context đúng nhưng không đầy đủ → Query ambiguous ("cấp độ phân loại") → Hybrid search không merge đủ documents liên quan
- **Root cause:** Query cần kết hợp 2 concepts: (1) phân loại dữ liệu, (2) quy chế chi trả lương. RRF fusion với top_k=20 có thể không cover đủ cả 2 sources.
- **Suggested fix:** Tăng hybrid_top_k hoặc dùng query expansion để decompose thành 2 sub-queries.

### #3
- **Question:** Nhân viên được nghỉ bao nhiêu ngày phép năm?
- **Expected:** Theo chính sách hiện hành (v2024), nhân viên được nghỉ 15 ngày phép năm có lương. Chính sách cũ (v2023) là 12 ngày nhưng đã bị thay thế.
- **Got:** Nhân viên được nghỉ 12 ngày phép năm có lương.
- **Worst metric:** Answer Relevancy
- **Error cause:** Retrieve được version cũ (v2023) thay vì version mới (v2024)
- **Error Tree:** Output sai version → Context có thông tin nhưng từ document cũ → Query "phép năm" không phân biệt version → Chunking không annotate version metadata
- **Root cause:** Cả 2 versions (v2023, v2024) đều có chunk về "phép năm". BM25 score tương đương, Dense similarity tương đương → RRF không ưu tiên document mới hơn. Metadata không có field "version" hoặc "effective_date".
- **Suggested fix:** Thêm metadata extraction với field "version"/"effective_date" trong M5. Khi search, boost documents có version mới hơn.

### #4
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Nhân viên Senior 9 năm thâm niên được nghỉ 18 ngày phép năm. [Lương không được trả lời đầy đủ]
- **Worst metric:** Context Recall
- **Error cause:** Multi-hop query cần 2 pieces of info: (1) cách tính phép thâm niên, (2) mức lương Senior. Retrieve được phần phép thâm niên nhưng KHÔNG retrieve được bảng lương.
- **Error Tree:** Output thiếu thông tin → Context không cover đủ 2 aspects → Query quá rộng ("Senior + thâm niên + lương") → Single retrieval không đủ
- **Root cause:** Đây là multi-hop question. Hybrid search chỉ retrieve top-20 chunks cho 1 query. Lương và phép năm nằm ở 2 documents khác nhau.
- **Suggested fix:** Implement multi-hop retrieval: decompose query → parallel retrieve → merge contexts. Hoặc dùng parent-child retrieval để get thêm context.

### #5
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Nhân viên bị phạt 300.000 VNĐ (2%/tháng × 15 triệu) nhưng không giải thích rõ cách tính pro-rata.
- **Worst metric:** Faithfulness
- **Error cause:** Answer đúng số nhưng thiếu breakdown chi tiết về pro-rata calculation
- **Error Tree:** Output đúng số nhưng không đầy đủ → Context có thông tin nhưng LLM không extract đủ → Retrieval OK, generation có thể bị length limit
- **Root cause:** Context có đủ thông tin (15 ngày, 2%, pro-rata) nhưng LLM generation bị truncation hoặc không structure output tốt.
- **Suggested fix:** Cải thiện prompt template để yêu cầu "Trình bày cách tính chi tiết" thay vì chỉ đưa kết quả.

---

## Case Study (cho presentation)

**Question chọn phân tích:** Nhân viên thử việc có được nghỉ phép năm không?

**Error Tree walkthrough:**
1. Output đúng? → **SAI** — Model trả lời "có được nghỉ phép" trong khi thực tế KHÔNG
2. Context đúng? → **Có liên quan nhưng thiếu chi tiết** — Context chứa thông tin về nghỉ phép nói chung, không có phần loại trừ cho thử việc
3. Query rewrite OK? → **OK** — Query "nhân viên thử việc có được nghỉ phép năm không" đã capture intent
4. Fix ở bước: **Retrieval + Enrichment**
   - Thêm metadata filter: `category="hr"` và `document_type="thu_viec"`
   - Hoặc dùng HyQA để generate "Nhân viên thử việc có được nghỉ phép không?" và index cùng chunk

**Nếu có thêm 1 giờ, sẽ optimize:**
- Implement query expansion: thêm synonyms như "nhân viên mới", "nhân viên trong thời gian thử việc"
- Thêm RRF với 3 retrieval methods: BM25 + Dense + keyword boost cho document type
- Dùng contextual prepend để annotate: "Đoạn này nói về chính sách nghỉ phép dành cho nhân viên thử việc"
