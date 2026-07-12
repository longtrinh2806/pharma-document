# pharma-ai-assistant-service — Implementation Plan

> **Stack:** .NET 10, Clean Architecture (API / Application / Domain / Infrastructure)  
> **Pattern:** Re-implement Anthropic open-multi-agent framework in C#, từng layer một  
> **Rule:** Phase sau mới implement → Phase trước `throw new NotImplementedException()`
> **LLM runtime:** Ollama (local) thay vì Anthropic API — `ILlmAdapter` là abstraction layer, có thể swap sau.

---

## Trạng thái tổng quan

_Cập nhật 2026-07-11 — scan trực tiếp code, không dựa vào draft cũ._

| Phase | Mô tả | Trạng thái |
|-------|-------|-----------|
| 1 | Core types + ILlmAdapter + OllamaAdapter + basic chat | ✅ Done (diverged — xem chi tiết) |
| 2 | Tools infrastructure + AgentRunner | ✅ Done (diverged — xem chi tiết) |
| 3 | Chat history (multi-turn) + sliding window summary | ✅ Done (diverged — xem chi tiết) |
| 4 | RAG + Qdrant search | ✅ **Done** (trước đây ghi nhầm ⏳ Next — xem chi tiết) |
| 5 | Self-Correction (grade → rewrite → retry → fallback → escalate) | ❌ Chưa bắt đầu — chưa có file nào |
| 6 | Multi-agent (TaskQueue + AgentPool) | ❌ Chưa bắt đầu — chưa có file nào |
| 7 | Coordinator + Orchestrator | ❌ Chưa bắt đầu — chưa có file nào |

⚠️ **Security flag:** `OPENAI_API_KEY` thật (dạng `sk-proj-...`) đang hard-code trong `Pharma.AiAssistant.API/Properties/launchSettings.json` — file này thường bị commit vào git. Nên chuyển sang user-secrets/env thật và rotate key nếu đã từng push lên remote.

---

## AgentRunner — ✅ Done

File thực tế: `Pharma.AiAssistant.Infrastructure/Services/AgentRunner.cs`, implement `IAgentRunner` (`Pharma.AiAssistant.Application/Services/IAgentRunner.cs` — chỉ có `StreamAsync(IList<LlmMessage>, ct) : IAsyncEnumerable<StreamEvent>`, không có `RunAsync`).

Flow hiện tại:
```
StreamMessageHandler → agentRunner.StreamAsync()
                           ↓
                    while (turns++ < ChatOptions.MaxTurns):     ← mặc định 10, env LLM_MAX_TURNS
                        llmAdapter.StreamAsync()                 ← stream từng turn, forward ThinkingChunk/TextChunk ngay
                        collect ToolUseBlock[]
                        nếu KHÔNG có tool call → append assistant message, yield DoneEvent(RunResult), return
                        nếu CÓ tool call →
                            Task.WhenAll(toolExecutor.ExecuteAsync)  ← parallel, catch lỗi từng tool riêng
                            yield ToolResultEvent (+ SourcesEvent nếu tool trả citation)
                            append "user" message chứa ToolResultBlock[] → conversation
                            loop tiếp
                    hết MaxTurns mà chưa có turn cuối → log warning, yield DoneEvent rỗng
```

Divergence thực tế so với plan gốc:
- Stream **từng turn** (không phải ChatAsync + StreamAsync cuối) → user thấy thinking realtime
- `AgentRunner` nằm ở **Infrastructure** (không phải Domain) vì depend vào `ILlmAdapter`
- `IAgentRunner` chỉ có `StreamAsync` (không có `RunAsync`) — đủ cho use case hiện tại
- Tool execution error bị **catch**, trả về error message thay vì crash → agent tiếp tục
- `RunOptions` như plan gốc mô tả **không tồn tại** — cấu hình turns/model/thinking nằm ở `ChatOptions` (Application/Models), build 1 lần thành singleton trong DI
- `StreamEvent` (Domain/Ai) là base type polymorphic (`[JsonPolymorphic]`, discriminator `"type"`) với các variant: `TextChunk`, `ThinkingChunk`, `ToolUseEvent`, `ToolResultEvent`, `SourcesEvent(IReadOnlyList<Source>)`, `DoneEvent(RunResult)` — `Source(DrugName, FileName, Score, ChunkText)` dùng để trả citation chip về UI khi RAG tool trả kết quả

---

## Project Structure (tạo 1 lần ở đầu, không đổi)

```
pharma-ai-assistant-service/
  pharma-ai-assistant-service.sln
  Directory.Build.props              ← copy từ identity-service
  docker-compose.yml
  Dockerfile
  global.json
  Pharma.AiAssistant.API/            ← HTTP endpoints, DI wiring
  Pharma.AiAssistant.Application/    ← Use cases (ChatUseCase, AnalyzeDrugUseCase...)
  Pharma.AiAssistant.Domain/         ← Core AI types + interfaces (không phụ thuộc gì)
  Pharma.AiAssistant.Infrastructure/ ← Adapters (Ollama HTTP, Postgres, VectorDB, Tools)
```

**Dependency direction:**
```
API → Application → Domain
Infrastructure → Domain (implements interfaces)
```

---

## Phase 1 — Scaffold + Core Types + Simple Chat ✅ Done

**Divergence từ plan gốc:** Architecture thực tế dùng MediatR (Commands/Queries) thay vì plain use-cases. Tất cả endpoint đi qua `IMediator`. Streaming dùng `IStreamMessageHandler` inject trực tiếp vào controller (MediatR không support `IAsyncEnumerable`).

### ✅ 1.1 Domain layer — Core AI types

Files đã có tại `Pharma.AiAssistant.Domain/Ai/`:
- `ContentBlock.cs` — `TextBlock`, `ToolUseBlock`, `ToolResultBlock`
- `LlmMessage.cs` — `record LlmMessage(string Role, IReadOnlyList<ContentBlock> Content)` + static helpers `UserText`, `AssistantText`
- `LlmResponse.cs` — `record LlmResponse(IReadOnlyList<ContentBlock> Content, TokenUsage Usage)`
- `TokenUsage.cs` — `record TokenUsage(int InputTokens, int OutputTokens)`
- `RunResult.cs` — ❌ Chưa tạo (cần khi implement AgentRunner)

### ✅ 1.2 Infrastructure layer — OllamaAdapter

`Pharma.AiAssistant.Infrastructure/Services/OllamaLlmAdapter.cs` đã implement:
- `ChatAsync` — POST `/api/chat` với `Stream: false`
- `StreamAsync` — POST `/api/chat` với `Stream: true`, NDJSON line-by-line, `HttpCompletionOption.ResponseHeadersRead`
- `JsonSerializerOptions { PropertyNameCaseInsensitive = true }` — fix JSON case sensitivity với Ollama

Config trong `launchSettings.json`:
```
OLLAMA_BASE_URL=http://localhost:11434
LLM_MODEL=gemma4:e4b
LLM_MAX_TOKENS=4096
LLM_SYSTEM_PROMPT=...
```

### ❌ 1.3 Domain layer — AgentRunner (THIẾU — cần implement)

`AgentRunner` chưa có. Hiện tại `StreamMessageHandler` gọi thẳng `llmAdapter.StreamAsync()` — bỏ qua tầng này. Đây là **blocker** cho Phase 2 (tool execution) và Phase 5 (multi-agent).

```
Pharma.AiAssistant.Domain/
  Ai/
    AgentRunner.cs    ← core while(true) loop
    RunOptions.cs     ← MaxTurns, CancellationToken
    RunResult.cs      ← record RunResult(string Output, IReadOnlyList<LlmMessage> Messages, ...)
```

```csharp
// AgentRunner.cs — Phase 1: chưa có tools
public async Task<RunResult> RunAsync(IList<LlmMessage> messages, RunOptions options = default)
{
    var conversation = new List<LlmMessage>(messages);
    int turns = 0;

    while (true)
    {
        if (turns++ >= options.MaxTurns) break;

        var response = await _adapter.ChatAsync(conversation, _chatOptions, options.CancellationToken);
        conversation.Add(new LlmMessage("assistant", response.Content));

        var toolUseBlocks = response.Content.OfType<ToolUseBlock>().ToList();
        if (!toolUseBlocks.Any())
        {
            var output = response.Content.OfType<TextBlock>().FirstOrDefault()?.Text ?? "";
            return new RunResult(output, conversation.Skip(messages.Count).ToList(), response.Usage);
        }

        // Phase 2 sẽ implement tool execution
        throw new NotImplementedException("Tool execution — implement in Phase 2");
    }

    return new RunResult("", [], TokenUsage.Zero);
}
```

### ✅ 1.4 Application layer — Chat handlers (MediatR pattern)

Thay vì `SimpleChatUseCase`, dùng MediatR:
- `CreateConversation.cs` — Command, tạo conversation + trả về `conversationId`
- `GetConversations.cs` — Query, phân trang
- `GetConversationDetail.cs` — Query, load messages (fix: `OrderBy(m => m.CreatedAt)`)
- `StreamMessageHandler.cs` — `IStreamMessageHandler`, inject trực tiếp vào controller

Flow của `StreamMessageHandler.StreamAsync`:
1. Save user message → DB
2. Load full history từ DB, sort theo `CreatedAt`
3. `await foreach` trên `llmAdapter.StreamAsync` → yield chunks về controller
4. Sau khi stream xong → save assistant message (full text) → DB
5. Nếu là message đầu tiên → gọi `ChatAsync` để gen title

### ✅ 1.5 API layer

`ConversationController.cs`:
- `POST /conversation` — `CreateConversation` command
- `GET /conversation` — `GetConversations` query (phân trang)
- `GET /conversation/{id}` — `GetConversationDetail` query
- `POST /conversation/{id}/messages/stream` — SSE endpoint, dùng `IStreamMessageHandler`

SSE format: `data: {json_string}\n\n` per chunk, kết thúc bằng `data: [DONE]\n`

---

## Phase 2 — Tools Infrastructure ✅ Done

**Divergence từ plan gốc:** `IToolDefinition` nằm ở **Infrastructure** (không phải Domain). `ToolExecutionResult` nằm ở Application/Models. Tools wired vào DI trong `DependencyInjection.cs`.

Files đã có:
- `Infrastructure/Tools/IToolDefinition.cs` — `Name`, `Description`, `InputSchema`, `ExecuteAsync`
- `Infrastructure/Tools/ToolRegistry.cs` — `Register` / `GetAll` / `GetByName`
- `Application/Services/IToolExecutor.cs` — `Task<ToolExecutionResult> ExecuteAsync(ToolUseBlock, ...)`
- `Infrastructure/Tools/ToolExecutor.cs` — dispatch đến tool đúng tên
- `Infrastructure/Tools/SearchDrugTool.cs` — stub, `throw NotImplementedException("Phase 4")`
- `Infrastructure/Tools/GetDrugInfoTool.cs` — stub, `throw NotImplementedException("Phase 4")`

Tool error hiện tại bị catch ở `AgentRunner` → trả error message về LLM, không crash.

---

## Phase 3 — Chat History ✅ Done

**Divergence từ plan gốc:** Dùng `Conversation` + `Message` entities (không phải `ChatSession` + `ChatMessage` như plan). Không có `IChatSessionRepository` — dùng `IGenericRepository<T>` từ SharedKernel.

### ✅ Domain entities

`Pharma.AiAssistant.Domain/Entities/`:
- `Conversation.cs` — `ConversationId`, `UserId (Ulid)`, `Title?`, audit fields
- `Message.cs` — `MessageId`, `ConversationId`, `Role`, `Content`, `Model?`, audit fields

### ✅ Persistence

`Pharma.AiAssistant.Infrastructure/Persistence/`:
- `WriteDbContext.cs`, `ReadOnlyDbContext.cs`
- `ConversationConfiguration.cs`, `MessageConfiguration.cs`
- Migration: `20260613121810_Initial.cs`

### ✅ Multi-turn history + windowed summarization (diverge đáng kể so với plan gốc)

`Conversation` entity có thêm 2 field (2 migration sau Initial):
- `20260616161951_AddConversationSummary` → `Conversation.Summary (string?)`
- `20260627160926_AlterTableConversationAddSummarizedUpToCount` → `Conversation.SummarizedUpToCount (int)`

`StreamMessageHandler.TrySummarizeAsync`: khi `messageCount >= SummarizedUpToCount + WindowSize(20) + SummarizeThreshold(20)` → gọi `llmAdapter.ChatAsync` để tóm tắt các message cũ, ghi vào `Conversation.Summary` + cập nhật `SummarizedUpToCount`. Khi build `llmMessages` cho mỗi request: prepend `Summary` như 1 synthetic user message + lấy 20 message gần nhất — không load toàn bộ history mỗi lần như plan gốc mô tả.

**Known bug (fixed):** `GetConversationDetail` đã dùng `OrderByDescending(m => m.ConversationId)` → sai thứ tự. Fixed: `OrderBy(m => m.CreatedAt)`.

---

## Phase 4 — RAG: Qdrant Search ✅ Done

**Verify thực tế (2026-07-11):** đã implement đầy đủ, không còn stub. Chi tiết bên dưới phản ánh code thật, không phải kế hoạch nữa.

### Kết quả thực tế

- `IEmbeddingService` **đã** move vào `Pharma.SharedKernel.Application.Interfaces` (đúng như plan) — `SearchDrugTool` và `StreamMessageHandler` import trực tiếp từ đó. Không có `OpenAiEmbeddingService.cs` riêng trong ai-assistant-service — implementation nằm trong SharedKernel package. Infrastructure chỉ đăng ký raw `OpenAI.Embeddings.EmbeddingClient` singleton (`OPENAI_EMBEDDING_MODEL`, mặc định `text-embedding-3-small`).
- `Pharma.AiAssistant.Application/Services/IVectorSearchService.cs` — `ListCollectionsAsync()`, `SearchAsync(vector, collection?, topK, ct)`.
- `Pharma.AiAssistant.Infrastructure/Services/QdrantVectorSearchService.cs` — implement thật, dùng `Qdrant.Client 1.18.1`:
  - `ListCollectionsAsync` → delegate thẳng `QdrantClient.ListCollectionsAsync`
  - `SearchAsync`: nếu có `collection` → chỉ search `drug-class-{collection}`; nếu không → search **tất cả** collections rồi merge + sort theo score, lấy topK
  - Catch `RpcException(NotFound)` riêng từng collection (đúng theo edge-case đã note trong plan gốc) → coi như không có kết quả thay vì crash
- `Infrastructure/Tools/SearchDrugTool.cs` — **implement đầy đủ**: validate `query` → `embeddingService.GenerateEmbeddingAsync` → `vectorSearchService.SearchAsync(vector, drugClass, topK:5)` → format context text + build `Source[]` cho `SourcesEvent` (citation chip trả về UI)
- `Infrastructure/Tools/ListDrugClassesTool.cs` — **implement đầy đủ**: đã đổi sang `IVectorSearchService.ListCollectionsAsync()` + strip prefix `drug-class-` như plan, **không còn** `HttpClient`/`IHttpContextAccessor` gọi document-service
- DI (`Pharma.AiAssistant.Infrastructure/DependencyInjection.cs`): `QdrantClient` singleton (`QDRANT_HOST`, `QDRANT_PORT`, `QDRANT_API_KEY`) → `IVectorSearchService`; cả 2 tool đăng ký singleton vào `ToolRegistry`; `IEmbeddingService` binding đến từ `services.AddSharedInfrastructure()` (SharedKernel) gọi ở cuối method
- `local.props` (không phải `.example`) đã tồn tại ở solution root với `UseLocalSharedKernel=true` — `.csproj` của Application/Infrastructure reference NuGet `Pharma.SharedKernel.Application 1.0.6` / `Pharma.SharedKernel.Infrastructure 1.0.7` khi không dùng local

### Phần dưới đây là nội dung plan gốc — giữ lại để tham khảo lịch sử, đã match với code thật

**Goal:** Tools thật sự query Qdrant, trả về context từ drug PDF đã được index bởi pharma-document-service.

**Context quan trọng:**
- Vector store: **Qdrant** (không phải pgvector) — pharma-document-service đã index sẵn
- Embedding model: **OpenAI** (`text-embedding-3-small`, 1536 dim) — phải dùng cùng model với document-service, không dùng Ollama
- Collection naming: `drug-class-{slug}` — 1 collection per drug class (verify: `BuildCollectionName` trong `DocumentUploadedConsumer.cs`, document-service)
- Payload per chunk (verify đúng key names trong code, không phải giả định): `documentId`, `chunkIndex`, `chunkText`, `fileName`, `drugName`, `drugClassId`
- AI assistant **không** cần ingest data — document-service đã xử lý toàn bộ pipeline PDF → chunk → embed → Qdrant
- Qdrant DI ở document-service truyền cả `apiKey` (env `QDRANT_API_KEY`, đã set trong `infrastructure/infra/.env`) — ai-assistant-service **phải truyền theo**, không thì bị reject request

**Đã verify vs code thực tế (khác so với draft ban đầu của plan này):**
- `GetDrugInfoTool` **không tồn tại** trong code — không có gì để xóa.
- `ListDrugClassesTool` **đã tồn tại**, nhưng hiện gọi HTTP sang document-service (`GET api/documentclass`, forward Authorization header qua `IHttpContextAccessor`). Quyết định: đổi sang query Qdrant trực tiếp (`ListCollectionsAsync`, strip prefix `drug-class-`) — bỏ HTTP client/`IHttpContextAccessor` dependency, tool query cùng nguồn dữ liệu với `search_drug`.
- `QdrantVectorStoreService` bên document-service (dùng cho write path — `EnsureCollectionExistsAsync`, `UpsertBatchAsync`) **không có method search**, và nằm ở solution riêng — ai-assistant-service phải tự viết read-path riêng bằng `Qdrant.Client`, không tái sử dụng được service class đó.

### 4.1 Tool redesign

**Update** `SearchDrugTool` schema:

```csharp
public string Name => "search_drug";
public string Description => "Search pharmaceutical knowledge base for drug information. Use specific queries for best results (e.g. 'Warfarin dosage elderly renal failure').";
public JsonObject InputSchema => new()
{
    ["type"] = "object",
    ["properties"] = new JsonObject
    {
        ["query"] = new JsonObject
        {
            ["type"] = "string",
            ["description"] = "Semantic search query — be specific (drug name + topic)"
        },
        ["drug_class"] = new JsonObject
        {
            ["type"] = "string",
            ["description"] = "Optional drug class slug to narrow search (from list_drug_classes). Omit to search all classes."
        }
    },
    ["required"] = new JsonArray { "query" }
};
```

**Update** `ListDrugClassesTool` (đã tồn tại, đổi implementation): bỏ `HttpClient` + `IHttpContextAccessor` (gọi document-service), thay bằng inject `IVectorSearchService`:

```csharp
public string Name => "list_drug_classes";
public string Description => "List all available drug classes in the knowledge base. Call this first when you don't know which drug class to search in.";
public JsonObject InputSchema => new() { ["type"] = "object", ["properties"] = new JsonObject() };
// ExecuteAsync → vectorSearchService.ListCollectionsAsync() → strip "drug-class-" prefix
```

### 4.2 SharedKernel — IEmbeddingService (prerequisite)

`IEmbeddingService` hiện đang nằm ở `Pharma.Document.Application/Services/` — cần **move lên SharedKernel** để ai-assistant-service dùng chung, tránh define lại interface trùng.

```
Pharma.SharedKernel.Application/
  Interfaces/
    IEmbeddingService.cs      ← move từ document-service (giữ nguyên contract)
```

```csharp
// Giữ nguyên contract — chỉ đổi namespace
public interface IEmbeddingService
{
    Task<float[]> GenerateEmbeddingAsync(string text, CancellationToken cancellationToken);
    Task<IReadOnlyList<float[]>> GenerateEmbeddingsAsync(IEnumerable<string> texts, CancellationToken cancellationToken = default);
}
```

Sau khi move:
- `Pharma.Document.Application` → update using sang `Pharma.SharedKernel.Application.Interfaces`
- `Pharma.Document.Infrastructure/Services/EmbeddingService.cs` → implement từ SharedKernel interface

### 4.3 Application layer — interfaces

```
Pharma.AiAssistant.Application/
  Services/
    IVectorSearchService.cs   ← Task<IReadOnlyList<SearchResult>> SearchAsync(float[] vector, string? collection, int topK, ...)
                                 Task<IReadOnlyList<string>> ListCollectionsAsync(...)  ← dùng chung bởi ListDrugClassesTool
```

> `IEmbeddingService` lấy từ `Pharma.SharedKernel.Application.Interfaces` — không tạo mới.

```csharp
public record SearchResult(string ChunkText, string DrugName, string FileName, float Score);
```

### 4.4 Infrastructure layer

```
Pharma.AiAssistant.Infrastructure/
  Services/
    OpenAiEmbeddingService.cs     ← implements SharedKernel.IEmbeddingService, dùng OpenAI.Embeddings SDK
    QdrantVectorSearchService.cs  ← implements IVectorSearchService, dùng Qdrant.Client SDK
  Tools/
    SearchDrugTool.cs             ← inject IEmbeddingService + IVectorSearchService, implement thật
    ListDrugClassesTool.cs        ← đổi inject: bỏ HttpClient/IHttpContextAccessor, dùng IVectorSearchService.ListCollectionsAsync
```

```csharp
// QdrantVectorSearchService.cs
public async Task<IReadOnlyList<SearchResult>> SearchAsync(float[] vector, string? collection, int topK, CancellationToken ct)
{
    var collections = collection is not null
        ? [$"drug-class-{collection}"]
        : await _client.ListCollectionsAsync(ct); // trả về IReadOnlyList<string>, không phải object có .Name

    var results = new List<SearchResult>();
    foreach (var col in collections)
    {
        var hits = await _client.SearchAsync(col, vector, limit: (ulong)topK, cancellationToken: ct);
        results.AddRange(hits.Select(h => new SearchResult(
            ChunkText: h.Payload["chunkText"].StringValue,
            DrugName: h.Payload["drugName"].StringValue,
            FileName: h.Payload["fileName"].StringValue,
            Score: h.Score)));
    }

    return results.OrderByDescending(r => r.Score).Take(topK).ToList();
}
```

Edge case cần xử lý: nếu `drug_class` do LLM truyền vào không map tới collection nào tồn tại, `SearchAsync` trên 1 collection sẽ throw (collection not found). Bọc try/catch quanh từng collection trong loop — coi như "no results" thay vì crash cả request, tương tự cách `AgentRunner` catch lỗi tool hiện tại.

```csharp
// SearchDrugTool.cs — implement thật
public async Task<ToolExecutionResult> ExecuteAsync(string toolUseId, JsonObject input, CancellationToken ct)
{
    var query = input["query"]!.GetValue<string>();
    var drugClass = input["drug_class"]?.GetValue<string>();

    var vector = await _embeddingService.GenerateEmbeddingAsync(query, ct);
    var results = await _vectorSearchService.SearchAsync(vector, drugClass, topK: 5, ct);

    if (results.Count == 0)
        return new ToolExecutionResult(toolUseId, "No relevant drug information found.", IsError: false);

    var context = string.Join("\n\n", results.Select(r =>
        $"[{r.DrugName} — {r.FileName}] (score: {r.Score:F2})\n{r.ChunkText}"));

    return new ToolExecutionResult(toolUseId, context, IsError: false);
}
```

### 4.5 DI wiring

```csharp
// DependencyInjection.cs — thêm region "RAG Configuration"
var openAiApiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY")!;
services.AddSingleton(_ => new EmbeddingClient("text-embedding-3-small", openAiApiKey));
services.AddSingleton<IEmbeddingService, OpenAiEmbeddingService>();

var qdrantHost = Environment.GetEnvironmentVariable("QDRANT_HOST") ?? "localhost";
var qdrantPort = int.TryParse(Environment.GetEnvironmentVariable("QDRANT_PORT"), out var qp) ? qp : 6334;
services.AddSingleton(_ => new QdrantClient(
    host: qdrantHost,
    port: qdrantPort,
    apiKey: Environment.GetEnvironmentVariable("QDRANT_API_KEY"))); // bắt buộc, giống document-service
services.AddSingleton<IVectorSearchService, QdrantVectorSearchService>();

// Tools Configuration — bỏ AddHttpClient<ListDrugClassesTool>(...) hiện có (không còn gọi HTTP nữa)
toolRegistry.Register(new SearchDrugTool(
    sp.GetRequiredService<IEmbeddingService>(),
    sp.GetRequiredService<IVectorSearchService>()));
toolRegistry.Register(new ListDrugClassesTool(
    sp.GetRequiredService<IVectorSearchService>(),
    sp.GetRequiredService<ILogger<ListDrugClassesTool>>()));
```

### 4.6 NuGet packages cần thêm

```
Qdrant.Client
OpenAI  (đã có ở document-service, thêm vào ai-assistant-service)
```

### 4.7 Verify Phase 4

```
1. Upload 1 PDF lên document-service → đợi status = Completed
2. POST /conversation/{id}/messages/stream { message: "Warfarin tương tác với Amiodarone?" }
   → Log thấy LLM gọi list_drug_classes → thấy collections
   → LLM gọi search_drug("Warfarin Amiodarone interaction", drug_class="anticoagulants")
   → Tool query Qdrant → trả về chunks
   → LLM trả lời có context từ PDF
```

---

## Phase 5 — Self-Correction (Agentic RAG failure handling) ❌ Chưa bắt đầu

**Nguồn cảm hứng:** Microsoft "Agentic RAG" self-correction pattern (`aka.ms/ai-agents-beginners`) — grade → rewrite → retry → fallback → escalate.

**Đã scan code thật (2026-07-12)** để biết baseline trước khi thiết kế phase này — xem `AgentRunner.cs`, `SearchDrugTool.cs`, `QdrantVectorSearchService.cs`. Kết luận: hiện tại đã có "xương" (multi-turn tool loop, bounded bởi `MaxTurns`) nhưng **không có "não"** — mọi hành vi self-correction hiện tại chỉ là chỉ dẫn trong system prompt, không phải code:

| Failure mode (theo sơ đồ Microsoft) | Trạng thái hiện tại | Bằng chứng |
|---|---|---|
| Grade retrieved docs | ❌ Không có | `SearchDrugTool.cs` trả raw top-K, không chấm điểm/lọc |
| Rewrite Query | ❌ Không có | Chỉ có gợi ý trong system prompt, không có component rewrite |
| New Search / retry có điều kiện | ⚠️ Một phần | Có turn-budget loop nhưng không "loop-until-good-enough"; hết turn → log warning, trả rỗng |
| Try Alternative Source | ⚠️ Một phần | Chỉ prompt-level: bảo LLM tự trả lời bằng kiến thức chung nếu tool rỗng/lỗi |
| Request Human Help | ❌ Không có | Không có confidence scoring/flag/escalation path |
| Tracing/observability | ❌ Tắt | Aspire `ServiceDefaults` (OpenTelemetry) bị comment `#if DEBUG` trong `Program.cs` |
| Malformed query validation | ⚠️ Một phần | Chỉ check null/whitespace |

**Goal:** Biến `SearchDrugTool` + `AgentRunner` từ "retrieve-once-and-hope" thành vòng lặp tự sửa lỗi thật sự, có thể quan sát được (traceable), với escalation rõ ràng khi vượt giới hạn tự sửa.

### 5.1 Domain layer — types mới

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

### 5.2 Application layer — interfaces

```
Pharma.AiAssistant.Application/
  Services/
    IRetrievalGrader.cs   ← Task<RetrievalGrade> GradeAsync(string query, IReadOnlyList<SearchResult> results, ct)
    IQueryRewriter.cs     ← Task<string> RewriteAsync(string originalQuery, string failureReason, ct)
```

Cả 2 nên là **1 LLM call rẻ** (model nhỏ hoặc cùng model hiện tại với prompt ngắn, temperature thấp) — không tái sử dụng `AgentRunner` (tránh vòng lặp lồng vòng lặp).

```csharp
public record GradeRequest(string Query, IReadOnlyList<SearchResult> Results);
```

### 5.3 Infrastructure layer

```
Pharma.AiAssistant.Infrastructure/
  Services/
    RelevanceGraderService.cs   ← implements IRetrievalGrader
    QueryRewriterService.cs     ← implements IQueryRewriter
  Tools/
    SearchDrugTool.cs           ← sửa: thêm self-correction loop bên trong ExecuteAsync
```

**`RelevanceGraderService`** — 2 chiến lược, chọn 1 để bắt đầu đơn giản trước:
- **Threshold-based (rẻ, nên làm trước):** nếu tất cả `SearchResult.Score` dưới ngưỡng (vd. `0.5`, config qua env `RAG_RELEVANCE_THRESHOLD`) → `Irrelevant`. Không tốn thêm LLM call.
- **LLM-as-judge (chính xác hơn, làm sau nếu threshold không đủ):** prompt ngắn hỏi model "các đoạn trích này có trả lời được câu hỏi không?" → parse `Relevant`/`Irrelevant`/`Insufficient`.

**`SearchDrugTool.ExecuteAsync`** — sửa flow thành:

```csharp
public async Task<ToolExecutionResult> ExecuteAsync(string toolUseId, JsonObject input, CancellationToken ct)
{
    var query = input["query"]!.GetValue<string>();
    var drugClass = input["drug_class"]?.GetValue<string>();
    var attempted = new List<string> { query };

    for (int attempt = 0; attempt <= MaxRewriteAttempts; attempt++)
    {
        var vector  = await _embeddingService.GenerateEmbeddingAsync(query, ct);
        var results = await _vectorSearchService.SearchAsync(vector, drugClass, topK: 5, ct);

        var grade = await _grader.GradeAsync(query, results, ct);
        if (grade == RetrievalGrade.Relevant)
            return BuildSuccessResult(toolUseId, results);

        if (attempt == MaxRewriteAttempts) break; // hết lượt tự sửa

        query = await _rewriter.RewriteAsync(query, grade.ToString(), ct);
        attempted.Add(query);
    }

    // Fallback: không phải throw — trả kết quả rỗng có đánh dấu, để AgentRunner/LLM biết mà escalate
    return new ToolExecutionResult(toolUseId,
        $"No relevant drug information found after {attempted.Count} query attempts: {string.Join(" → ", attempted)}. " +
        "Answer must state this explicitly and suggest escalation if this is a critical safety question.",
        IsError: false);
}
```

`MaxRewriteAttempts` mặc định `2` (config qua env `RAG_MAX_REWRITE_ATTEMPTS`) — tổng cộng tối đa 3 lần search cho 1 tool call, tránh vòng lặp vô hạn/tốn cost.

### 5.4 Escalation — AgentRunner

Sau khi tool trả kết quả "no relevant data" (đánh dấu ở 5.3), **không** để LLM tự quyết định im lặng — `AgentRunner` cần phát hiện pattern này và emit `EscalationEvent` song song với câu trả lời cuối, để UI luôn hiển thị banner nhất quán thay vì phụ thuộc LLM có "nhớ" nói ra hay không:

```csharp
// AgentRunner.cs — sau khi collect ToolResultBlock[]
var hasUnresolvedRetrieval = toolResults.Any(r => r.Content.Contains("No relevant drug information found after"));
if (hasUnresolvedRetrieval && turns == options.MaxTurns) // hoặc: câu hỏi có vẻ critical (dosage/interaction) — heuristic đơn giản trước
    yield return new EscalationEvent(
        Reason: "Retrieval exhausted rewrite attempts without finding relevant data",
        AttemptedQueries: /* lấy từ tool result */);
```

_Lưu ý: heuristic "câu hỏi critical" ban đầu có thể chỉ là keyword match (`dosage`, `interaction`, `contraindication`, `liều`, `tương tác`) — không cần ML, tinh chỉnh sau khi có dữ liệu thật._

### 5.5 Observability — bật lại tracing

- Bỏ guard `#if DEBUG` quanh `builder.AddServiceDefaults()` / `app.MapDefaultEndpoints()` trong `Program.cs` (Aspire `ServiceDefaults` đã wire sẵn OpenTelemetry, chỉ đang bị tắt ở Release).
- Thêm `ActivitySource` riêng (`"Pharma.AiAssistant.Rag"`) bọc quanh: retrieval, grading, rewrite, mỗi tool call — set tag `rag.grade`, `rag.attempt`, `rag.rewritten_query` để trace hiển thị đúng bước nào tự sửa lỗi, giống ý "Diagnostic Tools" trong bài Microsoft.
- Không cần Azure AI Tracing cụ thể ngay — `ActivitySource` chuẩn OTel là đủ, export sang bất kỳ backend nào sau (Aspire dashboard lúc dev, Application Insights lúc prod).

### 5.6 NuGet packages cần thêm

```
(không cần thêm gì mới — ActivitySource nằm trong System.Diagnostics.DiagnosticSource, đã có sẵn qua ASP.NET Core)
```

### 5.7 Verify Phase 5

```
1. Query rõ ràng có trong KB (vd. "Warfarin dosage elderly")
   → grade = Relevant ngay lần đầu, không rewrite, trace chỉ có 1 attempt

2. Query mơ hồ/không có trong KB (vd. "thuốc gì tốt cho tim")
   → grade = Irrelevant lần 1 → rewriter sinh câu cụ thể hơn → search lại
   → nếu vẫn Irrelevant sau MaxRewriteAttempts → tool trả "no relevant data" có đánh dấu
   → nếu câu hỏi match keyword critical → EscalationEvent xuất hiện, UI hiển thị banner

3. Check Aspire dashboard / OTel export → thấy đủ span: retrieval → grade → rewrite → retry,
   tag rõ ràng để debug được vì sao 1 câu trả lời bị escalate
```

---

## Phase 6 — Multi-Agent (TaskQueue + AgentPool + SharedMemory) ❌ Chưa bắt đầu

**Goal:** Câu hỏi phức tạp → nhiều agents chạy song song, chia sẻ kết quả.

### 5.1 Domain layer

```
Pharma.AiAssistant.Domain/
  Ai/
    MultiAgent/
      AgentTask.cs          ← Id, Title, Prompt, Assignee, DependsOn[], Status, Result
      TaskStatus.cs         ← enum: Pending, Blocked, InProgress, Completed, Failed, Skipped
      TaskQueue.cs          ← dependency graph + auto-unblock + cascade failure
      AgentPool.cs          ← Dictionary<Agent> + SemaphoreSlim (per-agent + pool-wide)
      SharedMemory.cs       ← Dictionary namespace KV + GetSummary()
```

```csharp
// TaskQueue.cs — event-driven
public event Action<AgentTask>? TaskReady;
public event Action<AgentTask>? TaskCompleted;
public event Action<AgentTask>? TaskFailed;
public event Action? AllCompleted;

public void Complete(string taskId, string result)
{
    Update(taskId, TaskStatus.Completed, result);
    UnblockDependents(taskId);
    if (IsComplete()) AllCompleted?.Invoke();
}

public void Fail(string taskId, string error)
{
    Update(taskId, TaskStatus.Failed, error);
    CascadeFailure(taskId);
}
```

```csharp
// AgentPool.cs — double semaphore
public async Task<RunResult> RunAsync(string agentName, string prompt)
{
    var agentLock = _agentLocks[agentName];

    await agentLock.WaitAsync();
    try
    {
        await _poolSemaphore.WaitAsync();
        try   { return await _agents[agentName].RunAsync(prompt); }
        finally { _poolSemaphore.Release(); }
    }
    finally { agentLock.Release(); }
}
```

### 5.2 Application layer — MultiAgentAnalysisUseCase

```csharp
public async Task<string> ExecuteAsync(IReadOnlyList<AgentTask> tasks)
{
    var memory = new SharedMemory();
    var queue  = new TaskQueue();

    queue.TaskReady += async task => {
        var context = await memory.GetSummaryAsync(task.DependsOn);
        var prompt  = $"{task.Prompt}\n\nContext:\n{context}";
        var result  = await _pool.RunAsync(task.Assignee, prompt);

        await memory.WriteAsync(task.Assignee, $"task:{task.Id}:result", result.Output);
        queue.Complete(task.Id, result.Output);
    };

    queue.AddBatch(tasks);
    await queue.WaitAllAsync();

    return await memory.GetSummaryAsync();
}
```

**Verify Phase 5:**
```
Task A: Tìm Warfarin info (drug-retriever)
Task B: Tìm Amiodarone info (drug-retriever)
Task C: Phân tích tương tác (dependsOn: A, B) (analyst)
→ A và B chạy song song
→ C tự unblock khi A+B xong, có context từ SharedMemory
```

---

## Phase 7 — Coordinator + Orchestrator ❌ Chưa bắt đầu

**Goal:** User hỏi ngôn ngữ tự nhiên → Coordinator tự sinh task list → chạy.

### 6.1 Domain layer

```
Pharma.AiAssistant.Domain/
  Ai/
    Orchestrator/
      OrchestratorService.cs    ← RunTeamAsync(goal, availableAgents)
      CoordinatorAgent.cs       ← gọi LLM với outputSchema = TaskListSchema
      TaskListSchema.cs         ← schema để ép LLM trả JSON đúng format
```

```csharp
// OrchestratorService.cs
public async Task<string> RunTeamAsync(string goal)
{
    // [1] Coordinator LLM → sinh task list
    var coordinator = _agentFactory.CreateCoordinator(_availableAgents);
    var plan = await coordinator.RunAsync(goal);
    var tasks = JsonSerializer.Deserialize<List<AgentTask>>(plan.Output)!;

    // [2] Chạy qua TaskQueue + AgentPool (Phase 5)
    var result = await _multiAgentUseCase.ExecuteAsync(tasks);

    // [3] Synthesize final answer
    var finalPrompt = $"Goal: {goal}\n\nResults:\n{result}\n\nSynthesize a final answer.";
    var finalResult = await coordinator.RunAsync(finalPrompt);

    return finalResult.Output;
}
```

### 6.2 API endpoint

```
POST /api/analyze
{ "question": "Bệnh nhân 70 tuổi, suy thận độ 2, đang dùng Warfarin. Thêm Amiodarone có an toàn không?" }
→ Coordinator sinh tasks
→ Agents chạy song song
→ Final synthesized answer
```

**Verify Phase 6:**
```
→ Log thấy Coordinator sinh ra task list JSON
→ TaskQueue chạy đúng thứ tự
→ Final answer có quality cao hơn Phase 3 (single agent)
```

---

## Thứ tự NuGet packages theo phase

| Phase | Package |
|-------|---------|
| 1–3 (done) | `System.Net.Http.Json`, `MediatR`, `Npgsql.EntityFrameworkCore.PostgreSQL` |
| 4 | `Qdrant.Client`, `OpenAI` |
| 5 | _(không cần thêm — `System.Diagnostics.DiagnosticSource` đã có sẵn)_ |
| All | `Microsoft.AspNetCore.OpenApi`, `Scalar.AspNetCore` |

---

## Bước tiếp theo

_Cập nhật 2026-07-12: thêm Phase 5 (Self-Correction) vào giữa Phase 4 và Multi-Agent, dựa trên scan code thật + bài "Agentic RAG" của Microsoft. Việc còn lại:_

1. **[Optional/vá lỗi] Rotate `OPENAI_API_KEY`** đang lộ trong `launchSettings.json` — chuyển sang user-secrets hoặc env ngoài, kiểm tra file này có bị commit lên git không.
2. **Phase 5 — Self-Correction:** chưa có file nào. Nên làm **trước** Multi-agent vì đây là nền tảng chất lượng RAG mà mỗi agent trong pool (Phase 6) sẽ dùng lại — làm sau Phase 6 thì phải sửa lại nhiều tool ở nhiều agent cùng lúc. Thứ tự đề xuất trong phase: 5.1 → 5.3 (threshold-based grader trước, bỏ qua LLM-as-judge ban đầu) → 5.4 (escalation) → 5.5 (tracing) — LLM-as-judge grader để sau khi có dữ liệu thật đánh giá threshold có đủ tốt không.
3. **Phase 6 — Multi-agent:** chưa có file nào (`TaskQueue`, `AgentPool`, `SharedMemory` — 0 kết quả grep toàn solution). Thiết kế ở phần Phase 6 bên dưới vẫn là plan (chưa implement), review lại trước khi bắt tay vì đã cách xa thời điểm viết ban đầu — cân nhắc tận dụng lại `AgentRunner`/`IAgentRunner` hiện có làm nền cho từng agent trong pool thay vì viết mới.
4. **Phase 7 — Coordinator + Orchestrator:** chưa có file nào, phụ thuộc Phase 6 xong trước.
