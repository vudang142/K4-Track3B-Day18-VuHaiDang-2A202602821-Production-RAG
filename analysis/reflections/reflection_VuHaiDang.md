# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** VuHaiDang  
**MSSV:** 2A202602821  
**Khóa:** K4 - Track 3B  
**Ngày hoàn thành:** 4/10/2026

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Map từng concept trong lecture vào code đã implement trong lab:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích |
|----------------|--------|-------------|--------------------------|
| Semantic chunking | M1 | `chunk_semantic()` | Threshold 0.85 tạo X chunks vs basic Y chunks; bảo toàn ngữ nghĩa câu liên quan. Cosine similarity giữa embeddings giúp nhóm câu cùng chủ đề, tránh cắt giữa ý. Tuy nhiên, semantic chunking chậm hơn basic vì cần encode tất cả sentences. |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | RRF kết hợp điểm xếp hạng lexical (từ khóa chính xác) và semantic (ngữ nghĩa). BM25 đặc biệt hiệu quả với tiếng Việt nhờ underthesea tokenization, bắt được compound words như "nghỉ_phép". Dense search bắt được synonyms và semantic similarity. RRF với k=60 là sweet spot tránh over-fitting. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Latency ~150-200ms cho 20 docs; tăng độ chính xác top-3 kết quả từ top-20 candidate. Cross-encoder xử lý query-document pair nên hiểu context tốt hơn bi-encoder. Reranking là bước tốn latency nhất nhưng đáng giá vì precision quan trọng hơn recall trong RAG. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Đánh giá 4 chỉ số: Faithfulness (độ trung thực answer vs context), Answer Relevancy (answer vs question), Context Precision (relevant docs vs retrieved), Context Recall (relevant docs vs all relevant). Faithfulness và Context Recall thường thấp nhất vì retrieval chưa perfect. Diagnostic tree giúp map metric → root cause → fix. |
| Contextual embeddings | M5 | `_enrich_single_call()` | Giảm retrieval failure bằng cách bổ sung context tóm tắt trước chunk. Combined mode (1 API call/chunk) tiết kiệm cost. Extractive fallback hoạt động khi không có API key. HyQA generate questions giúp bridge vocabulary gap. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

### Lỗi kỹ thuật gặp phải (Exact error messages):

1. **Qdrant connection timeout**
   ```
   httpx.ConnectTimeout: Connection timeout
   ```
   - **Nguyên nhân:** Docker Qdrant chưa khởi động hoặc port 6333 bị block
   - **Cách debug:** Kiểm tra `docker ps` và `docker compose up -d`
   - **Fix:** Thêm fallback `:memory:` mode trong `DenseSearch.__init__()`

2. **underthesea import error**
   ```
   ModuleNotFoundError: No module named 'underthesea'
   ```
   - **Nguyên nhân:** Chưa install `underthesea`
   - **Cách debug:** `pip install underthesea`
   - **Fix:** Đã thêm vào requirements.txt

3. **sentence_transformers CrossEncoder crash**
   ```
   XLMRobertaTokenizer error với transformers>=5.0
   ```
   - **Nguyên nhân:** FlagReranker không tương thích với transformers mới
   - **Cách debug:** Dùng `sentence_transformers.CrossEncoder` thay vì FlagReranker
   - **Fix:** Đã implement đúng cách

4. **BM25 query không match Vietnamese compound words**
   ```
   Query "nghỉ phép" → 0 results
   ```
   - **Nguyên nhân:** underthesea tokenize thành "nghỉ_phép" (1 token) nhưng query split thành 2 tokens
   - **Cách debug:** In ra tokenized text và query
   - **Fix:** Thêm `.replace("_", " ")` sau word_tokenize

5. **RAGAS evaluation failed (no API key)**
   ```
   ⚠️ RAGAS evaluation failed: OpenAI API key not found
   ```
   - **Nguyên nhân:** Chưa set OPENAI_API_KEY trong .env
   - **Cách debug:** Check `.env` file và `echo $OPENAI_API_KEY`
   - **Fix:** Đã wrap trong try/except với fallback scores = 0

### Kiến thức còn thiếu & Cách bổ sung:

- **Chunking strategy selection:** Chưa hiểu rõ khi nào dùng semantic vs hierarchical vs structure-aware
  - **Cách khắc phục:** Thực hành A/B test với `compare_strategies()` để so sánh

- **RRF parameter tuning:** Chưa biết optimal k value cho different use cases
  - **Cách khắc phục:** Đọc paper "Reciprocal Rank Fusion" và experiment với k=60 vs k=100

- **Multi-hop retrieval:** Chưa implement được multi-hop cho complex questions
  - **Cách khắc phục:** Tìm hiểu GraphRAG và query decomposition techniques

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

Dựa trên những kỹ thuật đã học và thực hành, lập kế hoạch cụ thể áp dụng vào project của bạn:

### Project: [HR Policy Chatbot với RAG]

#### 1. Hiện trạng
- **Pipeline hiện tại:** Basic RAG với LangChain + ChromaDB, simple text chunking (500 chars), dense-only retrieval
- **Vấn đề / Bottlenecks đang gặp:**
  - Context precision thấp (0.58) — retrieve nhiều irrelevant docs
  - Không handle được negation queries ("không được", "không phải")
  - Version conflicts (policy v2023 vs v2024)
  - Latency cao với large corpus (>1000 docs)

#### 2. Kế hoạch cải tiến

1. **Chunking strategy:** Chọn **Hierarchical** (parent=2048, child=256)
   - **Lý do:** Parent-child hierarchy tốt cho HR policy vì:
     - Child chunks cho precision retrieval (256 chars vừa đủ cho 1 policy point)
     - Parent chunks cung cấp full context khi cần
     - Metadata có `parent_id` để trace back

2. **Search retrieval:** Chọn **Hybrid (BM25 + Dense + RRF)**
   - **Lý do:**
     - BM25 với underthesea bắt được Vietnamese compound words ("nghỉ_phép", "bảo_hiểm")
     - Dense (bge-m3) bắt được synonyms và semantic similarity
     - RRF fusion kết hợp cả 2, robust hơn
   - **Parameters:** BM25 top_k=20, Dense top_k=20, RRF k=60

3. **Reranking:** **Có dùng Cross-encoder**
   - **Model:** `BAAI/bge-reranker-v2-m3`
   - **Lý do:** Top-3 từ top-20 candidate, cần reranking để precision cao
   - **Latency budget:** ~150-200ms acceptable cho HR chatbot (không real-time critical)

4. **Evaluation:** Dùng **RAGAS 4 metrics** + custom HR-specific metrics
   - **Metrics:** Faithfulness, Answer Relevancy, Context Precision, Context Recall
   - **Custom:** Negation accuracy (TP/TP+FP cho "không"/"không phải" queries)
   - **Benchmark:** Target ≥ 0.75 cho all metrics

5. **Enrichment:** Áp dụng **Combined Mode** (`_enrich_single_call`)
   - **Techniques:** Summary + HyQA + Contextual Prepend + Metadata Extraction
   - **Lý do:** 1 API call/chunk tiết kiệm cost, đủ information cho retrieval
   - **Metadata fields:** `version`, `effective_date`, `category`, `policy_type`

#### 3. Timeline triển khai

- **Tuần 1: Infrastructure Setup**
  - [ ] Setup Docker với Qdrant
  - [ ] Implement hierarchical chunking cho existing corpus
  - [ ] Pre-download embedding models (bge-m3, bge-reranker-v2-m3)

- **Tuần 2: Search & Retrieval**
  - [ ] Implement BM25 + underthesea tokenization
  - [ ] Implement Dense search với Qdrant
  - [ ] Implement RRF fusion
  - [ ] A/B test vs baseline (dense-only)

- **Tuần 3: Reranking & Evaluation**
  - [ ] Implement Cross-encoder reranking
  - [ ] Setup RAGAS evaluation pipeline
  - [ ] Run evaluation trên test set (20 questions)
  - [ ] Identify bottom-5 failures

- **Tuần 4: Enrichment & Optimization**
  - [ ] Implement combined enrichment pipeline
  - [ ] Add metadata extraction (version, category)
  - [ ] Tune RRF k parameter
  - [ ] Final evaluation và benchmark report

- **Tuần 5: Production & Monitoring**
  - [ ] Deploy lên staging
  - [ ] Setup logging cho retrieval quality
  - [ ] A/B test với production users
  - [ ] Iterate based on user feedback
