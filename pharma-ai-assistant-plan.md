# pharma-ai-assistant-service — Implementation Plan

> **Stack:** .NET 10, Clean Architecture (API / Application / Domain / Infrastructure)  
> **Pattern:** Re-implement Anthropic open-multi-agent framework in C#, từng layer một  
> **Rule:** Phase sau mới implement → Phase trước `throw new NotImplementedException()`
> **LLM runtime:** Ollama (local) thay vì Anthropic API — `ILlmAdapter` là abstraction layer, có thể swap sau.

---

## Trạng thái tổng quan

| Phase | Mô tả | Trạng thái |
|-------|-------|-----------|
| 1 | Core types + ILlmAdapter + OllamaAdapter + basic chat | ✅ Done (diverged — xem chi tiết) |
| 2 | Tools infrastructure + AgentRunner | ✅ Done (diverged — xem chi tiết) |
| 3 | Chat history (multi-turn) + sliding window summary | ✅ Done (diverged — xem chi tiết) |
| 4 | RAG + Qdrant search | ⏳ Next |
| 5 | Multi-agent (TaskQueue + AgentPool) | ❌ Chưa bắt đầu |
| 6 | Coordinator + Orchestrator | ❌ Chưa bắt đầu |

---

## AgentRunner — ✅ Done

Flow hiện tại (`AgentRunner.cs` tại Infrastructure/Services):
```
StreamMessageHandler → agentRunner.StreamAsync()
                           ↓
                    while (có tool call):
                        llmAdapter.StreamAsync()    ← stream từng turn, collect ToolUseBlock
                        toolExecutor.ExecuteAsync() ← parallel Task.WhenAll
                        yield ToolResultEvent → feed back vào conversation
                           ↓
                    turn cuối (không có tool call):
                        yield TextChunk + DoneEvent
```

Divergence thực tế so với plan gốc:
- Stream **từng turn** (không phải ChatAsync + StreamAsync cuối) → user thấy thinking realtime
- `AgentRunner` nằm ở **Infrastructure** (không phải Domain) vì depend vào `ILlmAdapter`
- `IAgentRunner` chỉ có `StreamAsync` (không có `RunAsync`) — đủ cho use case hiện tại
- Tool execution error bị **catch**, trả về error message thay vì crash → agent tiếp tục

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

### ✅ Multi-turn history

`StreamMessageHandler` build history từ DB trước mỗi request:
```csharp
var histories = await messageRepository.FindAsync(
    m => m.ConversationId == conversationId, cancellationToken);

var llmMessages = histories
    .OrderBy(m => m.CreatedAt)
    .Select(m => m.Role == "user"
        ? LlmMessage.UserText(m.Content)
        : LlmMessage.AssistantText(m.Content))
    .ToList();
```

**Known bug (fixed):** `GetConversationDetail` đã dùng `OrderByDescending(m => m.ConversationId)` → sai thứ tự. Fixed: `OrderBy(m => m.CreatedAt)`.

---

## Phase 4 — RAG: Qdrant Search ⏳ Next

**Goal:** Tools thật sự query Qdrant, trả về context từ drug PDF đã được index bởi pharma-document-service.

**Context quan trọng:**
- Vector store: **Qdrant** (không phải pgvector) — pharma-document-service đã index sẵn
- Embedding model: **OpenAI** (`text-embedding-3-small`, 1536 dim) — phải dùng cùng model với document-service, không dùng Ollama
- Collection naming: `drug-class-{slug}` — 1 collection per drug class
- Payload per chunk: `chunkText`, `drugName`, `drugClassId`, `documentId`, `fileName`, `chunkIndex`
- AI assistant **không** cần ingest data — document-service đã xử lý toàn bộ pipeline PDF → chunk → embed → Qdrant

### 4.1 Tool redesign

**Xóa** `GetDrugInfoTool` — duplicate logic, `info_type` enum không map được vào cách Qdrant lưu trữ.

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

**Thêm** `ListDrugClassesTool`:

```csharp
public string Name => "list_drug_classes";
public string Description => "List all available drug classes in the knowledge base. Call this first when you don't know which drug class to search in.";
public JsonObject InputSchema => new() { ["type"] = "object", ["properties"] = new JsonObject() };
// ExecuteAsync → qdrantClient.ListCollectionsAsync() → strip "drug-class-" prefix
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
                                 Task<IReadOnlyList<string>> ListCollectionsAsync(...)
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
    ListDrugClassesTool.cs        ← inject IVectorSearchService, gọi ListCollectionsAsync
    GetDrugInfoTool.cs            ← XÓA
```

```csharp
// QdrantVectorSearchService.cs
public async Task<IReadOnlyList<SearchResult>> SearchAsync(float[] vector, string? collection, int topK, CancellationToken ct)
{
    var collections = collection is not null
        ? [$"drug-class-{collection}"]
        : (await _client.ListCollectionsAsync(ct)).Select(c => c.Name).ToList();

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
// DependencyInjection.cs — thêm vào LLM Configuration block
var openAiApiKey = Environment.GetEnvironmentVariable("OPENAI_API_KEY")!;
services.AddSingleton(new EmbeddingClient("text-embedding-3-small", openAiApiKey));
services.AddSingleton<IEmbeddingService, OpenAiEmbeddingService>();

var qdrantHost = Environment.GetEnvironmentVariable("QDRANT_HOST") ?? "localhost";
var qdrantPort = int.TryParse(Environment.GetEnvironmentVariable("QDRANT_PORT"), out var qp) ? qp : 6334;
services.AddSingleton(new QdrantClient(qdrantHost, qdrantPort));
services.AddSingleton<IVectorSearchService, QdrantVectorSearchService>();

// Tools — thay GetDrugInfoTool bằng ListDrugClassesTool
toolRegistry.Register(new SearchDrugTool(...));   // inject services qua constructor
toolRegistry.Register(new ListDrugClassesTool(...));
// XÓA: toolRegistry.Register(new GetDrugInfoTool());
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

## Phase 5 — Multi-Agent (TaskQueue + AgentPool + SharedMemory) ❌ Chưa bắt đầu

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

## Phase 6 — Coordinator + Orchestrator ❌ Chưa bắt đầu

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
| All | `Microsoft.AspNetCore.OpenApi`, `Scalar.AspNetCore` |

---

## Bước tiếp theo

1. **Phase 4 — RAG:**
   - Move `IEmbeddingService` từ `Pharma.Document.Application/Services/` → `Pharma.SharedKernel.Application/Interfaces/`, update usings ở document-service
   - Xóa `GetDrugInfoTool`
   - Update `SearchDrugTool` schema (thêm `drug_class` optional param)
   - Thêm `ListDrugClassesTool`
   - Thêm `IVectorSearchService` + `QdrantVectorSearchService` + `OpenAiEmbeddingService` (implement SharedKernel interface)
   - Implement `SearchDrugTool.ExecuteAsync` và `ListDrugClassesTool.ExecuteAsync`
   - Wire DI: OpenAI embedding client + QdrantClient + env vars
2. Phase 5: Multi-agent (TaskQueue + AgentPool + SharedMemory)
3. Phase 6: Coordinator + Orchestrator
