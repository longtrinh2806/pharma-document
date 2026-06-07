# Open Multi-Agent Framework — Giải thích toàn bộ kiến trúc

> **Dành cho:** Middle/Senior .NET Engineer mới học AI  
> **Source:** [open-multi-agent](https://github.com/anthropics/open-multi-agent) — đọc trực tiếp từ source code  
> **Mục tiêu:** Hiểu đủ để viết lại bằng C#

---

## Big Picture — Framework này giải quyết vấn đề gì?

**Vấn đề:** LLM là stateless. Mỗi lần gọi là một HTTP request độc lập — LLM không nhớ gì giữa các lần gọi. Ngoài ra, một LLM đơn lẻ không giỏi làm nhiều việc phức tạp song song.

**Giải pháp:** Tạo nhiều LLM instances (agents), mỗi cái có một vai trò chuyên biệt, phối hợp với nhau qua shared memory và task dependencies.

```
Browser / API
    │
    ▼
OpenMultiAgentOrchestrator   ← entry point duy nhất
    │
    ├── runAgent()            ← 1 agent, 1 câu hỏi đơn giản
    ├── runTasks()            ← nhiều tasks, bạn tự định nghĩa
    └── runTeam()             ← killer feature: coordinator tự chia task
            │
            ├── Coordinator (LLM) phân tích goal → JSON task list
            ├── TaskQueue        dependency graph, unblock khi xong
            ├── AgentPool        concurrency control (Semaphore)
            ├── Agent × N        mỗi agent = 1 AgentRunner + state
            │       │
            │       └── AgentRunner   vòng lặp LLM ↔ tool chính
            │               │
            │               └── LLMAdapter   HTTP call đến Anthropic/OpenAI/...
            │               └── ToolExecutor  bash, file_read, delegate_to_agent...
            └── SharedMemory     KV store, inject context vào prompt
```

**Tương đương .NET:**

| Framework concept | .NET analogy |
|---|---|
| `OpenMultiAgentOrchestrator` | `IHostBuilder` + orchestration service |
| `Team` | một module gồm nhiều services |
| `Agent` | một service có một trách nhiệm (SRP) |
| `AgentRunner` | core business logic của service đó |
| `Task` | Unit of Work |
| `LLMAdapter` | `IHttpClientFactory` (abstract provider) |
| `MemoryStore` | `IRepository<T>` |
| `AgentPool` | `SemaphoreSlim` + service registry |

---

## Layer 1: Kiểu dữ liệu nền tảng (`types.ts`)

Trước khi implement bất cứ thứ gì, cần hiểu 3 khái niệm cốt lõi.

### 1.1 ContentBlock — đơn vị nội dung

```typescript
TextBlock        → { type: "text",        text: "..." }
ToolUseBlock     → { type: "tool_use",    id: "abc", name: "bash", input: {...} }
ToolResultBlock  → { type: "tool_result", tool_use_id: "abc", content: "..." }
ImageBlock       → { type: "image",       source: { type: "base64", data: "..." } }
```

### 1.2 LLMMessage — một tin nhắn trong conversation

```typescript
{ role: "user" | "assistant", content: ContentBlock[] }
```

Một message có thể chứa **nhiều blocks**. Ví dụ: LLM reply vừa có text vừa gọi tool → 1 assistant message với 2 blocks.

### 1.3 Conversation = danh sách messages append liên tục

```
[
  { role: "user",      content: [TextBlock("Warfarin tối đa bao nhiêu mg?")] },
  { role: "assistant", content: [TextBlock("Tôi sẽ tra cứu..."), ToolUseBlock(id="1", name="search_db")] },
  { role: "user",      content: [ToolResultBlock(tool_use_id="1", content="10mg/day")] },
  { role: "assistant", content: [TextBlock("Liều tối đa Warfarin là 10mg/ngày")] },
]
```

LLM không "gọi" tool theo nghĩa thông thường. Nó **trả về JSON mô tả tool cần dùng**, framework execute thật rồi append kết quả vào conversation, gọi LLM lại.

**C# tương đương:**
```csharp
public abstract record ContentBlock(string Type);
public sealed record TextBlock(string Text) : ContentBlock("text");
public sealed record ToolUseBlock(string Id, string Name, JsonObject Input) : ContentBlock("tool_use");
public sealed record ToolResultBlock(string ToolUseId, string Content, bool? IsError = null) : ContentBlock("tool_result");

public sealed record LlmMessage(string Role, IReadOnlyList<ContentBlock> Content);
```

---

## Layer 2: AgentRunner — Trái tim của framework (`runner.ts`)

Đây là component quan trọng nhất. Toàn bộ framework xây quanh vòng lặp này.

### 2.1 Vòng lặp chính

```
stream(initialMessages):
    conversation = [...initialMessages]
    turns = 0

    WHILE TRUE:
        if (aborted || turns >= maxTurns) → BREAK
        turns++

        ─── Step 1: Gọi LLM ──────────────────────────────────────────
        response = await llmAdapter.chat(conversation, options)
        totalUsage += response.usage

        ─── Step 2: Append assistant reply ───────────────────────────
        conversation.push({ role: "assistant", content: response.content })
        yield { type: "text", data: response.textContent }   // stream cho frontend

        ─── Step 3: Có tool call không? ──────────────────────────────
        toolUseBlocks = response.content.filter(b => b.type == "tool_use")

        if toolUseBlocks.isEmpty:
            finalOutput = response.textContent
            BREAK   ← LLM nói "xong rồi", không cần tool nữa

        ─── Step 4: Execute TẤT CẢ tools SONG SONG ──────────────────
        executions = await Promise.all(
            toolUseBlocks.map(block => toolExecutor.execute(block))
        )

        ─── Step 5: Append tool results vào conversation ─────────────
        conversation.push({
            role: "user",
            content: executions.map(e => ToolResultBlock(e.id, e.output))
        })

        → Loop lại Step 1: LLM đọc tool results, tiếp tục reasoning
```

### 2.2 Điều kiện thoát vòng lặp

| Điều kiện | Hành động |
|---|---|
| LLM trả về không có `tool_use` block | BREAK — hoàn thành bình thường |
| `turns >= maxTurns` (default: 10) | BREAK — tránh vòng lặp vô tận |
| `abortSignal` triggered | BREAK — cancel từ bên ngoài |
| Token budget exceeded | BREAK + emit budget_exceeded |
| Loop detection: cùng tool call lặp ≥ 3 lần | WARN hoặc TERMINATE |

### 2.3 Context management — tránh context window đầy

Khi conversation quá dài (nhiều turns), framework có 4 chiến lược:

```typescript
contextStrategy:
  | { type: "sliding-window", maxTurns: 10 }     // giữ N turns gần nhất, bỏ cũ
  | { type: "summarize", maxTokens: 8000 }        // gọi LLM tóm tắt phần cũ
  | { type: "compact", maxTokens: 8000 }          // rule-based: compress tool results dài
  | { type: "custom", compress: fn }              // tự implement
```

Mặc định không có strategy → conversation cứ grow. Quan trọng khi implement chatbot nhiều turns.

### 2.4 C# tương đương

```csharp
public async Task<RunResult> RunAsync(IList<LlmMessage> messages, RunOptions options = default)
{
    var conversation = new List<LlmMessage>(messages);
    var allToolCalls = new List<ToolCallRecord>();
    string finalOutput = "";
    var totalUsage = TokenUsage.Zero;
    int turns = 0;

    while (true)
    {
        if (turns >= _maxTurns) break;
        turns++;

        // [1] Call LLM
        var response = await _adapter.ChatAsync(conversation, _chatOptions, options.CancellationToken);
        totalUsage += response.Usage;

        // [2] Append assistant message
        conversation.Add(new LlmMessage("assistant", response.Content));

        // [3] Check for tool calls
        var toolUseBlocks = response.Content.OfType<ToolUseBlock>().ToList();
        if (!toolUseBlocks.Any())
        {
            finalOutput = response.Content.OfType<TextBlock>().FirstOrDefault()?.Text ?? "";
            break;
        }

        // [4] Execute all tools in parallel
        var executions = await Task.WhenAll(
            toolUseBlocks.Select(block => ExecuteToolAsync(block, options))
        );

        // [5] Append tool results as new user message
        var resultBlocks = executions.Select(e => (ContentBlock)new ToolResultBlock(e.ToolUseId, e.Output));
        conversation.Add(new LlmMessage("user", resultBlocks.ToList()));

        allToolCalls.AddRange(executions.Select(e => e.ToRecord()));
    }

    return new RunResult(
        Output: finalOutput,
        Messages: conversation.Skip(messages.Count).ToList(),
        ToolCalls: allToolCalls,
        TokenUsage: totalUsage,
        Turns: turns
    );
}
```

---

## Layer 3: Agent (`agent.ts`)

`Agent` là wrapper quanh `AgentRunner`. Nó thêm:
- **State management:** `idle → running → completed | error`
- **2 execution modes:**
  - `run(prompt)` — fresh conversation, không dùng history
  - `prompt(message)` — multi-turn, append vào history
- **Hooks:** `beforeRun` (modify prompt), `afterRun` (transform result)
- **Structured output:** validate kết quả bằng Zod schema, retry nếu fail

```
agent.run("câu hỏi")     → AgentRunner với messages = [{ user: "câu hỏi" }]
agent.prompt("turn 2")   → AgentRunner với messages = [turn1..., { user: "turn 2" }]
agent.stream("câu hỏi")  → AsyncGenerator<StreamEvent>
```

**Lazy initialization:** `AgentRunner` chỉ được tạo khi lần đầu gọi `run/prompt/stream`. Tránh import SDK không cần thiết.

**C# tương đương:**
```csharp
public sealed class Agent
{
    private AgentRunner? _runner;   // lazy init
    private AgentStatus _status = AgentStatus.Idle;
    private List<LlmMessage> _history = new();
    public AgentConfig Config { get; }

    // Fresh conversation — không dùng history
    public Task<AgentRunResult> RunAsync(string prompt, RunOptions? options = null)
    {
        var messages = new List<LlmMessage> { LlmMessage.UserText(prompt) };
        return ExecuteAsync(messages, options);
    }

    // Multi-turn — dùng và update history
    public async Task<AgentRunResult> PromptAsync(string message)
    {
        _history.Add(LlmMessage.UserText(message));
        var result = await ExecuteAsync(_history.ToList());
        _history.AddRange(result.Messages);  // persist for next turn
        return result;
    }

    public void Reset()
    {
        _history.Clear();
        _status = AgentStatus.Idle;
    }
}
```

---

## Layer 4: TaskQueue — Dependency Graph (`task/queue.ts`)

TaskQueue quản lý lifecycle của tasks và tự động "mở khóa" task phụ thuộc khi dependency hoàn thành.

### 4.1 State machine của Task

```
                    ┌─────────────────────────────┐
                    │                             │
   add(task)        ▼         complete()          ▼
  ──────────► pending ────────────────────► completed (terminal)
                    │
  dependsOn chưa    │  unblockDependents()
  thỏa mãn          │  khi dependency complete
                    ▼        ▲
                 blocked ────┘
                    │
                    │  fail()
                    ▼
               failed (terminal)  → cascade: fail tất cả dependents
```

### 4.2 Logic unblock dependency

Khi task A hoàn thành:
```
complete("task-A"):
    1. task-A.status = "completed"
    2. scan TẤT CẢ blocked tasks:
       - task có dependsOn.includes("task-A")?
       - TẤT CẢ dependencies của task đó đã "completed"?
       - nếu yes → status = "pending"
    3. emit("task:ready") cho từng task vừa unblock
```

### 4.3 Cascade failure

Khi task A fail → tất cả tasks phụ thuộc A cũng fail (đệ quy):
```
A fails → B (dependsOn A) fails → C (dependsOn B) fails → ...
```

**C# tương đương:**
```csharp
public sealed class TaskQueue
{
    private readonly Dictionary<string, AgentTask> _tasks = new();

    public event Action<AgentTask>? TaskReady;
    public event Action<AgentTask>? TaskFailed;

    public void Complete(string taskId, string result)
    {
        var task = Update(taskId, TaskStatus.Completed, result);
        UnblockDependents(taskId);
        if (IsComplete()) AllComplete?.Invoke();
    }

    public void Fail(string taskId, string error)
    {
        Update(taskId, TaskStatus.Failed, error);
        CascadeFailure(taskId);  // recursive
    }

    private void UnblockDependents(string completedId)
    {
        foreach (var task in _tasks.Values.Where(t => t.Status == TaskStatus.Blocked))
        {
            if (task.DependsOn?.Contains(completedId) != true) continue;

            // check ALL dependencies are completed
            bool ready = task.DependsOn.All(depId =>
                _tasks.TryGetValue(depId, out var dep) && dep.Status == TaskStatus.Completed);

            if (ready)
            {
                task.Status = TaskStatus.Pending;
                TaskReady?.Invoke(task);
            }
        }
    }

    private void CascadeFailure(string failedId)
    {
        foreach (var task in _tasks.Values.Where(t =>
            t.Status is TaskStatus.Blocked or TaskStatus.Pending &&
            t.DependsOn?.Contains(failedId) == true))
        {
            task.Status = TaskStatus.Failed;
            task.Result = $"Cancelled: dependency '{failedId}' failed.";
            TaskFailed?.Invoke(task);
            CascadeFailure(task.Id);  // recurse
        }
    }
}
```

---

## Layer 5: AgentPool — Concurrency Control (`agent/pool.ts`)

Pool kiểm soát bao nhiêu agents chạy đồng thời với **2 lớp semaphore**.

### 5.1 Tại sao cần 2 lớp?

```
agentLock (Semaphore(1) per agent):
    Mỗi Agent có mutable state (messageHistory, status).
    Nếu 2 tasks cùng chạy trên 1 agent → race condition.
    → agentLock serialize: task B phải đợi task A xong mới chạy trên agent đó.

poolSemaphore (Semaphore(maxConcurrency)):
    Giới hạn tổng số LLM calls song song.
    VD: maxConcurrency=5 → tối đa 5 HTTP calls đến Anthropic cùng lúc.
    → Tránh quá tải API, kiểm soát cost.
```

### 5.2 Thứ tự acquire — quan trọng để tránh deadlock

```
ĐÚNG:
    await agentLock.acquire()    ← chờ agent rảnh TRƯỚC
      await poolSemaphore.acquire()  ← rồi mới lấy slot trong pool
        await agent.run(prompt)
      poolSemaphore.release()
    agentLock.release()

SAI (deadlock scenario):
    Giả sử maxConcurrency=2, có 2 tasks cùng lúc với CÙNG agent:
    - Task A lấy poolSlot → đợi agentLock → giữ slot mãi
    - Task B lấy poolSlot → đợi agentLock → giữ slot mãi
    → Cả 2 slots bị giữ, không ai nhả → deadlock
```

### 5.3 runEphemeral — cho delegate_to_agent

Khi agent A muốn "nhờ" agent B xử lý một việc (delegation), không dùng `run()` mà dùng `runEphemeral()`:
- Tạo **instance Agent mới** cho B (không reuse instance trong pool)
- Chỉ acquire poolSemaphore, không acquire agentLock
- Tránh deadlock khi A→B→A (mutual delegation)

**C# tương đương:**
```csharp
public sealed class AgentPool
{
    private readonly Dictionary<string, Agent> _agents = new();
    private readonly Dictionary<string, SemaphoreSlim> _agentLocks = new();
    private readonly SemaphoreSlim _poolSemaphore;

    public AgentPool(int maxConcurrency = 5)
        => _poolSemaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);

    public void Add(Agent agent)
    {
        _agents[agent.Name] = agent;
        _agentLocks[agent.Name] = new SemaphoreSlim(1, 1);
    }

    public async Task<AgentRunResult> RunAsync(string agentName, string prompt, RunOptions? options = null)
    {
        var agent = _agents[agentName];
        var agentLock = _agentLocks[agentName];

        await agentLock.WaitAsync();      // [1] per-agent lock trước
        try
        {
            await _poolSemaphore.WaitAsync();  // [2] pool slot sau
            try
            {
                return await agent.RunAsync(prompt, options);
            }
            finally { _poolSemaphore.Release(); }
        }
        finally { agentLock.Release(); }
    }

    // Dùng cho delegation: agent instance do caller tạo, chỉ cần pool slot
    public async Task<AgentRunResult> RunEphemeralAsync(Agent agent, string prompt, RunOptions? options = null)
    {
        await _poolSemaphore.WaitAsync();
        try { return await agent.RunAsync(prompt, options); }
        finally { _poolSemaphore.Release(); }
    }

    public int AvailableSlots => _poolSemaphore.CurrentCount;
}
```

---

## Layer 6: SharedMemory — KV Store với namespace (`memory/shared.ts`)

Đơn giản hơn những gì Gemini mô tả. Không có TTL, không có turn counter trong phiên bản hiện tại của repo.

### 6.1 Cấu trúc

```
SharedMemory
    └── InMemoryStore (Dictionary<string, MemoryEntry>)

MemoryEntry:
    key:       "researcher/findings"       ← "<agentName>/<localKey>"
    value:     "Warfarin max dose: 10mg"   ← luôn là string
    metadata:  { agent: "researcher" }
    createdAt: DateTime
```

### 6.2 Namespace pattern

```
write("researcher", "findings", "Warfarin max dose: 10mg")
→ store["researcher/findings"] = value

write("analyst", "conclusion", "Không kết hợp Warfarin + Aspirin")
→ store["analyst/conclusion"] = value
```

Mục đích namespace:
- Tránh collision giữa agents
- Biết entry do agent nào write mà không cần parse
- `listByAgent("researcher")` → filter theo prefix

### 6.3 getSummary() — cầu nối giữa memory và LLM

Method quan trọng nhất. Biến KV store thành text để inject vào prompt:

```
getSummary() →

"## Shared Team Memory

### researcher
- findings: Warfarin max dose: 10mg
- side-effects: Chảy máu, bầm tím

### analyst
- conclusion: Không kết hợp Warfarin + Aspirin (score 0.92)"
```

LLM hiểu markdown tốt → inject trực tiếp vào system prompt hoặc user message.

### 6.4 Khi nào framework tự inject

Trong `buildTaskPrompt()` (orchestrator):
- **Default** (`memoryScope: "dependencies"`): chỉ inject kết quả của dependency tasks
- **Explicit** (`memoryScope: "all"`): inject toàn bộ shared memory summary

Mặc định **không** inject tất cả — chỉ inject đúng những gì task cần (dependency results). Điều này quan trọng để tránh bloat context window.

**C# tương đương:**
```csharp
public sealed class SharedMemory
{
    private readonly Dictionary<string, MemoryEntry> _store = new();

    public Task WriteAsync(string agentName, string key, string value, Dictionary<string, object>? metadata = null)
    {
        var namespacedKey = $"{agentName}/{key}";
        _store[namespacedKey] = new MemoryEntry(
            Key: namespacedKey,
            Value: value,
            Metadata: new Dictionary<string, object>(metadata ?? []) { ["agent"] = agentName },
            CreatedAt: _store.TryGetValue(namespacedKey, out var existing)
                ? existing.CreatedAt      // preserve original createdAt on update
                : DateTime.UtcNow
        );
        return Task.CompletedTask;
    }

    public string GetSummary()
    {
        if (_store.Count == 0) return string.Empty;

        var byAgent = _store.Values
            .GroupBy(e => e.Key.Contains('/') ? e.Key.Split('/')[0] : "_unknown");

        var sb = new StringBuilder("## Shared Team Memory\n\n");
        foreach (var group in byAgent)
        {
            sb.AppendLine($"### {group.Key}");
            foreach (var entry in group)
            {
                var localKey = entry.Key.Contains('/') ? entry.Key.Split('/')[1] : entry.Key;
                var display = entry.Value.Length > 200
                    ? entry.Value[..197] + "…"
                    : entry.Value;
                sb.AppendLine($"- {localKey}: {display}");
            }
            sb.AppendLine();
        }

        return sb.ToString().TrimEnd();
    }
}
```

---

## Layer 7: Coordinator Pattern — Killer Feature (`orchestrator.ts`)

`runTeam()` là 5 bước tuyến tính. Đây là điểm khác biệt lớn nhất với "gọi LLM thông thường".

### 7.1 Flow đầy đủ

```
runTeam(team, "Phân tích tương tác thuốc Warfarin + Aspirin"):

─── Bước 1: Coordinator decompose ──────────────────────────────────
coordinator (LLM) nhận:
  - goal: "Phân tích tương tác thuốc..."
  - team roster: [researcher(claude), analyst(claude), writer(claude)]

output của coordinator (JSON):
  [
    { title: "Tìm dữ liệu Warfarin",   assignee: "researcher", dependsOn: [] },
    { title: "Tìm dữ liệu Aspirin",    assignee: "researcher", dependsOn: [] },
    { title: "Phân tích tương tác",    assignee: "analyst",    dependsOn: ["Tìm dữ liệu Warfarin", "Tìm dữ liệu Aspirin"] },
    { title: "Viết báo cáo",           assignee: "writer",     dependsOn: ["Phân tích tương tác"] },
  ]

─── Bước 2: Build TaskQueue ─────────────────────────────────────────
TaskQueue nhận task list, build dependency graph:
  Task-1 (pending)   Task-2 (pending)
       \                  /
        \                /
         Task-3 (blocked: đợi 1+2)
              │
         Task-4 (blocked: đợi 3)

─── Bước 3: Scheduler.autoAssign() ─────────────────────────────────
Assign tasks chưa có assignee bằng keyword scoring

─── Bước 4: executeQueue() ──────────────────────────────────────────
Round 1:
  pending = [Task-1, Task-2]
  dispatch song song: researcher chạy Task-1 ∥ researcher chạy Task-2
  await Promise.all()
  → Task-1 done → unblock Task-3? (Task-2 chưa xong → không)
  → Task-2 done → unblock Task-3? (Task-1 đã xong → YES)

Round 2:
  pending = [Task-3]
  analyst chạy Task-3 với context từ Task-1 + Task-2 inject vào prompt
  → Task-3 done → unblock Task-4

Round 3:
  pending = [Task-4]
  writer chạy Task-4 với context từ Task-3

─── Bước 5: Coordinator synthesize ─────────────────────────────────
coordinator (LLM) nhận tất cả task results
→ tổng hợp final answer
```

### 7.2 Short-circuit optimization

Nếu goal **đơn giản** (ngắn, không có từ "step/phase/collaborate/in parallel") → bỏ qua coordinator:
```
"Warfarin liều tối đa?" → đơn giản → chọn agent phù hợp nhất → gọi thẳng
"Bước 1: tìm X, bước 2: phân tích Y" → phức tạp → cần coordinator
```

Detector dùng regex pattern matching và độ dài goal (< 200 chars).

### 7.3 Task prompt construction

Trước khi chạy mỗi task, framework build prompt như sau:

```
# Task: Phân tích tương tác thuốc

<task.description>

## Context from prerequisite tasks

### Tìm dữ liệu Warfarin (by researcher)
Warfarin: anticoagulant, max dose 10mg/day, INR monitoring required...

### Tìm dữ liệu Aspirin (by researcher)
Aspirin: NSAID, antiplatelet properties, inhibits COX-1...

## Messages from team members
- researcher: Lưu ý Warfarin có window điều trị hẹp
```

Analyst nhận context đầy đủ từ researcher → output tốt hơn nhiều so với gọi standalone.

---

## Thứ tự implement C# từ dưới lên

```
1. Records/Types
   LlmMessage, ContentBlock subtypes, AgentTask, AgentConfig, RunResult

2. ILlmAdapter
   ChatAsync(messages, options) → LlmResponse
   → Implement: AnthropicAdapter (gọi claude-opus-4-6)

3. IToolDefinition + ToolRegistry + ToolExecutor
   Register(ITool) / ExecuteAsync(name, input, context) → ToolResult
   → Implement built-in tools sau: SearchDb, GetDrugInfo, ...

4. AgentRunner                          ← trái tim, viết trước
   Core loop: while(true) { LLM → tool? → execute → append → repeat }

5. Agent
   Wrap AgentRunner + state (idle/running/completed)
   RunAsync() / PromptAsync() / StreamAsync()

6. TaskQueue
   Dictionary<AgentTask> + dependency unblocking + cascade failure

7. AgentPool
   Dictionary<Agent> + SemaphoreSlim (per-agent + pool-wide)

8. SharedMemory
   Dictionary<MemoryEntry> với namespace prefix + GetSummary()

9. Scheduler
   AutoAssignAsync() — keyword match task description → agent systemPrompt

10. OpenMultiAgentOrchestrator
    RunAgentAsync() / RunTasksAsync() / RunTeamAsync()
    Coordinator pattern trong RunTeamAsync()
```

---

## Key Takeaways để nhớ

1. **LLM không "gọi" tool** — LLM trả về JSON mô tả tool, framework execute thật rồi append kết quả, gọi LLM lại. Conversation là list messages grow dần.

2. **AgentRunner là core** — tất cả thứ khác chỉ là infrastructure xung quanh vòng lặp `while(true) { LLM → tool? → execute → loop }`.

3. **2 tầng Semaphore trong Pool** — per-agent lock (prevent race condition) + pool-wide limit (prevent API overload). Thứ tự acquire quan trọng để tránh deadlock.

4. **TaskQueue unblock tự động** — không cần poll. Khi dependency complete, queue tự emit event, orchestrator dispatch task tiếp theo.

5. **SharedMemory inject theo dependency** — framework không inject tất cả memory, chỉ inject kết quả của dependency tasks. Tránh bloat context window.

6. **Coordinator là LLM** — không phải hard-coded logic. Framework nhờ chính LLM phân tích goal và quyết định task decomposition. Đây là điểm linh hoạt nhất.
