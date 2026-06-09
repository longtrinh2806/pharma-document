# pharma-ai-assistant-service — Implementation Plan

> **Stack:** .NET 10, Clean Architecture (API / Application / Domain / Infrastructure)  
> **Pattern:** Re-implement Anthropic open-multi-agent framework in C#, từng layer một  
> **Rule:** Phase sau mới implement → Phase trước `throw new NotImplementedException()`

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
  Pharma.AiAssistant.Infrastructure/ ← Adapters (Anthropic HTTP, Postgres, VectorDB, Tools)
```

**Dependency direction:**
```
API → Application → Domain
Infrastructure → Domain (implements interfaces)
```

---

## Phase 1 — Scaffold + Core Types + Simple Chat

**Goal:** Gọi được Anthropic API, nhận được response. Chưa có tools, chưa có history.

### 1.1 Domain layer — Core AI types

```
Pharma.AiAssistant.Domain/
  Ai/
    ContentBlock.cs      ← TextBlock, ToolUseBlock, ToolResultBlock
    LlmMessage.cs        ← record LlmMessage(string Role, IReadOnlyList<ContentBlock> Content)
    LlmResponse.cs       ← record LlmResponse(IReadOnlyList<ContentBlock> Content, TokenUsage Usage)
    TokenUsage.cs        ← record TokenUsage(int InputTokens, int OutputTokens)
    RunResult.cs         ← record RunResult(string Output, IReadOnlyList<LlmMessage> Messages, ...)
  Interfaces/
    ILlmAdapter.cs       ← Task<LlmResponse> ChatAsync(IList<LlmMessage> messages, ...)
```

```csharp
// ContentBlock.cs
public abstract record ContentBlock(string Type);
public sealed record TextBlock(string Text) : ContentBlock("text");
public sealed record ToolUseBlock(string Id, string Name, JsonObject Input) : ContentBlock("tool_use");
public sealed record ToolResultBlock(string ToolUseId, string Content, bool? IsError = null) : ContentBlock("tool_result");
```

### 1.2 Infrastructure layer — AnthropicAdapter

```
Pharma.AiAssistant.Infrastructure/
  Ai/
    Anthropic/
      AnthropicAdapter.cs     ← implements ILlmAdapter, gọi HTTP đến api.anthropic.com
      AnthropicRequest.cs     ← serialize sang format Anthropic API
      AnthropicResponse.cs    ← deserialize từ Anthropic API
```

```csharp
// AnthropicAdapter.cs — chỉ cần làm được ChatAsync
public async Task<LlmResponse> ChatAsync(IList<LlmMessage> messages, ChatOptions options, CancellationToken ct = default)
{
    // POST https://api.anthropic.com/v1/messages
    // Header: x-api-key, anthropic-version
    // Body: { model, max_tokens, system, messages }
}
```

### 1.3 Domain layer — AgentRunner

```
Pharma.AiAssistant.Domain/
  Ai/
    AgentRunner.cs    ← core while(true) loop
    RunOptions.cs     ← MaxTurns, CancellationToken, ...
```

```csharp
// AgentRunner.cs — vòng lặp chính, phase này chưa có tools
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
            // Không có tool call → xong
            var output = response.Content.OfType<TextBlock>().FirstOrDefault()?.Text ?? "";
            return new RunResult(output, conversation.Skip(messages.Count).ToList(), response.Usage);
        }

        // Phase 2 sẽ implement tool execution
        throw new NotImplementedException("Tool execution — implement in Phase 2");
    }

    return new RunResult("", [], TokenUsage.Zero);
}
```

### 1.4 Application layer — ChatUseCase

```
Pharma.AiAssistant.Application/
  UseCases/
    Chat/
      SimpleChatUseCase.cs
      SimpleChatRequest.cs    ← record SimpleChatRequest(string Message)
      SimpleChatResponse.cs   ← record SimpleChatResponse(string Reply)
```

```csharp
public async Task<SimpleChatResponse> ExecuteAsync(SimpleChatRequest request)
{
    var messages = new List<LlmMessage>
    {
        new("user", [new TextBlock(request.Message)])
    };
    var result = await _agentRunner.RunAsync(messages);
    return new SimpleChatResponse(result.Output);
}
```

### 1.5 API layer — Endpoint

```
Pharma.AiAssistant.API/
  Endpoints/
    ChatEndpoints.cs    ← POST /api/chat
  Program.cs
  appsettings.json      ← Anthropic:ApiKey, Anthropic:Model
```

**Verify Phase 1:**
```
POST /api/chat
{ "message": "Warfarin là thuốc gì?" }
→ Response có text từ Claude
```

---

## Phase 2 — Tools Infrastructure

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

## Phase 3 — Agent + Chat History

**Goal:** Multi-turn conversation. User hỏi "còn với người già?" → AI nhớ context.

### 3.1 Domain layer — Agent

```
Pharma.AiAssistant.Domain/
  Ai/
    Agent.cs            ← wrap AgentRunner, giữ messageHistory
    AgentConfig.cs      ← Name, Model, SystemPrompt, MaxTurns, Tools[]
    AgentStatus.cs      ← enum: Idle, Running, Completed, Error
```

```csharp
public sealed class Agent
{
    private AgentRunner? _runner;           // lazy init
    private List<LlmMessage> _history = [];
    public AgentStatus Status { get; private set; } = AgentStatus.Idle;

    // Fresh conversation — không dùng history
    public Task<RunResult> RunAsync(string prompt, RunOptions? options = null)
    {
        var messages = new List<LlmMessage> { LlmMessage.UserText(prompt) };
        return ExecuteAsync(messages, options);
    }

    // Multi-turn — append vào history
    public async Task<RunResult> PromptAsync(string message)
    {
        _history.Add(LlmMessage.UserText(message));
        var result = await ExecuteAsync([.._history]);
        _history.AddRange(result.Messages);
        return result;
    }

    public void Reset() { _history.Clear(); Status = AgentStatus.Idle; }
}
```

### 3.2 Domain layer — Chat session

```
Pharma.AiAssistant.Domain/
  ChatSession/
    ChatSession.cs         ← Entity: Id, UserId, Messages[], CreatedAt
    ChatMessage.cs         ← Role, Content, CreatedAt
  Interfaces/
    IChatSessionRepository.cs
```

### 3.3 Infrastructure layer — Postgres persistence

```
Pharma.AiAssistant.Infrastructure/
  Persistence/
    AiDbContext.cs
    ChatSessionRepository.cs    ← implements IChatSessionRepository
    Migrations/
```

### 3.4 Application layer — ChatWithHistoryUseCase

```csharp
public async Task<ChatResponse> ExecuteAsync(ChatRequest request)
{
    // Load session hoặc tạo mới
    var session = await _sessionRepo.GetOrCreateAsync(request.SessionId, request.UserId);

    // Build history từ DB
    var history = session.Messages
        .Select(m => new LlmMessage(m.Role, [new TextBlock(m.Content)]))
        .ToList();

    // Chat (dùng Agent.PromptAsync thay vì RunAsync)
    history.Add(LlmMessage.UserText(request.Message));
    var result = await _agentRunner.RunAsync(history);

    // Lưu lại
    session.AddMessage("user", request.Message);
    session.AddMessage("assistant", result.Output);
    await _sessionRepo.SaveAsync(session);

    return new ChatResponse(request.SessionId, result.Output);
}
```

**Verify Phase 3:**
```
POST /api/chat { sessionId: "abc", message: "Warfarin liều tối đa?" }
POST /api/chat { sessionId: "abc", message: "Còn với người già?" }
→ Response lần 2 nhắc đến Warfarin (nhớ context)
```

---

## Phase 4 — RAG: Vector Search

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
    Anthropic/
      AnthropicEmbeddingService.cs   ← implements IEmbeddingService (hoặc dùng OpenAI)
  Persistence/
    DrugKnowledgeRepository.cs      ← pgvector similarity search
    Migrations/                      ← thêm vector column
  Tools/
    SearchDrugTool.cs               ← implement thật (thay NotImplementedException)
    GetDrugInfoTool.cs              ← implement thật
```

```csharp
// SearchDrugTool.cs — implement thật
public async Task<ToolResult> ExecuteAsync(JsonObject input, CancellationToken ct)
{
    var query = input["query"]!.GetValue<string>();

    // 1. Embed câu hỏi thành vector
    var embedding = await _embeddingService.EmbedAsync(query, ct);

    // 2. Similarity search trong pgvector
    var docs = await _drugRepo.SearchSimilarAsync(embedding, topK: 5, ct);

    // 3. Format kết quả thành text cho LLM
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
POST /api/chat { message: "Warfarin tương tác với thuốc nào?" }
→ LLM gọi search_drug("Warfarin interactions")
→ Tool query pgvector → trả về docs
→ LLM trả lời có context từ drug DB
```

---

## Phase 5 — Multi-Agent (TaskQueue + AgentPool + SharedMemory)

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
    UnblockDependents(taskId);   // scan blocked tasks, promote ready ones
    if (IsComplete()) AllCompleted?.Invoke();
}

public void Fail(string taskId, string error)
{
    Update(taskId, TaskStatus.Failed, error);
    CascadeFailure(taskId);      // recursive fail downstream
}
```

```csharp
// AgentPool.cs — double semaphore
public async Task<RunResult> RunAsync(string agentName, string prompt)
{
    var agentLock = _agentLocks[agentName]; // SemaphoreSlim(1,1)

    await agentLock.WaitAsync();            // [1] per-agent lock trước
    try
    {
        await _poolSemaphore.WaitAsync();   // [2] pool-wide limit sau
        try   { return await _agents[agentName].RunAsync(prompt); }
        finally { _poolSemaphore.Release(); }
    }
    finally { agentLock.Release(); }
}
```

### 5.2 Application layer — MultiAgentAnalysisUseCase

```csharp
// Dùng trực tiếp TaskQueue + AgentPool, chưa có Coordinator
public async Task<string> ExecuteAsync(IReadOnlyList<AgentTask> tasks)
{
    var memory = new SharedMemory();
    var queue  = new TaskQueue();

    queue.on('task:ready', async (task) => {
        var context = await memory.GetSummaryAsync(task.DependsOn);
        var prompt  = $"{task.Prompt}\n\nContext:\n{context}";
        var result  = await _pool.RunAsync(task.Assignee, prompt);

        await memory.WriteAsync(task.Assignee, $"task:{task.Id}:result", result.Output);
        queue.Complete(task.Id, result.Output);
    });

    queue.AddBatch(tasks);
    await queue.WaitAllAsync();

    return await memory.GetSummaryAsync();
}
```

**Verify Phase 5:**
```
Tạo task list thủ công:
  Task A: Tìm Warfarin info (drug-retriever)
  Task B: Tìm Amiodarone info (drug-retriever)
  Task C: Phân tích tương tác (dependsOn: A, B) (analyst)

→ A và B chạy song song
→ C tự unblock khi A+B xong, có context từ SharedMemory
```

---

## Phase 6 — Coordinator + Orchestrator (Killer Feature)

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
    var plan = await coordinator.RunAsync(goal);     // output là JSON TaskList
    var tasks = JsonSerializer.Deserialize<List<AgentTask>>(plan.Output)!;

    // [2] Chạy qua TaskQueue + AgentPool (Phase 5)
    var result = await _multiAgentUseCase.ExecuteAsync(tasks);

    // [3] Coordinator synthesize final answer
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
POST /api/analyze với câu hỏi phức tạp
→ Log thấy Coordinator sinh ra task list JSON
→ TaskQueue chạy đúng thứ tự
→ Final answer có quality cao hơn Phase 3 (single agent)
```

---

## Thứ tự NuGet packages theo phase

| Phase | Package |
|-------|---------|
| 1 | `Anthropic.SDK` hoặc `System.Net.Http.Json` |
| 3 | `Microsoft.EntityFrameworkCore`, `Npgsql.EntityFrameworkCore.PostgreSQL` |
| 4 | `Pgvector.EntityFrameworkCore` |
| All | `Microsoft.AspNetCore.OpenApi`, `Scalar.AspNetCore` |

---

## Checklist bắt đầu Phase 1

- [ ] Tạo solution + 4 projects
- [ ] Copy `Directory.Build.props` từ identity-service
- [ ] Tạo `ContentBlock`, `LlmMessage`, `LlmResponse`, `TokenUsage`
- [ ] Tạo `ILlmAdapter` interface
- [ ] Implement `AnthropicAdapter` (POST to api.anthropic.com/v1/messages)
- [ ] Implement `AgentRunner` (while loop, không có tools)
- [ ] Implement `SimpleChatUseCase`
- [ ] Wire DI trong `Program.cs`
- [ ] Test: `POST /api/chat { "message": "Xin chào" }` → nhận reply từ Claude
