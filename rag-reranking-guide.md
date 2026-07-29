# RAG Retrieval Quality — Reranking + Self-Correction

> Đây là tài liệu **hướng dẫn**, không phải code hoàn chỉnh. Các phần "Bạn cần quyết định" là chỗ bạn tự thiết kế — mục tiêu là bạn hiểu *tại sao*, không phải copy-paste.
>
> Tài liệu này **thay thế** phần Phase 5 (Self-Correction) trước đây nằm trong `pharma-ai-assistant-plan.md` — đã gộp threshold/grading/rewrite/escalation vào đây cùng với reranking, vì hai mảng này thuộc cùng một pipeline retrieval và cùng ảnh hưởng lẫn nhau (xem §2 giải thích tại sao).

## Bối cảnh — bug thực tế đã quan sát được

Câu hỏi: *"tôi bị đau bụng và tiêu chảy, tôi nên dùng thuốc gì?"*
Model gọi `search_drug(query="loperamide dosing...")` và `search_drug(query="bismuth subsalicylate...")`
→ Cả hai đều trả về **Paracetamol** chunks (score 0.3–0.5), không liên quan gì đến câu hỏi.

Nguyên nhân (đã trace trong `QdrantVectorSearchService.SearchAsync` — [pharma-ai-assistant-service/Pharma.AiAssistant.Infrastructure/Services/QdrantVectorSearchService.cs:21-56](../pharma-ai-assistant-service/Pharma.AiAssistant.Infrastructure/Services/QdrantVectorSearchService.cs)):
```csharp
var scoredPoints = await qdrantClient.SearchAsync(
    collectionName: col,
    vector: vector,
    limit: (ulong)topK,
    cancellationToken: cancellationToken
);
// không có scoreThreshold, không lọc gì cả — cứ topK là trả, dù score thấp
```
Không có ngưỡng relevance, không có bước xếp hạng lại, không có gì tự sửa khi kết quả tệ — bất kể embedding gần/xa thế nào, cứ đủ `topK` là trả về nguyên, và `SearchDrugTool` chỉ gọi 1 lần rồi thôi.

**Đã scan code thật (2026-07-12)** để biết baseline — xem `AgentRunner.cs`, `SearchDrugTool.cs`, `QdrantVectorSearchService.cs`. Kết luận: hiện tại đã có "xương" (multi-turn tool loop, bounded bởi `MaxTurns`) nhưng **không có "não"** — mọi hành vi self-correction hiện tại chỉ là chỉ dẫn trong system prompt, không phải code:

| Failure mode (theo sơ đồ Microsoft "Agentic RAG") | Trạng thái hiện tại | Bằng chứng |
|---|---|---|
| Rerank / xếp hạng lại candidates | ❌ Không có | Đây là chủ đề chính của tài liệu này |
| Grade retrieved docs | ❌ Không có | `SearchDrugTool.cs` trả raw top-K, không chấm điểm/lọc |
| Rewrite Query | ❌ Không có | Chỉ có gợi ý trong system prompt, không có component rewrite |
| New Search / retry có điều kiện | ⚠️ Một phần | Có turn-budget loop nhưng không "loop-until-good-enough"; hết turn → log warning, trả rỗng |
| Try Alternative Source | ⚠️ Một phần | Chỉ prompt-level: bảo LLM tự trả lời bằng kiến thức chung nếu tool rỗng/lỗi |
| Request Human Help | ❌ Không có | Không có confidence scoring/flag/escalation path |
| Tracing/observability | ❌ Tắt | Aspire `ServiceDefaults` (OpenTelemetry) bị comment `#if DEBUG` trong `Program.cs` |
| Malformed query validation | ⚠️ Một phần | Chỉ check null/whitespace |

**Goal:** Biến `SearchDrugTool` + `AgentRunner` từ "retrieve-once-and-hope" thành một pipeline có: (1) recall rộng, (2) rerank chính xác, (3) grade/threshold quyết định tin hay không, (4) rewrite + retry khi không đạt, (5) escalation rõ ràng khi vượt giới hạn tự sửa, (6) traceable qua OpenTelemetry.

---

## 1. Vì sao rerank và threshold/grading phải đi cùng nhau

Threshold = một ngưỡng score tối thiểu. Nếu tất cả kết quả dưới ngưỡng đó → coi như "không tìm thấy gì liên quan", đừng trả về — rẻ, không tốn thêm LLM call, chỉ so sánh số.

**Vấn đề của threshold một mình:** nó chỉ trả lời được *"có nên tin batch kết quả này không"* (binary, dựa trên **cosine score** — tín hiệu khá nhiễu, xem ví dụ bug ở trên: Paracetamol có cosine score 0.42, không hề thấp một cách rõ ràng). Threshold không tự tách được "trong 5 kết quả, cái nào đúng cái nào rác" — nó chỉ so từng số với 1 ngưỡng cố định.

**Reranking giải quyết một vấn đề khác:** xếp hạng lại candidates bằng một tín hiệu chính xác hơn cosine similarity nhiều. Khi threshold áp dụng **sau** rerank (trên rerank score thay vì cosine score thô), ngưỡng sẽ "sạch" hơn và dễ chọn đúng hơn nhiều.

→ Thứ tự đúng trong pipeline: **recall (Qdrant) → rerank → threshold/grade → (nếu fail) rewrite query → retry → (nếu vẫn fail) escalate.**

---

## 2. Reranking là gì? — Giải thích từ đầu

### Phép ẩn dụ

Hình dung bạn có một thư viện 10,000 cuốn sách, và bạn hỏi thủ thư: *"sách nào nói về tương tác giữa Warfarin và Aspirin?"*

- **Bi-encoder search (cách hiện tại — embedding + cosine similarity):** thủ thư đã đọc trước **tóm tắt bìa sau** của cả 10,000 cuốn, biến mỗi tóm tắt thành 1 "cảm nhận chung" (vector). Khi bạn hỏi, thủ thư so cảm nhận câu hỏi của bạn với cảm nhận của từng cuốn, và đưa ra 20 cuốn "nghe có vẻ gần nhất". Rất nhanh (so sánh 10,000 con số có sẵn), nhưng vì chỉ dựa vào "cảm nhận chung" nên đôi khi đưa nhầm — 1 cuốn về "thuốc giảm đau nói chung" có thể có cảm nhận gần giống cuốn về "tương tác Warfarin-Aspirin cụ thể", dù nội dung thực tế không trả lời được câu hỏi.
- **Reranking (cross-encoder / LLM rerank):** với 20 cuốn đã được chọn sơ bộ ở trên, giờ thủ thư **thực sự đọc lướt nội dung từng cuốn cùng lúc với câu hỏi của bạn** (không chỉ so cảm nhận), rồi xếp hạng lại xem cuốn nào *thực sự* trả lời được câu hỏi. Bước này chậm hơn nhiều lần (đọc kỹ tốn công), nên **chỉ áp dụng cho 20 cuốn đã lọc sơ bộ**, không áp dụng cho cả 10,000 cuốn — đó là lý do người ta gọi đây là **two-stage retrieval**: giai đoạn 1 rẻ + rộng (recall), giai đoạn 2 đắt + hẹp (precision).

### Vì sao bi-encoder (cách hiện tại) không đủ?

Khi embed, query và document được biến thành vector **độc lập với nhau** — model không bao giờ "nhìn thấy" query và document cùng lúc. Nó nén toàn bộ ý nghĩa của cả đoạn văn vào 1 vector 1536 chiều rồi so cosine similarity — mất khá nhiều sắc thái/chi tiết cụ thể.

Reranker (cross-encoder hoặc LLM) thì **đọc query và document cùng lúc trong 1 lần forward pass** (hoặc 1 prompt) — nó có thể "chú ý" (attention) trực tiếp giữa từng từ trong câu hỏi và từng từ trong document → chính xác hơn nhiều, nhưng vì phải làm việc này cho **từng cặp (query, document)** nên không thể chạy trên toàn bộ 10,000 vector như Qdrant/HNSW làm được.

```
Bi-encoder (Qdrant hiện tại):
  embed(query) ──┐
                 ├─→ cosine_similarity  (so 2 vector đã tính sẵn, cực nhanh)
  embed(doc)   ──┘
  → chạy được trên hàng triệu doc

Cross-encoder / LLM rerank:
  score = Model(query, doc)  ← model "đọc" cả 2 cùng lúc, KHÔNG tách vector riêng
  → chậm hơn nhiều, chỉ chạy trên tập nhỏ đã được bi-encoder lọc trước (vd. 20)
```

### Trace cụ thể qua pipeline mới

```
Query: "loperamide dosing for adult diarrhea"

BƯỚC 1 — Bi-encoder recall (Qdrant, như hiện tại nhưng topK RỘNG hơn):
  embed(query) → search Qdrant, topK = 20 (thay vì 5)
  → 20 candidates, score 0.30–0.55 (toàn bộ đều "hơi gần", không ai thực sự chắc chắn đúng)

BƯỚC 2 — Rerank 20 candidates:
  reranker.Score(query, candidate_1)  → 0.91   ← đúng thật, dù bi-encoder score chỉ 0.41
  reranker.Score(query, candidate_2)  → 0.87
  reranker.Score(query, candidate_3)  → 0.12   ← Paracetamol, bi-encoder score 0.45 (cao!) nhưng rerank thấy KHÔNG liên quan
  ... (17 candidates còn lại, đa số điểm thấp)

BƯỚC 3 — Sort theo rerank score, lấy topN = 5:
  → candidate_1, candidate_2, ... (Paracetamol bị đẩy xuống dưới, không lọt top 5)

BƯỚC 4 — Threshold/grade trên 5 kết quả ĐÃ RERANK (xem §3):
  → giờ threshold so trên rerank score (đáng tin hơn cosine score nhiều)
```

### Những điều cốt lõi cần nhớ

1. Bi-encoder (Qdrant hiện tại) = rẻ + rộng, dùng để **thu hẹp** hàng triệu doc xuống một tập nhỏ (recall stage).
2. Reranker = đắt + hẹp, dùng để **xếp hạng chính xác** tập nhỏ đó (precision stage) — không bao giờ chạy trực tiếp trên toàn bộ collection.
3. Reranking không thay thế threshold/grading — nó làm cho tín hiệu score mà threshold dựa vào trở nên đáng tin hơn.
4. topK cho bước recall (Qdrant) nên **rộng hơn** con số cuối cùng bạn muốn trả cho LLM (vd. recall 20 → rerank → chỉ giữ 5), vì mục tiêu bước 1 là "đừng bỏ sót", còn mục tiêu bước 2 là "chọn đúng".

### Ba cách implement reranker — bạn cần chọn 1

| Cách | Ưu điểm | Nhược điểm | Hạ tầng cần thêm |
|---|---|---|---|
| **A. LLM-as-reranker** — dùng `ILlmAdapter.ChatAsync` đã có sẵn, 1 prompt hỏi model chấm điểm/sắp xếp N candidates | Không cần dependency mới, tận dụng OpenAI adapter đã wire sẵn (`Pharma.AiAssistant.Application/Services/ILlmAdapter.cs`) | Thêm 1 LLM call mỗi lần search → tốn thêm latency + cost; chất lượng phụ thuộc prompt engineering | Không cần gì mới |
| **B. Dedicated Rerank API** (Cohere Rerank, Voyage Rerank, Jina Rerank) | Model chuyên train cho task rerank → chính xác hơn LLM-as-reranker, thường rẻ hơn gọi GPT-4o mỗi lần | Thêm 1 external API key/dependency mới, thêm network call ra ngoài | HttpClient + API key mới trong config |
| **C. Local cross-encoder** (vd. `ms-marco-MiniLM-L-6-v2`) qua ONNX Runtime | Miễn phí, không phụ thuộc network, latency thấp và ổn định | Phức tạp nhất để tích hợp trong .NET (cần ONNX model file + tokenizer), project chưa có tiền lệ chạy model cục bộ kiểu này | `Microsoft.ML.OnnxRuntime`, model file, tokenizer |

**Gợi ý (không bắt buộc theo):** bắt đầu với **A (LLM-as-reranker)** — cùng tinh thần "1 LLM call rẻ" mà `RelevanceGraderService` ở §3 dùng, không cần setup gì mới, và bạn học được kỹ thuật prompt-based reranking trước khi cân nhắc B/C sau này nếu cần chính xác hơn hoặc giảm latency.

---

## 3. Threshold / Grading

`RelevanceGraderService` — 2 chiến lược, chọn 1 để bắt đầu đơn giản trước:
- **Threshold-based (rẻ, nên làm trước):** nếu tất cả `SearchResult.Score` (giờ là rerank score, không phải cosine score thô) dưới ngưỡng (vd. `0.5`, config qua env `RAG_RELEVANCE_THRESHOLD`) → `Irrelevant`. Không tốn thêm LLM call.
- **LLM-as-judge (chính xác hơn, làm sau nếu threshold không đủ):** prompt ngắn hỏi model "các đoạn trích này có trả lời được câu hỏi không?" → parse `Relevant`/`Irrelevant`/`Insufficient`.

---

## 4. Query Rewrite + Retry loop

Khi grade = `Irrelevant`/`Insufficient`, đừng dừng lại — viết lại query cụ thể hơn rồi thử lại, tối đa `MaxRewriteAttempts` lần (mặc định `2`, config qua env `RAG_MAX_REWRITE_ATTEMPTS` — tổng cộng tối đa 3 lần search cho 1 tool call, tránh vòng lặp vô hạn/tốn cost).

`IRetrievalGrader` và `IQueryRewriter` nên là **1 LLM call rẻ** mỗi cái (model nhỏ hoặc cùng model hiện tại với prompt ngắn, temperature thấp) — không tái sử dụng `AgentRunner` (tránh vòng lặp lồng vòng lặp).

---

## 5. Escalation — AgentRunner

Sau khi tool trả kết quả "no relevant data" (đánh dấu ở §4), **không** để LLM tự quyết định im lặng — `AgentRunner` cần phát hiện pattern này và emit `EscalationEvent` song song với câu trả lời cuối, để UI luôn hiển thị banner nhất quán thay vì phụ thuộc LLM có "nhớ" nói ra hay không:

```csharp
// AgentRunner.cs — sau khi collect ToolResultBlock[]
var hasUnresolvedRetrieval = toolResults.Any(r => r.Content.Contains("No relevant drug information found after"));
if (hasUnresolvedRetrieval && turns == options.MaxTurns) // hoặc: câu hỏi có vẻ critical (dosage/interaction) — heuristic đơn giản trước
    yield return new EscalationEvent(
        Reason: "Retrieval exhausted rewrite attempts without finding relevant data",
        AttemptedQueries: /* lấy từ tool result */);
```

_Lưu ý: heuristic "câu hỏi critical" ban đầu có thể chỉ là keyword match (`dosage`, `interaction`, `contraindication`, `liều`, `tương tác`) — không cần ML, tinh chỉnh sau khi có dữ liệu thật._

---

## 6. Observability — bật lại tracing

- Bỏ guard `#if DEBUG` quanh `builder.AddServiceDefaults()` / `app.MapDefaultEndpoints()` trong `Program.cs` (Aspire `ServiceDefaults` đã wire sẵn OpenTelemetry, chỉ đang bị tắt ở Release).
- Thêm `ActivitySource` riêng (`"Pharma.AiAssistant.Rag"`) bọc quanh: recall, rerank, grading, rewrite, mỗi tool call — set tag `rag.rerank_score`, `rag.grade`, `rag.attempt`, `rag.rewritten_query` để trace hiển thị đúng bước nào tự sửa lỗi, giống ý "Diagnostic Tools" trong bài Microsoft.
- Không cần Azure AI Tracing cụ thể ngay — `ActivitySource` chuẩn OTel là đủ, export sang bất kỳ backend nào sau (Aspire dashboard lúc dev, Application Insights lúc prod).

---

## 7. Bạn cần quyết định (tự thiết kế, đừng bỏ qua)

Trả lời và **ghi lại lý do** cho mỗi câu — đây chính là phần "kiến trúc sư" của việc này, không phải phần code:

1. **Format output của LLM reranker:** bạn sẽ yêu cầu model trả về JSON array `[{index, score}]`? hay chỉ 1 danh sách index đã sắp xếp? Xử lý thế nào nếu model trả sai format (retry? fallback về thứ tự cosine gốc?)
2. **RecallTopK trước rerank:** nên là bao nhiêu? Đánh đổi giữa "đủ rộng để không bỏ sót chunk đúng" và "không quá nhiều → prompt rerank quá dài, tốn token". Gợi ý thử: 15–25.
3. **Rerank có chạy khi search nhiều collection cùng lúc không?** (Nhắc lại: hiện tại khi không có `drug_class`, `QdrantVectorSearchService.SearchAsync` search **tất cả** collection rồi mới merge+sort — xem dòng 25-55 file đó). Rerank nên chạy **sau khi đã merge** toàn bộ candidates từ mọi collection, hay rerank riêng từng collection rồi mới merge? Cân nhắc: rerank sau-merge đơn giản hơn nhưng nếu có 50 collection × 20 candidates = 1000 candidates thì 1 lần gọi LLM để rerank hết là không khả thi (quá dài) — có thể cần giới hạn tổng số candidates đưa vào reranker.
4. **Khi nào bỏ qua rerank để tiết kiệm cost/latency?** Ví dụ: nếu đã có `drug_class` filter cụ thể (model đã biết chính xác nhóm thuốc) thì candidates vốn đã "sạch" hơn nhiều — có cần rerank không, hay chỉ rerank khi search toàn bộ collection (trường hợp dễ lẫn nhất, đúng như bug hôm nay)?
5. **Cache?** Cùng 1 `(query, drug_class)` có nên cache kết quả rerank để tránh gọi lại LLM mỗi lần user hỏi lại câu tương tự không? (Có thể để sau, không phải ưu tiên ban đầu.)
6. **Ngưỡng threshold (§3) nên là bao nhiêu** sau khi score đã là rerank score thay vì cosine? Rerank score (đặc biệt nếu LLM tự chấm 0-1 hoặc 0-100) có phân bố khác cosine similarity — đừng copy nguyên `0.75` từ ví dụ cosine cũ mà không kiểm tra lại.

---

## 8. Skeleton — types + interfaces, bạn viết phần thân

### 8.1 Domain layer — types mới

```
Pharma.AiAssistant.Domain/
  Ai/
    RetrievalGrade.cs     ← enum: Relevant, Irrelevant, Insufficient
    RetrievalOutcome.cs   ← record RetrievalOutcome(RetrievalGrade Grade, IReadOnlyList<Source> Sources, string? Reason)
```

`StreamEvent` (đã có, `[JsonPolymorphic]`) thêm 1 variant mới:

```csharp
// StreamEvent.cs — thêm variant
public record EscalationEvent(string Reason, IReadOnlyList<string> AttemptedQueries) : StreamEvent;
```

→ UI nhận `EscalationEvent` để hiển thị banner "Cần con người xác nhận" thay vì coi đây là câu trả lời bình thường.

### 8.2 Application layer — interfaces

```csharp
// Pharma.AiAssistant.Application/Services/IRerankerService.cs
namespace Pharma.AiAssistant.Application.Services;

public interface IRerankerService
{
    /// <summary>
    /// Xếp hạng lại candidates theo mức độ liên quan thực sự với query,
    /// trả về topN kết quả tốt nhất (không nhất thiết giữ nguyên thứ tự cosine score đầu vào).
    /// </summary>
    Task<IReadOnlyList<SearchResult>> RerankAsync(
        string query,
        IReadOnlyList<SearchResult> candidates,
        int topN,
        CancellationToken cancellationToken = default);
}
```

```
Pharma.AiAssistant.Application/
  Services/
    IRetrievalGrader.cs   ← Task<RetrievalGrade> GradeAsync(string query, IReadOnlyList<SearchResult> results, ct)
    IQueryRewriter.cs     ← Task<string> RewriteAsync(string originalQuery, string failureReason, ct)
```

```csharp
public record GradeRequest(string Query, IReadOnlyList<SearchResult> Results);
```

### 8.3 Infrastructure layer

```
Pharma.AiAssistant.Infrastructure/
  Services/
    LlmRerankerService.cs       ← implements IRerankerService (§2, cách A)
    RelevanceGraderService.cs   ← implements IRetrievalGrader (§3)
    QueryRewriterService.cs     ← implements IQueryRewriter (§4)
  Tools/
    SearchDrugTool.cs           ← sửa: recall rộng → rerank → grade → rewrite loop
```

```csharp
// Pharma.AiAssistant.Infrastructure/Services/LlmRerankerService.cs
namespace Pharma.AiAssistant.Infrastructure.Services;

public sealed class LlmRerankerService(
    ILlmAdapterResolver llmAdapterResolver,
    ILogger<LlmRerankerService> logger
) : IRerankerService
{
    public Task<IReadOnlyList<SearchResult>> RerankAsync(
        string query,
        IReadOnlyList<SearchResult> candidates,
        int topN,
        CancellationToken cancellationToken = default)
    {
        // TODO (bạn viết):
        // 1. Build prompt: liệt kê candidates đánh số [1]..[N] kèm chunkText,
        //    yêu cầu model trả JSON điểm relevance cho từng số.
        // 2. Gọi llmAdapter.ChatAsync(...) — non-streaming, temperature thấp.
        // 3. Parse response, map lại điểm vào candidates tương ứng.
        // 4. Sort theo điểm mới, Take(topN).
        // 5. Xử lý trường hợp parse lỗi (câu hỏi #1 ở §7).
        throw new NotImplementedException();
    }
}
```

**`SearchDrugTool.ExecuteAsync`** — flow đầy đủ (recall → rerank → grade → rewrite loop):

```csharp
public async Task<ToolExecutionResult> ExecuteAsync(string toolUseId, JsonObject input, CancellationToken ct)
{
    var query = input["query"]!.GetValue<string>();
    var drugClass = input["drug_class"]?.GetValue<string>();
    var attempted = new List<string> { query };

    for (int attempt = 0; attempt <= MaxRewriteAttempts; attempt++)
    {
        var vector = await _embeddingService.GenerateEmbeddingAsync(query, ct);

        var candidates = await _vectorSearchService.SearchAsync(
            vector, drugClass, topK: RecallTopK, ct);          // §2 — recall rộng, câu hỏi #2

        var results = await _reranker.RerankAsync(
            query, candidates, topN: FinalTopN, ct);            // §2 — rerank xuống topN

        var grade = await _grader.GradeAsync(query, results, ct); // §3 — threshold/LLM-judge trên rerank score

        if (grade == RetrievalGrade.Relevant)
            return BuildSuccessResult(toolUseId, results);

        if (attempt == MaxRewriteAttempts) break; // hết lượt tự sửa

        query = await _rewriter.RewriteAsync(query, grade.ToString(), ct); // §4
        attempted.Add(query);
    }

    // Fallback: không phải throw — trả kết quả rỗng có đánh dấu, để AgentRunner/LLM biết mà escalate
    return new ToolExecutionResult(toolUseId,
        $"No relevant drug information found after {attempted.Count} query attempts: {string.Join(" → ", attempted)}. " +
        "Answer must state this explicitly and suggest escalation if this is a critical safety question.",
        IsError: false);
}
```

### 8.4 NuGet packages cần thêm

```
(không cần thêm gì mới nếu chọn cách A (LLM-as-reranker) —
 ILlmAdapter + ActivitySource đều đã có sẵn qua ASP.NET Core / infra hiện tại)
```

---

## 9. Thứ tự làm — đề xuất, không bắt buộc

1. Đọc kỹ §2 (concept reranking) cho đến khi tự giải thích được sự khác biệt bi-encoder vs cross-encoder mà không cần nhìn lại.
2. Trả lời 6 câu hỏi ở §7, ghi ra (kể cả trong comment code hoặc trong tài liệu này) — lý do chọn quan trọng hơn đáp án.
3. Viết `IRerankerService` + `LlmRerankerService` (implementation A — LLM-as-reranker). Test riêng bước này trước (so sánh thứ tự trước/sau rerank cho vài query mẫu) trước khi động vào grading/rewrite.
4. Viết `IRetrievalGrader` (threshold-based trước) + `IQueryRewriter`.
5. Sửa `SearchDrugTool.ExecuteAsync` theo flow đầy đủ ở §8.3.
6. Thêm `EscalationEvent` + logic phát hiện ở `AgentRunner` (§5).
7. Bật lại tracing (§6), verify bằng Aspire dashboard.
8. Test lại đúng câu hỏi gây bug: *"tôi bị đau bụng và tiêu chảy, tôi nên dùng thuốc gì?"* — kỳ vọng: Paracetamol không còn lọt top kết quả, và nếu corpus thực sự không có tài liệu liên quan → trả lời trung thực "không tìm thấy dữ liệu liên quan" thay vì trả nhầm với vẻ tự tin.

---

## 10. Verify

```
1. Query rõ ràng có trong KB (vd. "Warfarin dosage elderly")
   → rerank giữ đúng candidate liên quan ở top, grade = Relevant ngay lần đầu,
     không rewrite, trace chỉ có 1 attempt

2. Query mơ hồ/không có trong KB (vd. "thuốc gì tốt cho tim")
   → sau rerank vẫn Irrelevant lần 1 → rewriter sinh câu cụ thể hơn → search lại
   → nếu vẫn Irrelevant sau MaxRewriteAttempts → tool trả "no relevant data" có đánh dấu
   → nếu câu hỏi match keyword critical → EscalationEvent xuất hiện, UI hiển thị banner

3. Check Aspire dashboard / OTel export → thấy đủ span: recall → rerank → grade → rewrite → retry,
   tag rõ ràng để debug được vì sao 1 câu trả lời bị escalate

4. Retrieval Precision (thủ công, review bằng mắt) — tham khảo §12 `idea-drug-rag-system.md`:
   Trong top-5 trả về, bao nhiêu % thực sự liên quan đến query?
   → So sánh con số này trước/sau khi thêm rerank.
```

---

## Tham khảo thêm

- [Cohere — What is Reranking?](https://docs.cohere.com/docs/reranking)
- [Pinecone — Rerankers and Two-Stage Retrieval](https://www.pinecone.io/learn/series/rag/rerankers/)
- [Cross-Encoders vs Bi-Encoders (Sentence-Transformers docs)](https://www.sbert.net/examples/applications/cross-encoder/README.html)
- Microsoft "Agentic RAG" self-correction pattern — `aka.ms/ai-agents-beginners`
- `pharma-document/idea-drug-rag-system.md` §12 — RAG Evaluation
