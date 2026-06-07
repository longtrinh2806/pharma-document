# AI-Powered Drug Information RAG System

> **Dành cho người mới với AI:** Document này giải thích từng khái niệm từ zero.
> Nếu bạn đã quen .NET/PostgreSQL, hãy nghĩ AI như một layer mới ở trên cùng — phần infrastructure quen thuộc không thay đổi.

---

## 1. Problem Statement

Pharmaceutical professionals (pharmacists, doctors, nurses) frequently need to look up:
- Drug interactions (e.g., "Is it safe to combine Warfarin with Aspirin?")
- Dosage guidelines for specific patient profiles
- Contraindications and side effects
- Drug equivalents / substitutions

Current solutions: static PDFs, disconnected databases, or generic Google searches. None of these provide **context-aware, conversational answers** grounded in authoritative pharmaceutical data.

**Goal:** Build an AI assistant that answers natural language questions about drugs, grounded exclusively in a curated pharmaceutical knowledge base (no hallucination).

---

## 2. Core AI Concepts — Giải Thích Từ Đầu

> Đây là phần quan trọng nhất nếu bạn mới với AI. Đọc kỹ trước khi xem architecture.

### 2.1 LLM (Large Language Model) là gì?

LLM là một model AI được train trên lượng văn bản khổng lồ (internet, sách, wikipedia...).
Nó học cách **predict từ tiếp theo** dựa trên ngữ cảnh — từ đó có khả năng hiểu và sinh ra ngôn ngữ tự nhiên.

```
Ví dụ đơn giản:
  Input:  "The capital of France is ___"
  LLM:    "Paris"  (vì nó đã "đọc" hàng triệu câu như vậy)

Ví dụ phức tạp hơn:
  Input:  "What is the max dose of Metformin for a patient with renal impairment?"
  LLM:    [sinh ra câu trả lời dựa trên training data]
```

**Vấn đề:** LLM được train đến một thời điểm nhất định (knowledge cutoff), không biết về:
- Data nội bộ của bạn (drug database riêng)
- Thông tin mới sau ngày training
- Thông tin chưa từng public

→ Giải pháp: **RAG** (xem Section 3)

---

### 2.2 Token là gì?

Token là đơn vị xử lý của LLM — **không phải ký tự, không phải từ**, mà là mảnh văn bản.

```
Câu: "Metformin is used for diabetes"
Tokens: ["Met", "form", "in", " is", " used", " for", " dia", "betes"]
        = 8 tokens (xấp xỉ)

Rule of thumb:
  1 token ≈ 0.75 từ tiếng Anh
  100 tokens ≈ 75 từ ≈ nửa trang A4

Tại sao quan trọng?
  - LLM có giới hạn "context window" (ví dụ: GPT-4 = 128,000 tokens)
  - Bạn tính tiền theo số tokens (input + output)
  - Chunking chia text thành chunks ≤ 512 tokens để fit vào LLM context
```

---

### 2.3 Embedding là gì? (Khái niệm quan trọng nhất)

**Embedding** là quá trình chuyển đổi text thành một mảng số thực (vector).

Tại sao làm vậy? Vì **máy tính không thể so sánh ngữ nghĩa của chữ, nhưng có thể tính khoảng cách giữa các số**.

```
Text → Embedding Model → Vector (mảng số)

"Metformin max dose is 2550mg"     → [0.12, -0.45, 0.78, 0.33, ...]  (1536 số)
"What is the maximum dose?"        → [0.11, -0.43, 0.76, 0.31, ...]  (1536 số)
"Today's weather is sunny"         → [0.89,  0.23, -0.12, 0.67, ...]  (1536 số)

Nhận xét:
  - "Metformin max dose" và "maximum dose" → vectors GẦN NHAU (cùng nghĩa)
  - "Today's weather" → vector XA (nghĩa hoàn toàn khác)
```

**Hình dung:** Mỗi câu được map vào một điểm trong không gian 1536 chiều. Câu có nghĩa tương tự → điểm gần nhau.

```
                    (không gian vector — hình dung 2D cho dễ)

  "Warfarin dosage" ●
                       ● "How much Warfarin to take?"   ← gần nhau = nghĩa tương tự




                                        ● "Today is Monday"   ← xa = nghĩa khác
```

**Model embedding dùng:** `text-embedding-3-small` của OpenAI
- Input: một đoạn text
- Output: vector 1536 chiều
- Cost: cực rẻ (~$0.02 / 1 triệu tokens)

---

### 2.4 Cosine Similarity / Cosine Distance là gì?

Để đo độ "giống nhau" giữa 2 vectors, ta dùng **cosine similarity**.

```
Cosine similarity = 1   → hoàn toàn giống nhau (cùng hướng)
Cosine similarity = 0   → không liên quan (vuông góc)
Cosine similarity = -1  → trái nghĩa (ngược hướng)

Cosine distance = 1 - cosine similarity
  distance = 0.0  → giống hệt
  distance = 0.25 → khá giống (threshold thường dùng)
  distance = 1.0  → hoàn toàn khác
```

```csharp
// Qdrant tìm top-5 chunks gần nhất với câu hỏi
var results = await qdrantClient.SearchAsync(
    collectionName: "drug_documents",
    vector: questionVector,
    limit: 5,
    scoreThreshold: 0.75f  // score = cosine similarity, cao hơn = giống hơn
);
```

---

### 2.5 Hallucination là gì?

LLM đôi khi "bịa" thông tin một cách tự tin — gọi là **hallucination**.

```
User:  "What is the max dose of Metformin?"
GPT-4: "The maximum dose of Metformin is 3000mg per day."
                                            ↑
                          WRONG! Thực tế là 2550mg — LLM bịa ra con số

Tại sao xảy ra?
  LLM được train để "predict từ hợp lý tiếp theo", không phải "kiểm tra sự thật".
  Nếu training data không rõ ràng, nó sẽ sinh ra câu nghe có vẻ đúng nhưng sai.
```

**RAG giải quyết bằng cách:** ép LLM chỉ được trả lời dựa trên context được cung cấp.
Nếu context không có câu trả lời → nói "Tôi không biết" thay vì bịa.

---

### 2.6 Streaming / SSE là gì?

Thay vì đợi LLM tạo xong toàn bộ câu trả lời rồi mới trả về (có thể mất 10-30 giây),
**streaming** gửi từng token ngay khi được tạo ra — giống như ChatGPT đang "gõ" chữ.

```
Không có streaming:
  [████████████████████████████] 15 giây → nhận được cả đoạn text

Có streaming (SSE):
  "Com" → "bining" → " War" → "farin" → " and" → " Aspirin" → ...
  User thấy text xuất hiện dần dần, UX tốt hơn nhiều
```

**SSE (Server-Sent Events):** Là HTTP protocol cho phép server push data liên tục về client mà không cần WebSocket. React dùng `EventSource` API để nhận SSE.

---

## 3. What is RAG? — Giải Thích Chi Tiết

**Retrieval-Augmented Generation (RAG)** = Tìm kiếm (Retrieval) + Sinh văn bản (Generation).

### Tại sao cần RAG?

```
Cách 1 — Hỏi LLM trực tiếp (KHÔNG an toàn):
  User: "Max dose of Metformin?"
  LLM:  [trả lời từ training data — có thể sai, có thể outdated]
  Risk: Hallucination trong y tế = nguy hiểm

Cách 2 — Fine-tuning LLM trên data của bạn (QUÁ đắt):
  Train lại model với drug database của bạn
  Cost: hàng chục nghìn USD, mất nhiều tuần
  Vấn đề: Mỗi khi data thay đổi phải train lại

Cách 3 — RAG (ĐÚNG):
  1. Lưu drug documents vào vector DB (offline, 1 lần)
  2. Khi user hỏi → tìm documents liên quan → đưa vào prompt → LLM trả lời DỰA TRÊN đó
  Cost: Rẻ, flexible, không cần train lại model
```

### RAG hoạt động như thế nào? (Ví dụ cụ thể)

**Scenario:** User hỏi "Can I take Warfarin with Aspirin?"

```
BƯỚC 1 — Embed câu hỏi:
  Question: "Can I take Warfarin with Aspirin?"
  → Gọi OpenAI Embeddings API
  → Nhận vector: [0.23, -0.11, 0.67, 0.45, ...]  (1536 số)

BƯỚC 2 — Tìm chunks liên quan trong Qdrant:
  qdrantClient.Search(collection: "drug_documents", vector: questionVector, limit: 5)

  Kết quả trả về:
  ┌─────────────────────────────────────────────────────────────┐
  │ Chunk 1 (score: 0.92): "Warfarin and antiplatelet agents   │
  │   such as aspirin significantly increase bleeding risk..."  │
  │ Source: warfarin_prescribing_info.pdf, page 12             │
  ├─────────────────────────────────────────────────────────────┤
  │ Chunk 2 (score: 0.88): "Concomitant use of aspirin with   │
  │   anticoagulants is generally contraindicated unless..."   │
  │ Source: drug_interactions_2024.pdf, page 45               │
  ├─────────────────────────────────────────────────────────────┤
  │ Chunk 3 (score: 0.81): "INR monitoring is essential        │
  │   when aspirin is co-administered with Warfarin..."        │
  │ Source: clinical_guidelines.pdf, page 8                   │
  └─────────────────────────────────────────────────────────────┘

BƯỚC 3 — Xây dựng prompt:
  ┌──────────────────────────────────────────────────────────────┐
  │ System: You are a pharmaceutical assistant. Answer ONLY      │
  │         based on the provided context. If not found, say     │
  │         "I don't have enough information."                   │
  │                                                              │
  │ Context:                                                     │
  │   [Chunk 1 text]                                             │
  │   [Chunk 2 text]                                             │
  │   [Chunk 3 text]                                             │
  │                                                              │
  │ Question: Can I take Warfarin with Aspirin?                  │
  └──────────────────────────────────────────────────────────────┘

BƯỚC 4 — LLM trả lời DỰA TRÊN context:
  "Combining Warfarin and Aspirin significantly increases the risk
   of bleeding. This combination is generally contraindicated unless
   specifically indicated. If co-administration is necessary, close
   INR monitoring is essential.

   Sources: warfarin_prescribing_info.pdf (p.12),
            drug_interactions_2024.pdf (p.45)"
```

### RAG Pipeline — Hai giai đoạn

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GIAI ĐOẠN 1: INGESTION (Offline — chạy một lần khi có data mới)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  PDF Drug Documents
        │
        │ 1. Parse PDF → extract raw text
        ▼
  "Metformin (metformin hydrochloride) Tablets... The maximum
   recommended daily dose of metformin HCl tablets in adults
   is 2550 mg. For children (10-16 years), the maximum daily
   dose is 2000 mg..."
        │
        │ 2. Chunking — chia nhỏ thành chunks 512 tokens
        ▼
  Chunk 1: "Metformin hydrochloride Tablets... The maximum
            recommended daily dose of metformin HCl tablets
            in adults is 2550 mg. For children (10-16 years),"

  Chunk 2: "years), the maximum daily dose is 2000 mg.
            Renal impairment: Metformin is contraindicated
            in patients with eGFR < 30 mL/min..."

  (overlap 50 tokens giữa chunks để không mất context)
        │
        │ 3. Embed mỗi chunk → gọi OpenAI Embeddings API
        ▼
  Chunk 1 → [0.12, -0.45, 0.78, ...]  (1536 floats)
  Chunk 2 → [0.09, -0.41, 0.81, ...]  (1536 floats)
        │
        │ 4. Upsert vào Qdrant collection "drug_documents"
        ▼
  Qdrant point:
  {
    id: "uuid",
    vector: [0.12, -0.45, 0.78, ...],
    payload: { drug_name, content, source_file, page_number, ... }
  }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GIAI ĐOẠN 2: QUERY (Online — mỗi khi user hỏi)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  User: "What is the max dose of Metformin for kids?"
        │
        │ 1. Embed câu hỏi (cùng model với ingestion)
        ▼
  Question vector: [0.10, -0.43, 0.79, ...]
        │
        │ 2. Qdrant cosine similarity search
        ▼
  qdrantClient.Search("drug_documents", questionVector, limit: 5)
  → Chunk về "max daily dose 2000 mg for children" được trả về (score: 0.95)
        │
        │ 3. Inject vào LLM prompt + gọi GPT-4
        ▼
  Answer: "For children aged 10-16 years, the maximum daily
           dose of Metformin is 2000 mg."
           Source: metformin_prescribing_info.pdf, page 3
```

---

## 4. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Client (React/Web)                          │
└───────────────┬────────────────────────────────┬────────────────────┘
                │ REST / SSE (streaming)          │ REST
                ▼                                 ▼
┌──────────────────────┐              ┌─────────────────────┐
│  ai-assistant-service│              │  drug-catalog-service│
│  (RAG Engine)        │◄────────────►│  (Master Drug Data) │
│  - Query handling    │  gRPC/HTTP   │  - CRUD drugs        │
│  - LLM orchestration │              │  - Search/filter     │
│  - Context retrieval │              └─────────────────────┘
└──────────┬───────────┘
           │ vector search
           ▼
┌──────────────────────┐
│  Qdrant              │◄────────────┐
│  (Vector Database)   │             │ upsert embeddings
│  - Drug embeddings   │             │
│  - Semantic search   │  ┌─────────────────────┐
└──────────────────────┘  │  document-service   │
                           │  - PDF ingestion    │
                           │  - Text chunking    │
                           │  - Embedding gen    │
                           └─────────────────────┘

Existing services:
┌─────────────────────┐    ┌─────────────────────┐
│  identity-service   │    │  notification-service│
│  (Auth / 2FA)       │    │  (Email / Push)      │
└─────────────────────┘    └─────────────────────┘

Infrastructure:
┌──────────────────────────────────────────────────────────────────────┐
│  RabbitMQ │ PostgreSQL │ Redis │ Qdrant │ OpenTelemetry │ Docker     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. Microservices Breakdown

### 5.1 `drug-catalog-service`

**Responsibility:** Master data management for drugs.

**Domain entities:**
- `Drug` — name, generic name, drug class, manufacturer
- `DrugInteraction` — drug A + drug B → severity (minor/moderate/severe)
- `Contraindication` — drug + condition → warning
- `Dosage` — drug + patient profile → recommended dose range

**Tech:**
- ASP.NET Core Web API
- PostgreSQL + EF Core
- CQRS with MediatR (read/write separation)
- Full-text search với PostgreSQL `tsvector`

**Key endpoints:**
```
GET  /drugs                    → paginated drug list
GET  /drugs/{id}               → drug detail
GET  /drugs/{id}/interactions  → interactions with other drugs
POST /drugs                    → admin: add drug
POST /drugs/import             → bulk import from JSON/CSV
```

**Events published (RabbitMQ):**
```
DrugCreatedEvent
DrugUpdatedEvent
DrugDeletedEvent
```
→ `document-service` subscribes để trigger re-indexing

---

### 5.2 `document-service`

**Responsibility:** Ingest pharmaceutical documents (PDF, text), chunk, generate embeddings, store vào Qdrant.

**Tech:**
- ASP.NET Core Background Worker
- `PdfPig` — PDF parsing (.NET library, MIT license)
- `OpenAI Embeddings API` (`text-embedding-3-small`) — generate vectors
- `Qdrant.Client` (.NET SDK) — upsert vectors vào Qdrant
- Hangfire — background job scheduling

**Chunking — Tại sao cần overlap?**

```
Document gốc (rút gọn):
  "...Metformin max dose is 2550mg per day in adults.
   For patients with renal impairment (eGFR 30-45),
   dose should be reduced to 1000mg..."

Chunking không overlap (BAD):
  Chunk 1: "...Metformin max dose is 2550mg"
  Chunk 2: "per day in adults. For patients with renal impairment"
  Chunk 3: "(eGFR 30-45), dose should be reduced to 1000mg..."

  → User hỏi: "Max dose for renal impairment?"
  → Không chunk nào chứa đủ context để trả lời!

Chunking với 50-token overlap (GOOD):
  Chunk 1: "...Metformin max dose is 2550mg per day in adults.
            For patients with renal impairment (eGFR 30-45),"

  Chunk 2: "For patients with renal impairment (eGFR 30-45),
            dose should be reduced to 1000mg..."
            ↑ phần overlap

  → User hỏi: "Max dose for renal impairment?"
  → Chunk 2 chứa đủ context!
```

**Processing pipeline:**
```
1. Receive document (PDF upload hoặc DrugUpdatedEvent từ RabbitMQ)
2. PdfPig extract raw text từ PDF
3. Clean & normalize (loại bỏ headers, footers, fix encoding)
4. Chia thành chunks 512 tokens, overlap 50 tokens
5. Với mỗi chunk → gọi OpenAI text-embedding-3-small API
6. Upsert vào Qdrant collection "drug_documents" với payload:
   {
     drug_id, drug_name, drug_class,
     content (text gốc của chunk),
     source_file, page_number, chunk_index
   }
```

---

### 5.3 `ai-assistant-service`

**Responsibility:** The RAG engine. Nhận câu hỏi từ user → tìm context trong Qdrant → gọi LLM → trả về câu trả lời có nguồn.

**Tech:**
- ASP.NET Core Web API
- **Microsoft Semantic Kernel** — AI orchestration (xem Section 8)
- `Qdrant.Client` (.NET SDK) — vector search
- `OpenAI` / `Azure OpenAI` / `Ollama` — LLM provider
- Server-Sent Events (SSE) — streaming từng token về frontend
- Redis — lưu conversation history (multi-turn context)

**Core RAG handler:**
```csharp
public async IAsyncEnumerable<string> AskStreamingAsync(string userId, string question)
{
    // 1. Chuyển câu hỏi thành vector
    var questionVector = await embeddingService.EmbedAsync(question);

    // 2. Tìm top-5 chunks gần nhất trong Qdrant
    var searchResult = await qdrantClient.SearchAsync(
        collectionName: "drug_documents",
        vector: questionVector,
        limit: 5,
        scoreThreshold: 0.75f  // bỏ qua chunks không liên quan
    );

    // Nếu không tìm được context liên quan → trả lời sớm, không gọi LLM
    if (!searchResult.Any())
    {
        yield return "I don't have enough information about this topic in my knowledge base.";
        yield break;
    }

    // 3. Lấy lịch sử hội thoại từ Redis
    var history = await conversationStore.GetHistoryAsync(userId);

    // 4. Build prompt với context từ Qdrant
    var chunks = searchResult.Select(r => r.Payload);
    var prompt = BuildPrompt(question, chunks, history);

    // 5. Gọi LLM và stream từng token về client qua SSE
    var answer = new StringBuilder();
    await foreach (var token in kernel.InvokeStreamingAsync(prompt))
    {
        answer.Append(token);
        yield return token;  // → SSE gửi về React ngay lập tức
    }

    // 6. Lưu vào conversation history để multi-turn hoạt động
    await conversationStore.AppendAsync(userId, question, answer.ToString());
}
```

**Prompt template:**
```
┌──────────────────────────────────────────────────────────────┐
│ You are a pharmaceutical assistant. Answer questions using   │
│ ONLY the provided context below. If the answer cannot be    │
│ found in the context, respond with:                         │
│ "I don't have enough information to answer this accurately." │
│ Do NOT use any external knowledge.                          │
│                                                              │
│ Context:                                                     │
│ --- Source: warfarin_prescribing_info.pdf (page 12) ---     │
│ {chunk_1_text}                                              │
│                                                              │
│ --- Source: drug_interactions_2024.pdf (page 45) ---        │
│ {chunk_2_text}                                              │
│                                                              │
│ Conversation history:                                        │
│ User: [previous question]                                   │
│ Assistant: [previous answer]                                │
│                                                              │
│ Question: {user_question}                                   │
│                                                              │
│ Answer (cite sources at the end):                           │
└──────────────────────────────────────────────────────────────┘
```

**Key endpoints:**
```
POST /chat                → single question, returns full answer (JSON)
POST /chat/stream         → SSE streaming — tokens appear as generated
GET  /chat/{sessionId}    → get conversation history
DELETE /chat/{sessionId}  → clear conversation
```

---

## 6. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Backend framework | ASP.NET Core 9 | API services |
| AI orchestration | Microsoft Semantic Kernel | LLM + plugin management |
| LLM provider | OpenAI GPT-4o / Azure OpenAI / Ollama | Language model |
| Embedding model | text-embedding-3-small | Text → vector (1536 dimensions) |
| **Vector database** | **Qdrant** | **Semantic similarity search** |
| Relational DB | PostgreSQL | Drug master data |
| Cache | Redis | Conversation history, answer cache |
| Message broker | RabbitMQ | Service-to-service events |
| PDF parsing | PdfPig | Extract text from drug leaflets |
| Background jobs | Hangfire | Async document processing |
| Observability | OpenTelemetry + Jaeger | Distributed tracing |
| Containerization | Docker + Docker Compose | Local development |
| Frontend | React + TypeScript | Chat UI |

---

## 7. Qdrant — Vector Database

Qdrant là vector database **purpose-built** cho AI/ML workloads. Chạy như một service độc lập (Docker), có .NET SDK, REST API, và dashboard UI.

### Tại sao Qdrant thay vì pgvector?

```
pgvector (PostgreSQL extension):
  ✅ Không cần service mới
  ❌ Không học được gì mới (đã biết PostgreSQL)
  ❌ Không đúng tinh thần microservices
  ❌ Ít phổ biến hơn trong AI/RAG stack thực tế

Qdrant (dedicated vector DB):
  ✅ Học được công nghệ mới — cực phổ biến trong AI stack
  ✅ Purpose-built: filtering, payload indexing, HNSW tối ưu cho vector search
  ✅ Microservices: service độc lập, scale riêng
  ✅ Dashboard UI tại localhost:6333/dashboard để debug
  ✅ REST + gRPC API, .NET SDK chính thức
```

### Docker setup

```yaml
# docker-compose.yml
services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    ports:
      - "6333:6333"   # REST API + Dashboard UI
      - "6334:6334"   # gRPC
    volumes:
      - qdrant-data:/qdrant/storage
    networks:
      - pharma-network

volumes:
  qdrant-data:
```

Sau khi start: mở `http://localhost:6333/dashboard` để xem collections, search, debug.

### Qdrant Data Model

Qdrant lưu dữ liệu theo **Collections** (tương đương table) chứa **Points** (tương đương row).

```
Collection: "drug_documents"

Point {
  id: "550e8400-e29b-41d4-a716-446655440000",
  vector: [0.12, -0.45, 0.78, ...],   // 1536 floats — embedding của chunk
  payload: {                            // metadata tự do, không cần schema cứng
    "drug_id":     "uuid",
    "drug_name":   "Metformin",
    "drug_class":  "Biguanide",
    "content":     "The maximum recommended daily dose of Metformin...",
    "source_file": "metformin_prescribing_info.pdf",
    "page_number": 4,
    "chunk_index": 12,
    "language":    "en",
    "indexed_at":  "2025-01-15T10:00:00Z"
  }
}
```

### .NET SDK — Tạo collection và upsert

```csharp
// Khởi tạo Qdrant client
var qdrantClient = new QdrantClient("localhost", 6333);

// Tạo collection (chỉ cần làm 1 lần)
await qdrantClient.CreateCollectionAsync(
    collectionName: "drug_documents",
    vectorsConfig: new VectorsConfig
    {
        Params = new VectorParams
        {
            Size = 1536,                      // dimension của text-embedding-3-small
            Distance = Distance.Cosine        // cosine similarity
        }
    }
);

// Upsert chunk sau khi embed
await qdrantClient.UpsertAsync(
    collectionName: "drug_documents",
    points: new[]
    {
        new PointStruct
        {
            Id = Guid.NewGuid().ToString(),
            Vectors = embeddingVector,        // float[] từ OpenAI API
            Payload = new Dictionary<string, Value>
            {
                ["drug_name"]   = "Metformin",
                ["drug_class"]  = "Biguanide",
                ["content"]     = chunkText,
                ["source_file"] = "metformin_prescribing_info.pdf",
                ["page_number"] = 4,
                ["chunk_index"] = 12
            }
        }
    }
);
```

### .NET SDK — Tìm kiếm với filter

```csharp
// Tìm top-5 chunks liên quan, chỉ trong drug class "Biguanide"
var results = await qdrantClient.SearchAsync(
    collectionName: "drug_documents",
    vector: questionEmbedding,
    filter: new Filter
    {
        Must = new[]
        {
            new FieldCondition
            {
                Key = "drug_class",
                Match = new MatchValue { Value = "Biguanide" }
            }
        }
    },
    limit: 5,
    scoreThreshold: 0.75f   // chỉ trả về kết quả có cosine similarity >= 0.75
);

foreach (var result in results)
{
    var content   = result.Payload["content"].StringValue;
    var source    = result.Payload["source_file"].StringValue;
    var page      = result.Payload["page_number"].IntegerValue;
    var score     = result.Score;  // 0.0 → 1.0, cao hơn = giống hơn
    Console.WriteLine($"[{score:F2}] {source} p.{page}: {content[..100]}...");
}
```

---

## 8. Microsoft Semantic Kernel

**Semantic Kernel (SK)** là framework của Microsoft để xây dựng AI applications với .NET.
Tương tự như LangChain (Python) nhưng dành cho C#.

### Tại sao cần SK thay vì gọi OpenAI API trực tiếp?

```
Gọi OpenAI API trực tiếp:
  - Phải tự quản lý retry logic
  - Phải tự format prompt
  - Phải tự implement tool calling
  - Phải tự swap provider khi đổi từ OpenAI → Azure OpenAI

Dùng Semantic Kernel:
  - Provider abstraction: thay 1 dòng config để đổi OpenAI → Azure → Ollama
  - Built-in retry, token counting
  - Plugin system: LLM có thể tự gọi functions của bạn (Agentic AI)
  - Memory/history management
  - Streaming built-in
```

### Setup

```csharp
// Program.cs
builder.Services.AddKernel()
    .AddOpenAIChatCompletion("gpt-4o", apiKey)                        // LLM
    .AddOpenAITextEmbeddingGeneration("text-embedding-3-small", apiKey); // Embedding

// Qdrant memory store via Semantic Kernel connector
builder.Services.AddSingleton<IMemoryStore>(sp =>
    new QdrantMemoryStore(host: "localhost", port: 6333, vectorSize: 1536));

// Register custom plugins — LLM có thể tự gọi những hàm này
builder.Services.AddSingleton<DrugSearchPlugin>();
builder.Services.AddSingleton<DrugInteractionPlugin>();
```

### Plugin System — Agentic AI

Plugin cho phép LLM **tự quyết định** khi nào cần gọi function nào.

```csharp
public class DrugSearchPlugin
{
    [KernelFunction("search_drug_interactions")]
    [Description("Search for known interactions between two drugs. Use this when user asks about combining medications.")]
    public async Task<string> SearchInteractionsAsync(
        [Description("First drug name, e.g. 'Warfarin'")] string drug1,
        [Description("Second drug name, e.g. 'Aspirin'")] string drug2)
    {
        var interactions = await drugCatalogClient.GetInteractionsAsync(drug1, drug2);
        return JsonSerializer.Serialize(interactions);
    }

    [KernelFunction("get_drug_dosage")]
    [Description("Get recommended dosage for a drug given patient profile")]
    public async Task<string> GetDosageAsync(
        [Description("Drug name")] string drugName,
        [Description("Patient profile: age group (adult/pediatric), weight in kg, renal function (normal/impaired)")] string patientProfile)
    {
        var dosage = await drugCatalogClient.GetDosageAsync(drugName, patientProfile);
        return JsonSerializer.Serialize(dosage);
    }
}
```

```
Ví dụ Agentic flow:
  User: "I'm 70kg adult with eGFR=35, can I take Metformin and Warfarin together?"

  LLM tự quyết định:
    Step 1: Gọi get_drug_dosage("Metformin", "adult, 70kg, renal impaired")
            → "Contraindicated for eGFR < 30, reduce dose for eGFR 30-45"
    Step 2: Gọi search_drug_interactions("Metformin", "Warfarin")
            → "No direct interaction, but both affect kidney function"
    Step 3: Tổng hợp kết quả → trả lời đầy đủ cho user

  Bạn không cần code logic "nếu user hỏi X thì gọi Y" — LLM tự suy luận.
```

---

## 9. Frontend Chat UI

```
┌─────────────────────────────────────────────┐
│  Drug Assistant                       [EN▾] │
├─────────────────────────────────────────────┤
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ [AI] Hello! Ask me anything about   │   │
│  │      medications, dosages, or drug  │   │
│  │      interactions.                  │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ [You] Can I take Warfarin with      │   │
│  │       Aspirin?                      │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │ [AI] Combining Warfarin and Aspirin │   │
│  │      significantly increases        │   │
│  │      bleeding risk...               │   │  ← tokens stream in real-time
│  │                                     │   │
│  │      Sources:                       │   │
│  │      • warfarin_prescribing_info    │   │
│  │        .pdf (page 12)               │   │
│  │      • drug_interactions_2024       │   │
│  │        .pdf (page 45)               │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────────────────┐ [Send]│
│  │ Type your question...           │       │
│  └─────────────────────────────────┘       │
└─────────────────────────────────────────────┘
```

**React streaming với SSE:**
```typescript
const askQuestion = async (question: string) => {
    const eventSource = new EventSource(`/api/chat/stream?q=${encodeURIComponent(question)}`);

    eventSource.onmessage = (event) => {
        // Mỗi token LLM tạo ra → nhận được 1 message
        setAnswer(prev => prev + event.data);
    };

    eventSource.onerror = () => eventSource.close();
};
```

**Key UI features:**
- Streaming response — tokens appear as LLM generates (giống ChatGPT)
- Source citations với tên file + số trang
- Conversation history per session
- Suggested questions để user biết hỏi gì
- Copy answer button

---

## 10. Implementation Roadmap

### Phase 1 — Foundation (Week 1–2)
- [ ] Tạo `drug-catalog-service` với CRUD cơ bản
- [ ] Design drug domain model (Drug, Interaction, Dosage, Contraindication)
- [ ] Seed database với sample data (DrugBank open dataset)
- [ ] Docker Compose setup cho tất cả infrastructure (PostgreSQL, Redis, RabbitMQ, **Qdrant**)

### Phase 2 — Document Pipeline (Week 3–4)
- [ ] Tạo Qdrant collection `drug_documents` (vector size 1536, cosine distance)
- [ ] Build `document-service` — PDF ingestion + chunking (PdfPig)
- [ ] Integrate OpenAI Embeddings API (`text-embedding-3-small`)
- [ ] Test semantic search với sample queries qua Qdrant dashboard
- [ ] Subscribe `DrugUpdatedEvent` → auto re-index embedding khi drug data thay đổi

### Phase 3 — AI Assistant (Week 5–6)
- [ ] Setup `ai-assistant-service` với Semantic Kernel + Qdrant.Client
- [ ] Implement RAG pipeline (embed → Qdrant search → prompt → LLM answer)
- [ ] SSE streaming endpoint
- [ ] Multi-turn conversation với Redis (lưu history theo sessionId)
- [ ] Thêm `DrugSearchPlugin` và `DrugInteractionPlugin`

### Phase 4 — Frontend (Week 7)
- [ ] Chat UI component với SSE streaming support
- [ ] Source citation display (tên file + số trang)
- [ ] Conversation history sidebar
- [ ] Tích hợp vào React app hiện tại như page mới

### Phase 5 — Production Hardening (Week 8–9)
- [ ] OpenTelemetry tracing qua tất cả services
- [ ] Rate limiting trên AI endpoints (LLM calls tốn tiền)
- [ ] Answer cache với Redis (cùng câu hỏi → bỏ qua LLM, trả cached)
- [ ] Relevance threshold: nếu score < 0.75 → "I don't know" thay vì hallucinate
- [ ] Prompt injection protection

---

## 11. Key Design Decisions & Tradeoffs

### Chunking strategy

| Strategy | Pros | Cons | Dùng khi |
|---|---|---|---|
| Fixed size (512 tokens) | Simple | Có thể cắt giữa câu | Prototype nhanh |
| Recursive character splitter | Respect paragraph boundaries | Phức tạp hơn | Hầu hết cases |
| Semantic chunking | Chất lượng tốt nhất | Tốn thêm 1 LLM call | Budget không giới hạn |

**Recommendation:** Bắt đầu với recursive character splitter, overlap 10-15%.

---

### LLM Provider

| Provider | Pros | Cons | Cost |
|---|---|---|---|
| OpenAI API | Dễ setup, chất lượng cao | Data ra ngoài, tốn tiền | ~$0.01/1K tokens |
| Azure OpenAI | Cùng model, enterprise compliance | Cần Azure subscription | Tương đương OpenAI |
| Ollama (local) | Miễn phí, private, offline | Chất lượng thấp hơn, cần GPU | Free nhưng cần hardware |

**Recommendation:** OpenAI cho development. Semantic Kernel cho phép swap provider bằng cách đổi 1 dòng config — không cần sửa business logic.

---

### Vector DB

| DB | Pros | Cons |
|---|---|---|
| **Qdrant** (chosen) | Purpose-built, cực nhanh, filtering mạnh, dashboard UI, học được công nghệ mới | Service riêng, thêm 1 container |
| pgvector | Reuse PostgreSQL, không cần service mới | Không học được gì mới, không đúng tinh thần microservices |
| Chroma | Dễ local dev | Chưa production-ready, không có .NET SDK tốt |

**Recommendation:** Qdrant — học được công nghệ AI phổ biến nhất, đúng tinh thần microservices, dashboard UI giúp debug dễ hơn.

---

## 12. Đo lường chất lượng RAG (RAG Evaluation)

```
Các metrics quan trọng:

1. Retrieval Precision
   → Trong top-5 chunks trả về, bao nhiêu % thực sự liên quan?
   → Đo bằng: human review hoặc LLM-as-judge

2. Answer Faithfulness
   → Câu trả lời có bịa thêm thông tin không có trong context không?
   → Tool: RAGAS framework (Python) hoặc custom LLM judge

3. Answer Relevance
   → Câu trả lời có trực tiếp trả lời câu hỏi không?

4. Qdrant score distribution
   → Dùng Qdrant dashboard để xem score histogram
   → Nếu nhiều queries có score < 0.75 → chunking strategy cần cải thiện

Ví dụ test case:
  Question: "Max dose of Metformin for adults?"
  Expected answer chứa: "2550mg"
  Actual answer: "The maximum dose is 2550mg per day"  ✅

  Question: "Can Metformin be used in pregnancy?"
  Context không có thông tin này
  Expected: "I don't have enough information"
  Actual: "Metformin is generally considered safe in pregnancy" ❌ hallucination!
```

---

## 13. Learning Outcomes

Sau khi hoàn thành project này, bạn sẽ có hands-on experience với:

| Skill | Level |
|---|---|
| RAG architecture design | Can design từ đầu |
| Embeddings & vector search | Hiểu sâu cách hoạt động |
| **Qdrant** — dedicated vector DB | Practical experience với AI-native DB |
| Microsoft Semantic Kernel | Build AI-powered .NET services |
| LLM prompt engineering | Context injection, grounding, anti-hallucination |
| Streaming APIs (SSE) | Real-time response |
| Document processing pipelines | Chunking, embedding, indexing |
| Distributed observability | OpenTelemetry tracing |
| AI evaluation | Đo lường RAG quality |

Đây là những **architect-level competencies** giúp phân biệt Senior Engineer với Tech Lead / Solution Architect trong thời đại AI.

---

## 14. Resources

- [Microsoft Semantic Kernel docs](https://learn.microsoft.com/en-us/semantic-kernel/overview/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Qdrant .NET SDK (Qdrant.Client)](https://github.com/qdrant/qdrant-dotnet)
- [Semantic Kernel Qdrant connector](https://github.com/microsoft/semantic-kernel/tree/main/dotnet/src/Connectors/Connectors.Memory.Qdrant)
- [RAG paper (original)](https://arxiv.org/abs/2005.11401)
- [RAGAS - RAG evaluation framework](https://docs.ragas.io/)
- [PdfPig - PDF parsing in .NET](https://uglytoad.github.io/PdfPig/)
- [DrugBank open data](https://go.drugbank.com/releases/latest#open-data)
- [OpenTelemetry for .NET](https://opentelemetry.io/docs/languages/net/)
- [text-embedding-3-small pricing & specs](https://openai.com/api/pricing/)
