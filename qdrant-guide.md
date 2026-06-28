# Qdrant & Vector Database — Hướng dẫn từ đầu

> **Tài liệu chính thức:** https://qdrant.tech/documentation/

---

## 1. Vector Database là gì? Tại sao cần nó?

### Database truyền thống tìm kiếm theo "khớp chính xác"

```sql
SELECT * FROM drugs WHERE drug_name = 'Warfarin'
```

Câu này chỉ tìm được đúng từ "Warfarin". Nếu user hỏi *"thuốc chống đông máu"* hay *"anticoagulant"*, SQL không tìm được — vì không có chữ "Warfarin" trong query.

### Vector Database tìm kiếm theo "ý nghĩa"

Vector DB lưu mỗi đoạn văn bản dưới dạng một mảng số thực (vector). Hai đoạn văn có **ý nghĩa gần nhau** sẽ có **vector gần nhau** trong không gian toán học — dù dùng từ ngữ khác nhau.

```
"Warfarin — thuốc chống đông máu"     → [0.12, -0.45, 0.88, 0.03, ...]  (1536 số)
"Anticoagulant medication Warfarin"   → [0.11, -0.43, 0.86, 0.05, ...]  (gần nhau)
"Paracetamol — giảm đau hạ sốt"      → [0.67,  0.21, -0.34, 0.78, ...] (xa)
```

Khi user hỏi *"thuốc chống đông máu"*, ta embed câu hỏi thành vector, rồi tìm các vector gần nhất trong DB → trả về những đoạn liên quan đến Warfarin dù không cùng từ ngữ.

---

## 2. Embedding là gì?

**Embedding** là quá trình dùng AI model để biến đổi text → vector số thực.

```
Text: "Warfarin gây xuất huyết khi dùng cùng Aspirin"
         ↓ Embedding model (OpenAI text-embedding-3-small)
Vector: [0.021, -0.134, 0.876, 0.003, -0.421, ..., 0.067]  ← 1536 chiều
```

**Tại sao 1536 chiều?** Model cần nhiều chiều để encode đủ thông tin về ngữ nghĩa. Giống như màu sắc cần 3 kênh (R, G, B) — văn bản cần 1536 "kênh ngữ nghĩa".

**Quan trọng:** Phải dùng **cùng một model** cho cả index và search. Nếu index bằng `text-embedding-3-small` (OpenAI) thì search cũng phải dùng `text-embedding-3-small`. Dùng model khác → vector không so sánh được.

---

## 3. Các khái niệm cốt lõi của Qdrant

> https://qdrant.tech/documentation/concepts/

### 3.1 Collection

**Collection = "Table" trong SQL** — nhưng thay vì lưu rows, lưu vectors.

Trong project này, mỗi drug class có 1 collection riêng:
```
drug-class-anticoagulants   ← chứa vectors của tài liệu thuốc chống đông máu
drug-class-antibiotics      ← chứa vectors của tài liệu kháng sinh
drug-class-analgesics       ← chứa vectors của thuốc giảm đau
```

Khi tạo collection, phải khai báo:
- **vector size:** số chiều của vector (1536 cho `text-embedding-3-small`)
- **distance metric:** cách tính "khoảng cách" giữa các vector

```csharp
await qdrantClient.CreateCollectionAsync(
    collectionName: "drug-class-anticoagulants",
    vectorsConfig: new VectorParams
    {
        Size = 1536,
        Distance = Distance.Cosine  // xem phần 3.4
    }
);
```

> Docs: https://qdrant.tech/documentation/concepts/collections/

---

### 3.2 Point

**Point = "Row" trong SQL** — một đơn vị dữ liệu trong Qdrant.

Mỗi Point gồm 3 phần:

```
Point {
    id:      "550e8400-e29b-41d4-a716-446655440000"   ← định danh duy nhất
    vector:  [0.021, -0.134, 0.876, ..., 0.067]       ← 1536 số (embedding của chunk)
    payload: {                                          ← metadata tự do (JSON)
        "chunkText":   "Warfarin tương tác với Aspirin...",
        "drugName":    "Warfarin",
        "fileName":    "warfarin-monograph.pdf",
        "chunkIndex":  3,
        "documentId":  "abc-123"
    }
}
```

Trong project, mỗi chunk của PDF là 1 Point. Document 50 trang → ~200 chunks → 200 Points.

> Docs: https://qdrant.tech/documentation/concepts/points/

---

### 3.3 Payload

**Payload = metadata đính kèm với vector** — có thể lưu bất kỳ JSON nào.

Payload **không ảnh hưởng** đến vector search, nhưng dùng để:
1. **Trả về kết quả có nghĩa** — sau khi tìm được vector gần nhất, đọc `chunkText` để trả cho LLM
2. **Filter** — tìm trong một subset (ví dụ: chỉ tìm documents của Warfarin)

```csharp
// Ví dụ: tìm giống vector nhưng chỉ trong documents của Warfarin
var results = await qdrantClient.SearchAsync(
    collectionName: "drug-class-anticoagulants",
    vector: queryVector,
    filter: new Filter
    {
        Must = [new Condition { Field = new FieldCondition
        {
            Key = "drugName",
            Match = new Match { Text = "Warfarin" }
        }}]
    },
    limit: 5
);
```

> Docs: https://qdrant.tech/documentation/concepts/payload/

---

### 3.4 Distance Metric — Cosine Similarity

Qdrant hỗ trợ 3 cách đo "khoảng cách" giữa 2 vectors:

| Metric | Dùng khi |
|--------|----------|
| **Cosine** | Text search, semantic similarity — **phổ biến nhất** |
| Dot Product | Khi vectors đã normalize, nhanh hơn Cosine |
| Euclidean | Image/audio embedding |

**Cosine Similarity** đo **góc** giữa 2 vectors (không phải độ dài):
- Score = **1.0** → giống hoàn toàn (cùng hướng)
- Score = **0.0** → không liên quan (vuông góc)
- Score = **-1.0** → ngược nghĩa

```
Query: "Warfarin liều dùng"
→ "Warfarin — liều khởi đầu 2-5mg/ngày"    score: 0.92  ✅ rất liên quan
→ "Warfarin cơ chế tác dụng ức chế Vitamin K"  score: 0.78  ✅ liên quan
→ "Paracetamol 500mg giảm đau"              score: 0.21  ❌ không liên quan
```

> Docs: https://qdrant.tech/documentation/concepts/search/

---

### 3.5 HNSW Index

Qdrant dùng **HNSW (Hierarchical Navigable Small World)** — thuật toán tìm kiếm vector gần nhất hiệu quả.

**Vấn đề:** Với 1 triệu vectors, so sánh query với tất cả 1M vectors = quá chậm (Brute Force).

**HNSW giải quyết:** Xây dựng đồ thị phân cấp — tìm kiếm approximate nearest neighbor với độ chính xác ~99% nhưng nhanh hơn hàng nghìn lần.

Bạn không cần config HNSW — Qdrant tự xây dựng khi upsert points. Chỉ cần biết: **tìm kiếm Qdrant rất nhanh dù có hàng triệu records.**

---

## 4. Full Flow trong Pharma Project

### 4.1 Ingestion (pharma-document-service)

```
PDF upload
    ↓
PdfTextExtractor → raw text
    ↓
TextChunkingService → ["chunk1...", "chunk2...", ...]  (mỗi chunk ~500 tokens)
    ↓
EmbeddingService (OpenAI) → [[0.02, -0.13, ...], [0.45, 0.67, ...], ...]
    ↓
QdrantVectorStoreService.UpsertBatchAsync()
    → collection: "drug-class-anticoagulants"
    → mỗi chunk = 1 Point { id, vector, payload{chunkText, drugName, ...} }
```

### 4.2 Search (pharma-ai-assistant-service — Phase 4)

```
User: "Warfarin tương tác với Amiodarone?"
    ↓
LLM → gọi tool: search_drug(query="Warfarin Amiodarone interaction", drug_class="anticoagulants")
    ↓
SearchDrugTool.ExecuteAsync()
    ↓
EmbeddingService (OpenAI, cùng model) → embed query → [0.11, -0.43, ...]
    ↓
QdrantVectorSearchService.SearchAsync()
    → SearchAsync("drug-class-anticoagulants", queryVector, limit=5)
    ↓
Qdrant trả về top-5 Points có vector gần nhất
    ↓
Đọc payload["chunkText"] từ mỗi Point
    ↓
Trả về context cho LLM:
    "[Warfarin — warfarin-monograph.pdf] (score: 0.91)
     Amiodarone tăng nồng độ Warfarin do ức chế CYP2C9..."
    ↓
LLM dùng context để trả lời có căn cứ
```

---

## 5. Qdrant vs pgvector

| | **Qdrant** | **pgvector (PostgreSQL)** |
|---|---|---|
| Loại | Vector DB chuyên dụng | Extension của PostgreSQL |
| Performance | Rất cao (HNSW native) | Tốt nhưng kém hơn Qdrant |
| Filtering | Mạnh, kết hợp vector + payload filter | SQL WHERE clause |
| Quản lý | Riêng biệt (Docker container) | Cùng PostgreSQL instance |
| Dùng khi | RAG, semantic search, hàng triệu vectors | Đã có PostgreSQL, scale nhỏ |

Project này dùng **Qdrant** — document-service đã setup sẵn.

---

## 6. Qdrant Client trong .NET

> Docs: https://qdrant.tech/documentation/interfaces/

### Setup

```csharp
// DI registration
services.AddSingleton(new QdrantClient("localhost", 6334));
```

### Các operations thường dùng

```csharp
// List tất cả collections
var collections = await client.ListCollectionsAsync();
// → ["drug-class-anticoagulants", "drug-class-antibiotics", ...]

// Tìm kiếm vector gần nhất
var results = await client.SearchAsync(
    collectionName: "drug-class-anticoagulants",
    vector: new float[] { 0.021f, -0.134f, ... },  // query vector
    limit: 5                                         // top-5 kết quả
);

// Mỗi result có:
foreach (var hit in results)
{
    var score = hit.Score;                                    // 0.0 → 1.0
    var chunkText = hit.Payload["chunkText"].StringValue;    // nội dung chunk
    var drugName  = hit.Payload["drugName"].StringValue;     // tên thuốc
}

// Upsert points (insert hoặc update)
await client.UpsertAsync(collectionName, new List<PointStruct>
{
    new PointStruct
    {
        Id = new PointId { Uuid = Guid.NewGuid().ToString() },
        Vectors = new float[] { 0.021f, -0.134f, ... },
        Payload = { ["chunkText"] = "Warfarin..." }
    }
});
```

---

## 7. Qdrant Dashboard

Qdrant có web UI tại `http://localhost:6333/dashboard` khi chạy Docker.

Có thể:
- Xem danh sách collections
- Browse points trong collection
- Chạy search thử trực tiếp trên UI

---

## Tham khảo thêm

- **Quickstart:** https://qdrant.tech/documentation/quickstart/
- **Concepts (full):** https://qdrant.tech/documentation/concepts/
- **Collections:** https://qdrant.tech/documentation/concepts/collections/
- **Points:** https://qdrant.tech/documentation/concepts/points/
- **Search:** https://qdrant.tech/documentation/concepts/search/
- **.NET Client:** https://github.com/qdrant/qdrant-dotnet
- **Payload & Filtering:** https://qdrant.tech/documentation/concepts/filtering/
- **HNSW explained:** https://qdrant.tech/articles/filtrable-hnsw/
