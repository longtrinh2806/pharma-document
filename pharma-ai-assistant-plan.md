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
| 2 | Tools infrastructure | ❌ Chưa bắt đầu |
| 3 | Chat history (multi-turn) | ✅ Done (diverged — xem chi tiết) |
| 4 | RAG + pgvector | ❌ Chưa bắt đầu |
| 5 | Multi-agent (TaskQueue + AgentPool) | ❌ Chưa bắt đầu |
| 6 | Coordinator + Orchestrator | ❌ Chưa bắt đầu |

**`AgentRunner` chưa implement** — đây là nền của Phase 2–6. Phải làm trước khi tiếp tục.

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

## Phase 2 — Tools Infrastructure ❌ Chưa bắt đầu

**Dependency:** Cần implement `AgentRunner` (Phase 1.3) trước.

**Goal:** LLM có thể "gọi tool", AgentRunner execute tool thật (dù tool chưa có logic RAG).

### 2.1 Domain layer — Tool contracts

```
Pharma.AiAssistant.Domain/
  Ai/
    Tools/
      IToolDefinition.cs    ← Name, Description, InputSchema (JsonObject)
      ToolResult.cs         ← record ToolResult(string ToolUseId, string Output, bool IsError)
      ToolRegistry.cs       ← Register / GetAll / GetByName
  Interfaces/
    IToolExecutor.cs        ← Task<ToolResult> ExecuteAsync(ToolUseBlock block, ...)
```

### 2.2 Infrastructure layer — Tool implementations (stubs)

```
Pharma.AiAssistant.Infrastructure/
  Tools/
    SearchDrugTool.cs       ← implements IToolDefinition, ExecuteAsync → throw NotImplementedException (Phase 4)
    GetDrugInfoTool.cs      ← implements IToolDefinition, ExecuteAsync → throw NotImplementedException (Phase 4)
    ToolExecutor.cs         ← implements IToolExecutor, dispatch đến đúng tool
```

```csharp
// SearchDrugTool.cs
public sealed class SearchDrugTool : IToolDefinition
{
    public string Name => "search_drug";
    public string Description => "Tìm kiếm thông tin thuốc từ cơ sở dữ liệu dược phẩm";
    public JsonObject InputSchema => ...; // { query: string }

    public Task<ToolResult> ExecuteAsync(JsonObject input, CancellationToken ct)
        => throw new NotImplementedException("Vector search — implement in Phase 4");
}
```

### 2.3 AgentRunner — bổ sung tool execution

```csharp
// Thay thế NotImplementedException ở Phase 1:
var executions = await Task.WhenAll(
    toolUseBlocks.Select(b => _toolExecutor.ExecuteAsync(b, options.CancellationToken))
);

var resultBlocks = executions
    .Select(e => (ContentBlock)new ToolResultBlock(e.ToolUseId, e.Output, e.IsError))
    .ToList();

conversation.Add(new LlmMessage("user", resultBlocks));
// → loop lại, LLM đọc tool results
```

**Verify Phase 2:**
```
Hỏi câu khiến LLM muốn gọi search_drug
→ Log thấy ToolUseBlock được nhận
→ NotImplementedException throw (expected ở phase này)
→ Chứng tỏ flow tool call đúng
```

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

## Phase 4 — RAG: Vector Search ❌ Chưa bắt đầu

**Goal:** SearchDrugTool thật sự query pgvector, trả về context liên quan.

### 4.1 Domain layer — Drug document

```
Pharma.AiAssistant.Domain/
  DrugKnowledge/
    DrugDocument.cs          ← Id, DrugName, Content, Embedding (float[])
  Interfaces/
    IDrugKnowledgeRepository.cs   ← SearchSimilarAsync(float[] embedding, int topK)
    IEmbeddingService.cs          ← Task<float[]> EmbedAsync(string text)
```

### 4.2 Infrastructure layer

```
Pharma.AiAssistant.Infrastructure/
  Ai/
    Ollama/
      OllamaEmbeddingService.cs   ← implements IEmbeddingService (POST /api/embeddings)
  Persistence/
    DrugKnowledgeRepository.cs   ← pgvector similarity search
    Migrations/                   ← thêm vector column
  Tools/
    SearchDrugTool.cs             ← implement thật (thay NotImplementedException)
    GetDrugInfoTool.cs            ← implement thật
```

```csharp
// SearchDrugTool.cs — implement thật
public async Task<ToolResult> ExecuteAsync(JsonObject input, CancellationToken ct)
{
    var query = input["query"]!.GetValue<string>();
    var embedding = await _embeddingService.EmbedAsync(query, ct);
    var docs = await _drugRepo.SearchSimilarAsync(embedding, topK: 5, ct);
    var context = string.Join("\n\n", docs.Select(d => $"[{d.DrugName}]\n{d.Content}"));
    return new ToolResult(input["_tool_use_id"]!.GetValue<string>(), context);
}
```

### 4.3 Data ingestion script

```
Pharma.AiAssistant.Infrastructure/
  Seeding/
    DrugKnowledgeSeedService.cs   ← đọc drug data → embed → lưu vào pgvector
```

**Verify Phase 4:**
```
POST /conversation/{id}/messages/stream { message: "Warfarin tương tác với thuốc nào?" }
→ LLM gọi search_drug("Warfarin interactions")
→ Tool query pgvector → trả về docs
→ LLM trả lời có context từ drug DB
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
| 1 (done) | `System.Net.Http.Json`, `MediatR`, `Npgsql.EntityFrameworkCore.PostgreSQL` |
| 4 | `Pgvector.EntityFrameworkCore` |
| All | `Microsoft.AspNetCore.OpenApi`, `Scalar.AspNetCore` |

---

## Bước tiếp theo

1. **Implement `AgentRunner` + `RunResult` + `RunOptions`** (Phase 1.3 còn thiếu) — prerequisite cho mọi thứ tiếp theo
2. Wire `AgentRunner` vào `StreamMessageHandler` để thay thế direct `llmAdapter.StreamAsync()` call
3. Phase 2: Tools stubs (`IToolDefinition`, `ToolRegistry`, `IToolExecutor`, `SearchDrugTool`)
4. Phase 4: RAG + pgvector
