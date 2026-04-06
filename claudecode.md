# Ultra (Claude Code) — Design Document

## Концепция

В dropdown выбора модели чата добавляется четвёртый вариант **"Ultra (Claude Code)"**. При его выборе сообщение пользователя выполняется не через OpenRouter LLM API, а через **Claude Code CLI** (`claude -p`), который запускается как subprocess, читает stream-json output построчно и транслирует события в WebSocket.

```
Текущие варианты:              Добавляется:
├── Standard (Sonnet)          ├── Ultra (Claude Code)
├── Fast (Haiku)               │
└── Advanced (Opus)            │
```

Референсная имплементация — `ClaudeCodeAgent` из AiCoreApi.

---

## Архитектура

### Текущая цепочка (OpenRouter)

```
Frontend  ──ws──▶  ChatWebSocketHandler
                        │
                        ▼
                   SmartOrchestrator
                        │
               ┌────────┴────────┐
               ▼                 ▼
         SingleAgentLoop    SpecialistRunner
               │                 │
               ▼                 ▼
          AnthropicClient   AnthropicClient
          (OpenRouter API)  (OpenRouter API)
```

### Новая цепочка (Claude Code)

```
Frontend  ──ws──▶  ChatWebSocketHandler
                        │
                        │  model == "claude-code"
                        ▼
                   SmartOrchestrator
                        │
                        ▼  (bypass planner, route = "claude-code")
                   ClaudeCodeRunner
                        │
                        ▼
                   Process.Start("claude", "-p ... --output-format stream-json")
                        │  stdout: NDJSON line-by-line
                        ▼
                   ParseStreamLine → ChatChannel → WebSocket → Frontend
```

**Ключевой принцип:** Claude Code CLI запускается как child process. `stdout` читается построчно (NDJSON). Каждая строка парсится и транслируется в существующий WebSocket протокол. Паттерн один в один как в референсной имплементации `ClaudeCodeAgent`.

---

## Детальный план реализации

### 1. Frontend: добавить "Ultra" в dropdown

**Файл: `frontend/src/components/ChatInterface.tsx`**

```tsx
// Расширить тип модели:
onChange={(e) => setModel(e.target.value as "sonnet" | "haiku" | "opus" | "claude-code")}

// Добавить option в select:
<option value="sonnet">Standard (Sonnet)</option>
<option value="haiku">Fast (Haiku)</option>
<option value="opus">Advanced (Opus)</option>
<option value="claude-code">Ultra (Claude Code)</option>
```

При `model === "claude-code"` скрыть чекбокс "Extended thinking" (Claude Code управляет thinking сам):

```tsx
{model !== "claude-code" && (
  <label className="flex items-center gap-1.5 ...">
    <input type="checkbox" checked={useThinking} ... />
    Extended thinking
  </label>
)}
```

**Файлы без изменений:** `wsClient.ts`, `useWebSocket.ts` — `model: "claude-code"` передаётся как строка через существующий протокол.

### 2. Frontend: новые WS message types

**Файл: `frontend/src/lib/wsClient.ts`**

```typescript
export type WsMessageType =
  | ... // существующие
  | "claude_code_start"       // Claude Code процесс запущен
  | "claude_code_tool_use"    // Claude Code вызывает инструмент
  | "claude_code_tool_result" // Результат инструмента
  | "claude_code_cost"        // Стоимость запроса (из result)
  ;
```

Типы `token`, `thinking_start/end`, `done`, `error` — **переиспользуются без изменений**.

### 3. Backend: распознать model "claude-code"

**Файл: `backend/src/AgenticIT.Api/WebSockets/ChatWebSocketHandler.cs`**

В блоке резолюции модели (строки 162-168):

```csharp
var resolvedModel = requestedModel switch
{
    "opus"        => "anthropic/claude-opus-4.6",
    "sonnet"      => "anthropic/claude-sonnet-4.6",
    "haiku"       => "anthropic/claude-haiku-4.5",
    "claude-code" => "claude-code",  // ← специальный маркер
    _             => null,
};
```

### 4. Backend: SmartOrchestrator — bypass planner

**Файл: `backend/src/AgenticIT.Agent/Orchestration/SmartOrchestrator.cs`**

В `RunAsync`, **перед** вызовом planner, добавить early return:

```csharp
// ── Claude Code route: bypass planner entirely ──────────────
if (req.Model == "claude-code")
{
    await req.WebSocket.SendObjectAsync(new { type = "claude_code_start" });

    var mergedContext = CombineContexts(projectContext, req.ExtraSystem);

    await _claudeCodeRunner.RunAsync(new ClaudeCodeRequest
    {
        UserMessage = req.Parameter1 ?? req.UserMessage,
        ConversationHistory = req.ConversationHistory,
        WebSocket = req.WebSocket,
        ProjectContext = mergedContext,
        ProjectId = req.ProjectId,
        ChatId = req.ChatId,
        Attachments = req.Attachments,
        TenantId = req.TenantId,
        UserId = req.UserId,
    }, ct);

    return;
}
```

Инжектировать `IClaudeCodeRunner` в конструктор.

### 5. Backend: ClaudeCodeRunner — ядро интеграции

**Новый файл: `backend/src/AgenticIT.Agent/ClaudeCode/ClaudeCodeRunner.cs`**

Паттерн один в один из AiCoreApi: запуск процесса → чтение stdout → парсинг NDJSON → трансляция в ChatChannel.

```csharp
using System.Diagnostics;
using System.Text;
using System.Text.Json;
using AgenticIT.Agent.Channel;
using AgenticIT.Infrastructure.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace AgenticIT.Agent.ClaudeCode;

public sealed class ClaudeCodeRunner : IClaudeCodeRunner
{
    private static readonly object InstallLock = new();
    private static string? _installedVersion;

    private readonly AgenticITOptions _options;
    private readonly ILogger<ClaudeCodeRunner> _logger;

    // Streaming state per-request (content_block_start/delta/stop)
    private readonly Dictionary<int, StringBuilder> _streamingBuffers = new();
    private readonly Dictionary<int, string> _streamingBlockTypes = new();
    private readonly Dictionary<int, string> _streamingToolNames = new();

    public ClaudeCodeRunner(
        IOptions<AgenticITOptions> options,
        ILogger<ClaudeCodeRunner> logger)
    {
        _options = options.Value;
        _logger = logger;
    }

    public async Task RunAsync(ClaudeCodeRequest request, CancellationToken ct = default)
    {
        var timeoutMinutes = _options.ClaudeCodeTimeoutMinutes > 0
            ? _options.ClaudeCodeTimeoutMinutes : 10;

        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromMinutes(timeoutMinutes));

        // 1. Ensure Claude Code is installed
        EnsureInstalled(_options.ClaudeCodeVersion ?? "latest");

        // 2. Setup credentials file
        await SetupCredentialsAsync(_options.ClaudeCodeCredentialsJson);

        // 3. Prepare working directory
        var workDir = Path.Combine(Path.GetTempPath(), "claude", Guid.NewGuid().ToString("N")[..12]);
        Directory.CreateDirectory(workDir);

        try
        {
            // 4. Write project artifacts to working directory
            await WriteAttachmentsToDirectoryAsync(request, workDir);

            // 5. Write CLAUDE.md with project context
            string? claudeMdPath = null;
            if (request.ProjectContext is not null)
            {
                claudeMdPath = Path.Combine(workDir, "CLAUDE.md");
                await File.WriteAllTextAsync(claudeMdPath, request.ProjectContext);
            }

            // 6. Snapshot directory state before execution
            var beforeSnapshot = SnapshotDirectory(workDir);

            // 7. Execute Claude Code
            var result = await ExecuteAsync(request, workDir, timeoutMinutes, cts.Token);

            // 8. Cleanup CLAUDE.md before collecting files
            if (claudeMdPath != null)
                TryDelete(claudeMdPath);

            // 9. Send final result
            await request.WebSocket.SendObjectAsync(new { type = "done" });

            // 10. Add result text to conversation history
            request.ConversationHistory.Add(new()
            {
                ["role"] = "assistant",
                ["content"] = result,
            });
        }
        finally
        {
            try { Directory.Delete(workDir, true); }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to cleanup temp directory {Path}", workDir);
            }
        }
    }

    // ── Process execution ────────────────────────────────────────────────

    private async Task<string> ExecuteAsync(
        ClaudeCodeRequest request, string workDir, int timeoutMinutes,
        CancellationToken ct)
    {
        var claudePath = FindClaudeBinary();

        var process = new Process
        {
            StartInfo = new ProcessStartInfo
            {
                FileName = claudePath,
                WorkingDirectory = workDir,
                RedirectStandardOutput = true,
                RedirectStandardError = true,
                RedirectStandardInput = true,
                UseShellExecute = false,
                CreateNoWindow = true,
            }
        };

        // Build arguments using ArgumentList (safe from injection)
        process.StartInfo.ArgumentList.Add("-p");
        process.StartInfo.ArgumentList.Add(request.UserMessage);
        process.StartInfo.ArgumentList.Add("--output-format");
        process.StartInfo.ArgumentList.Add("stream-json");
        process.StartInfo.ArgumentList.Add("--verbose");
        process.StartInfo.ArgumentList.Add("--include-partial-messages");

        if (_options.ClaudeCodeMaxTurns > 0)
        {
            process.StartInfo.ArgumentList.Add("--max-turns");
            process.StartInfo.ArgumentList.Add(_options.ClaudeCodeMaxTurns.ToString());
        }

        // Append system prompt via temp file (safe for large prompts)
        string? systemPromptFile = null;
        if (request.ProjectContext is not null)
        {
            systemPromptFile = Path.Combine(Path.GetTempPath(),
                $"claude-system-{Guid.NewGuid():N}.txt");
            await File.WriteAllTextAsync(systemPromptFile,
                BuildAppendSystemPrompt(request), ct);
            process.StartInfo.ArgumentList.Add("--append-system-prompt-file");
            process.StartInfo.ArgumentList.Add(systemPromptFile);
        }

        // MCP config
        string? mcpConfigFile = null;
        if (!string.IsNullOrEmpty(_options.ClaudeCodeMcpConfig))
        {
            mcpConfigFile = Path.Combine(Path.GetTempPath(),
                $"claude-mcp-{Guid.NewGuid():N}.json");
            await File.WriteAllTextAsync(mcpConfigFile,
                _options.ClaudeCodeMcpConfig, ct);
            process.StartInfo.ArgumentList.Add("--mcp-config");
            process.StartInfo.ArgumentList.Add(mcpConfigFile);
        }

        // Extra CLI arguments from config
        if (!string.IsNullOrWhiteSpace(_options.ClaudeCodeCliArgs))
        {
            foreach (var arg in SplitCliArguments(_options.ClaudeCodeCliArgs))
                process.StartInfo.ArgumentList.Add(arg);
        }

        // Environment variables
        if (!string.IsNullOrEmpty(_options.ClaudeCodeApiKey))
            process.StartInfo.Environment["ANTHROPIC_API_KEY"] = _options.ClaudeCodeApiKey;

        try
        {
            process.Start();
            process.StandardInput.Close(); // prevent hanging

            // Read stderr in background
            var stderrBuilder = new StringBuilder();
            var stderrTask = Task.Run(async () =>
            {
                while (await process.StandardError.ReadLineAsync(ct) is { } line)
                    stderrBuilder.AppendLine(line);
            }, ct);

            // Stream stdout line-by-line (NDJSON)
            var finalResult = string.Empty;

            while (await process.StandardOutput.ReadLineAsync(ct) is { } line)
            {
                if (string.IsNullOrWhiteSpace(line)) continue;
                await ProcessStreamLineAsync(line, request.WebSocket, ref finalResult, ct);
            }

            await stderrTask;
            await process.WaitForExitAsync(ct);

            if (process.ExitCode != 0 && string.IsNullOrEmpty(finalResult))
            {
                var stderr = stderrBuilder.ToString();
                _logger.LogWarning("Claude Code failed (exit {Code}): {Stderr}",
                    process.ExitCode, stderr);
                throw new InvalidOperationException(
                    $"Claude Code failed (exit code {process.ExitCode}): {stderr}");
            }

            return finalResult;
        }
        catch (OperationCanceledException)
        {
            try { process.Kill(true); } catch { /* best effort */ }
            throw;
        }
        finally
        {
            TryDelete(systemPromptFile);
            TryDelete(mcpConfigFile);
        }
    }

    // ── NDJSON stream processing ─────────────────────────────────────────
    //
    // Claude Code с --output-format stream-json выдаёт построчно JSON:
    //
    //   {"type":"system", "session_id":"...", "model":"...", ...}
    //   {"type":"content_block_start", "index":0, "content_block":{"type":"thinking"}}
    //   {"type":"content_block_delta", "index":0, "delta":{"type":"thinking_delta","thinking":"..."}}
    //   {"type":"content_block_stop", "index":0}
    //   {"type":"content_block_start", "index":1, "content_block":{"type":"text"}}
    //   {"type":"content_block_delta", "index":1, "delta":{"type":"text_delta","text":"..."}}
    //   {"type":"content_block_stop", "index":1}
    //   {"type":"content_block_start", "index":2, "content_block":{"type":"tool_use","name":"Bash"}}
    //   {"type":"content_block_delta", "index":2, "delta":{"type":"input_json_delta","partial_json":"..."}}
    //   {"type":"content_block_stop", "index":2}
    //   {"type":"assistant", "message":{"content":[...]}}    // полное сообщение
    //   {"type":"user", "message":{"content":[...]}}          // tool_result
    //   {"type":"result", "result":"...", "total_cost_usd":0.05, ...}

    private async Task ProcessStreamLineAsync(
        string line, ChatChannel ws, ref string finalResult, CancellationToken ct)
    {
        try
        {
            using var json = JsonDocument.Parse(line);
            var root = json.RootElement;

            if (!root.TryGetProperty("type", out var typeProp))
                return;

            var type = typeProp.GetString();

            switch (type)
            {
                case "system":
                    await ProcessSystemEventAsync(root, ws);
                    break;

                case "content_block_start":
                    ProcessContentBlockStart(root);
                    break;

                case "content_block_delta":
                    await ProcessContentBlockDeltaAsync(root, ws);
                    break;

                case "content_block_stop":
                    await ProcessContentBlockStopAsync(root, ws);
                    break;

                case "assistant":
                    await ProcessAssistantEventAsync(root, ws);
                    break;

                case "user":
                    await ProcessUserEventAsync(root, ws);
                    break;

                case "result":
                    finalResult = await ProcessResultEventAsync(root, ws);
                    break;
            }
        }
        catch (JsonException)
        {
            _logger.LogDebug("Non-JSON line from claude: {Line}", line);
        }
    }

    private async Task ProcessSystemEventAsync(JsonElement root, ChatChannel ws)
    {
        var model = root.TryGetProperty("model", out var m) ? m.GetString() : "unknown";
        var sessionId = root.TryGetProperty("session_id", out var s) ? s.GetString() : null;

        _logger.LogInformation("Claude Code session started: model={Model}, session={Session}",
            model, sessionId);

        await ws.SendJsonAsync(new
        {
            type = "claude_code_start",
            model,
            session_id = sessionId,
        });
    }

    // ── Streaming content blocks (content_block_start/delta/stop) ────────
    //
    // Эти события приходят ДО "assistant" сообщения и позволяют стримить
    // текст и thinking в реальном времени (как в референсной имплементации).

    private void ProcessContentBlockStart(JsonElement root)
    {
        if (!root.TryGetProperty("index", out var indexProp))
            return;
        var index = indexProp.GetInt32();

        if (!root.TryGetProperty("content_block", out var block) ||
            !block.TryGetProperty("type", out var blockTypeProp))
            return;

        var blockType = blockTypeProp.GetString() ?? "";
        _streamingBlockTypes[index] = blockType;
        _streamingBuffers[index] = new StringBuilder();

        if (blockType == "tool_use")
        {
            var toolName = block.TryGetProperty("name", out var n)
                ? n.GetString() ?? "unknown" : "unknown";
            _streamingToolNames[index] = toolName;
        }
    }

    private async Task ProcessContentBlockDeltaAsync(JsonElement root, ChatChannel ws)
    {
        if (!root.TryGetProperty("index", out var indexProp) ||
            !root.TryGetProperty("delta", out var delta))
            return;

        var index = indexProp.GetInt32();
        if (!_streamingBuffers.TryGetValue(index, out var buffer))
            return;

        if (!delta.TryGetProperty("type", out var deltaTypeProp))
            return;

        var deltaType = deltaTypeProp.GetString();

        switch (deltaType)
        {
            // Thinking delta → отправить как thinking_start + token
            case "thinking_delta" when delta.TryGetProperty("thinking", out var t):
            {
                var text = t.GetString() ?? "";
                buffer.Append(text);

                // Для первого символа thinking — отправить thinking_start
                if (buffer.Length == text.Length)
                    await ws.SendObjectAsync(new { type = "thinking_start" });

                // Стримить thinking текст
                await ws.SendJsonAsync(new { type = "token", content = text });
                break;
            }

            // Text delta → отправить как token (основной текст ответа)
            case "text_delta" when delta.TryGetProperty("text", out var txt):
            {
                var text = txt.GetString() ?? "";
                buffer.Append(text);
                await ws.SendJsonAsync(new { type = "token", content = text });
                break;
            }

            // Tool input JSON delta → накапливать в буфер
            case "input_json_delta" when delta.TryGetProperty("partial_json", out var j):
            {
                buffer.Append(j.GetString());
                break;
            }
        }
    }

    private async Task ProcessContentBlockStopAsync(JsonElement root, ChatChannel ws)
    {
        if (!root.TryGetProperty("index", out var indexProp))
            return;
        var index = indexProp.GetInt32();

        if (!_streamingBuffers.TryGetValue(index, out var buffer) ||
            !_streamingBlockTypes.TryGetValue(index, out var blockType))
            return;

        switch (blockType)
        {
            case "thinking":
                await ws.SendObjectAsync(new { type = "thinking_end" });
                break;

            case "tool_use":
            {
                var toolName = _streamingToolNames.GetValueOrDefault(index, "unknown");
                var inputJson = buffer.ToString();

                // Parse tool input для human-readable описания
                JsonElement toolInput = default;
                try
                {
                    using var doc = JsonDocument.Parse(inputJson);
                    toolInput = doc.RootElement.Clone();
                }
                catch { /* partial/invalid JSON */ }

                var description = FormatToolDescription(toolName, toolInput);

                await ws.SendJsonAsync(new
                {
                    type = "claude_code_tool_use",
                    tool = toolName,
                    description,
                    input = inputJson,
                });
                break;
            }
        }

        // Cleanup
        _streamingBuffers.Remove(index);
        _streamingBlockTypes.Remove(index);
        _streamingToolNames.Remove(index);
    }

    // ── Full message events (assistant / user / result) ──────────────────

    private async Task ProcessAssistantEventAsync(JsonElement root, ChatChannel ws)
    {
        // "assistant" приходит ПОСЛЕ всех content_block_* событий для этого turn.
        // Streaming уже обработал текст. Здесь только tool_use summary если нужен.

        if (!root.TryGetProperty("message", out var message) ||
            !message.TryGetProperty("content", out var contentArray) ||
            contentArray.ValueKind != JsonValueKind.Array)
            return;

        foreach (var block in contentArray.EnumerateArray())
        {
            if (!block.TryGetProperty("type", out var bt))
                continue;

            // tool_use блоки (если streaming не обработал — fallback)
            if (bt.GetString() == "tool_use")
            {
                var toolName = block.TryGetProperty("name", out var n)
                    ? n.GetString() ?? "unknown" : "unknown";
                var toolInput = block.TryGetProperty("input", out var inp)
                    ? inp : default;

                await ws.SendJsonAsync(new
                {
                    type = "claude_code_tool_use",
                    tool = toolName,
                    description = FormatToolDescription(toolName, toolInput),
                });
            }
        }
    }

    private async Task ProcessUserEventAsync(JsonElement root, ChatChannel ws)
    {
        // "user" содержит tool_result — результат выполнения инструмента
        if (!root.TryGetProperty("message", out var message) ||
            !message.TryGetProperty("content", out var contentArray) ||
            contentArray.ValueKind != JsonValueKind.Array)
            return;

        foreach (var block in contentArray.EnumerateArray())
        {
            if (!block.TryGetProperty("type", out var bt) ||
                bt.GetString() != "tool_result")
                continue;

            var toolUseId = block.TryGetProperty("tool_use_id", out var tid)
                ? tid.GetString() : "";
            var content = ExtractToolResultContent(block);

            // Truncate для WebSocket (полный результат может быть огромным)
            const int maxLen = 500;
            var displayContent = content?.Length > maxLen
                ? content[..maxLen] + "..." : content;

            await ws.SendJsonAsync(new
            {
                type = "claude_code_tool_result",
                tool_use_id = toolUseId,
                content = displayContent,
            });
        }
    }

    private async Task<string> ProcessResultEventAsync(JsonElement root, ChatChannel ws)
    {
        var result = root.TryGetProperty("result", out var r) ? r.GetString() ?? "" : "";
        var isError = root.TryGetProperty("is_error", out var err) && err.GetBoolean();
        var numTurns = root.TryGetProperty("num_turns", out var nt) ? nt.GetInt32() : 0;
        var totalCost = root.TryGetProperty("total_cost_usd", out var cost)
            ? cost.GetDouble() : (double?)null;

        // Отправить информацию о стоимости
        await ws.SendJsonAsync(new
        {
            type = "claude_code_cost",
            cost_usd = totalCost,
            num_turns = numTurns,
            is_error = isError,
        });

        if (isError)
        {
            var subtype = root.TryGetProperty("subtype", out var sub)
                ? sub.GetString() : "unknown";
            await ws.SendJsonAsync(new
            {
                type = "error",
                message = $"Claude Code failed ({subtype}): {result}",
            });
        }

        _logger.LogInformation(
            "Claude Code finished: turns={Turns}, cost=${Cost:F4}, error={IsError}",
            numTurns, totalCost, isError);

        return result;
    }

    // ── Tool description formatting ──────────────────────────────────────
    //
    // Human-readable описания инструментов Claude Code для UI.
    // Адаптировано из AiCoreApi ClaudeCodeAgent.FormatToolDescription.

    private static string FormatToolDescription(string toolName, JsonElement input)
    {
        return toolName switch
        {
            "Read" => FormatWithPath(input, "file_path", "Reading"),
            "Write" => FormatWithPath(input, "file_path", "Writing"),
            "Edit" => FormatWithPath(input, "file_path", "Editing"),
            "Glob" => FormatWithProp(input, "pattern", "Searching files"),
            "Grep" => FormatWithProp(input, "pattern", "Searching code"),
            "Bash" => FormatBash(input),
            "WebFetch" => FormatWithProp(input, "url", "Fetching"),
            "WebSearch" => FormatWithProp(input, "query", "Searching"),
            _ when toolName.StartsWith("mcp__") => FormatMcpTool(toolName, input),
            _ => $"Using {toolName}",
        };
    }

    private static string FormatWithPath(JsonElement input, string prop, string verb)
    {
        var path = GetInputString(input, prop);
        return path != null ? $"{verb} {Path.GetFileName(path)}" : $"{verb} file";
    }

    private static string FormatWithProp(JsonElement input, string prop, string verb)
    {
        var value = GetInputString(input, prop);
        return value != null ? $"{verb}: {value}" : verb;
    }

    private static string FormatBash(JsonElement input)
    {
        var cmd = GetInputString(input, "command");
        if (cmd == null) return "Running command";
        return cmd.Length > 80 ? $"Running: {cmd[..80]}..." : $"Running: {cmd}";
    }

    private static string FormatMcpTool(string toolName, JsonElement input)
    {
        var parts = toolName.Split("__", 3);
        var server = parts.Length > 1 ? parts[1] : "unknown";
        var tool = parts.Length > 2 ? parts[2] : "unknown";
        return $"MCP {server}: {tool}";
    }

    private static string? GetInputString(JsonElement input, string property)
    {
        if (input.ValueKind != JsonValueKind.Object) return null;
        return input.TryGetProperty(property, out var p) && p.ValueKind == JsonValueKind.String
            ? p.GetString() : null;
    }

    private static string? ExtractToolResultContent(JsonElement block)
    {
        if (!block.TryGetProperty("content", out var content))
            return null;

        if (content.ValueKind == JsonValueKind.String)
            return content.GetString();

        if (content.ValueKind == JsonValueKind.Array)
        {
            var sb = new StringBuilder();
            foreach (var part in content.EnumerateArray())
            {
                if (part.TryGetProperty("type", out var t) &&
                    t.GetString() == "text" &&
                    part.TryGetProperty("text", out var text))
                    sb.Append(text.GetString());
            }
            return sb.Length > 0 ? sb.ToString() : null;
        }

        return null;
    }

    // ── Installation & credentials ───────────────────────────────────────

    private void EnsureInstalled(string version)
    {
        lock (InstallLock)
        {
            if (_installedVersion == version)
                return;

            var packageName = version == "latest"
                ? "@anthropic-ai/claude-code"
                : $"@anthropic-ai/claude-code@{version}";

            var process = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = "npm",
                    Arguments = $"install -g {packageName}",
                    RedirectStandardOutput = true,
                    RedirectStandardError = true,
                    UseShellExecute = false,
                    CreateNoWindow = true,
                }
            };

            process.Start();
            process.WaitForExit();

            if (process.ExitCode != 0 && FindClaudeBinary() == null)
            {
                var stderr = process.StandardError.ReadToEnd();
                throw new InvalidOperationException(
                    $"Failed to install Claude Code: {stderr}");
            }

            _installedVersion = version;
            _logger.LogInformation("Claude Code installed: {Version}", version);
        }
    }

    private const string CredentialsPath = "/root/.claude/.credentials.json";

    private async Task SetupCredentialsAsync(string? credentialsJson)
    {
        if (string.IsNullOrWhiteSpace(credentialsJson))
            return;

        var dir = Path.GetDirectoryName(CredentialsPath)!;
        if (!Directory.Exists(dir))
            Directory.CreateDirectory(dir);

        await File.WriteAllTextAsync(CredentialsPath, credentialsJson);
    }

    private static string? FindClaudeBinary()
    {
        var candidates = new[]
        {
            "/root/.local/bin/claude",
            "/usr/local/bin/claude",
            "/usr/bin/claude",
        };
        return candidates.FirstOrDefault(File.Exists)
            ?? throw new InvalidOperationException(
                $"Claude Code binary not found. Searched: {string.Join(", ", candidates)}");
    }

    // ── Helpers ──────────────────────────────────────────────────────────

    private static string BuildAppendSystemPrompt(ClaudeCodeRequest request)
    {
        var parts = new List<string>
        {
            "You are an AI assistant within the Agentic IT platform.",
            "The user is interacting with you through a web chat interface.",
        };

        if (request.ProjectContext is not null)
        {
            parts.Add("");
            parts.Add(request.ProjectContext);
        }

        return string.Join("\n", parts);
    }

    private async Task WriteAttachmentsToDirectoryAsync(
        ClaudeCodeRequest request, string workDir)
    {
        if (request.Attachments is null or { Count: 0 })
            return;

        foreach (var att in request.Attachments)
        {
            var name = att.GetValueOrDefault("name")?.ToString();
            var content = att.GetValueOrDefault("content")?.ToString();
            if (name is null || content is null) continue;

            var safeName = Path.GetFileName(name);
            if (string.IsNullOrWhiteSpace(safeName)) continue;

            await File.WriteAllTextAsync(
                Path.Combine(workDir, safeName), content);
        }
    }

    private static Dictionary<string, (long Size, DateTime LastWrite)> SnapshotDirectory(
        string path)
    {
        var snapshot = new Dictionary<string, (long, DateTime)>();
        if (!Directory.Exists(path)) return snapshot;

        foreach (var file in Directory.EnumerateFiles(path, "*", SearchOption.AllDirectories))
        {
            var info = new FileInfo(file);
            snapshot[file] = (info.Length, info.LastWriteTimeUtc);
        }
        return snapshot;
    }

    private static void TryDelete(string? path)
    {
        if (path == null) return;
        try { if (File.Exists(path)) File.Delete(path); } catch { /* best effort */ }
    }

    private static List<string> SplitCliArguments(string arguments)
    {
        var args = new List<string>();
        var current = new StringBuilder();
        var inQuotes = false;
        var quoteChar = '\0';

        foreach (var c in arguments)
        {
            if (inQuotes)
            {
                if (c == quoteChar) inQuotes = false;
                else current.Append(c);
            }
            else if (c is '"' or '\'')
            {
                inQuotes = true;
                quoteChar = c;
            }
            else if (char.IsWhiteSpace(c))
            {
                if (current.Length > 0)
                {
                    args.Add(current.ToString());
                    current.Clear();
                }
            }
            else
            {
                current.Append(c);
            }
        }

        if (current.Length > 0) args.Add(current.ToString());
        return args;
    }
}
```

### 6. Backend: ClaudeCodeRequest и IClaudeCodeRunner

**Файл: `backend/src/AgenticIT.Agent/ClaudeCode/ClaudeCodeRequest.cs`**

```csharp
using AgenticIT.Agent.Channel;

namespace AgenticIT.Agent.ClaudeCode;

public sealed class ClaudeCodeRequest
{
    public required string UserMessage { get; init; }
    public required List<Dictionary<string, object?>> ConversationHistory { get; init; }
    public required ChatChannel WebSocket { get; init; }
    public string? ProjectContext { get; init; }
    public string? ProjectId { get; init; }
    public string? ChatId { get; init; }
    public List<Dictionary<string, object?>>? Attachments { get; init; }
    public string? TenantId { get; init; }
    public string? UserId { get; init; }
}
```

**Файл: `backend/src/AgenticIT.Agent/ClaudeCode/IClaudeCodeRunner.cs`**

```csharp
namespace AgenticIT.Agent.ClaudeCode;

public interface IClaudeCodeRunner
{
    Task RunAsync(ClaudeCodeRequest request, CancellationToken ct = default);
}
```

### 7. Backend: конфигурация в AgenticITOptions

**Файл: `backend/src/AgenticIT.Infrastructure/Configuration/AgenticITOptions.cs`**

Добавить свойства:

```csharp
// Claude Code integration
public string? ClaudeCodeApiKey { get; set; }
public string? ClaudeCodeCredentialsJson { get; set; }
public string? ClaudeCodeVersion { get; set; }
public int ClaudeCodeMaxTurns { get; set; } = 25;
public int ClaudeCodeTimeoutMinutes { get; set; } = 10;
public string? ClaudeCodeCliArgs { get; set; }
public string? ClaudeCodeMcpConfig { get; set; }
```

### 8. Backend: DI Registration

**Файл: `backend/src/AgenticIT.Api/Program.cs`**

```csharp
builder.Services.AddSingleton<IClaudeCodeRunner, ClaudeCodeRunner>();
```

---

## Stream-JSON протокол Claude Code

### Типы сообщений и трансляция в WebSocket

```
Claude Code stdout (NDJSON)         →    WebSocket (ChatChannel)
─────────────────────────────────        ──────────────────────────
{"type":"system",...}                →    {"type":"claude_code_start"}
{"type":"content_block_start",            (internal state only)
  "content_block":{"type":"thinking"}}
{"type":"content_block_delta",       →    {"type":"thinking_start"} (first delta)
  "delta":{"type":"thinking_delta",       {"type":"token","content":"..."}
           "thinking":"..."}}
{"type":"content_block_stop"}        →    {"type":"thinking_end"}

{"type":"content_block_start",            (internal state only)
  "content_block":{"type":"text"}}
{"type":"content_block_delta",       →    {"type":"token","content":"..."}
  "delta":{"type":"text_delta",
           "text":"..."}}
{"type":"content_block_stop"}             (nothing)

{"type":"content_block_start",            (internal state, accumulate input JSON)
  "content_block":{"type":"tool_use",
                   "name":"Bash"}}
{"type":"content_block_delta",
  "delta":{"type":"input_json_delta"}}
{"type":"content_block_stop"}        →    {"type":"claude_code_tool_use",
                                           "tool":"Bash",
                                           "description":"Running: git status"}

{"type":"assistant",...}                  (fallback only if streaming missed)
{"type":"user",...}                  →    {"type":"claude_code_tool_result"}
{"type":"result",...}                →    {"type":"claude_code_cost"} + {"type":"done"}
```

### Ключевые детали из референса

1. **`ArgumentList` вместо `Arguments`** — защита от prompt injection:
   ```csharp
   process.StartInfo.ArgumentList.Add("-p");
   process.StartInfo.ArgumentList.Add(prompt); // безопасно, даже если prompt содержит кавычки
   ```

2. **System prompt через файл** — для больших промптов:
   ```csharp
   process.StartInfo.ArgumentList.Add("--append-system-prompt-file");
   process.StartInfo.ArgumentList.Add(tempFilePath);
   ```

3. **Stdin закрывается сразу** — предотвращает зависание:
   ```csharp
   process.StandardInput.Close();
   ```

4. **Stderr читается в background Task** — предотвращает deadlock:
   ```csharp
   var stderrTask = Task.Run(async () => { while (await process.StandardError.ReadLineAsync()...) });
   ```

5. **Credentials file** (`/root/.claude/.credentials.json`) — OAuth токен, который Claude Code может обновить. После выполнения — синхронизировать обратно.

6. **Temp working directory** — каждый запрос в изолированной директории. Артефакты проекта копируются туда, изменённые файлы собираются обратно.

---

## Аутентификация

### Два варианта

| Метод | Как настроить | Биллинг |
|-------|--------------|---------|
| **API Key** | `ANTHROPIC_API_KEY=sk-ant-api03-...` | Pay-per-token |
| **OAuth (credentials.json)** | Записать файл в `/root/.claude/.credentials.json` | Subscription (Max/Pro) |

### credentials.json формат

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-...",
    "refreshToken": "...",
    "expiresAt": "2026-04-01T00:00:00Z"
  }
}
```

Claude Code автоматически обновляет `accessToken` через `refreshToken`. После выполнения нужно прочитать обновлённый файл и сохранить обратно (как в референсе — `SyncCredentialsBackAsync`).

### Ограничения

- С февраля 2026 Anthropic **запретила** OAuth в third-party apps / Agent SDK
- **CLI подход** (`claude -p`) технически работает с OAuth credentials.json
- Для серверного деплоя **самый надёжный вариант — API Key** (`ANTHROPIC_API_KEY`)
- OAuth credentials.json — серая зона, работает но может быть заблокировано

---

## Working Directory и артефакты проекта

Из референсной имплементации — паттерн работы с файлами:

```
1. Создать temp dir: /tmp/claude/{guid}
2. Скопировать артефакты проекта → temp dir
3. Записать аттачменты пользователя → temp dir
4. Записать CLAUDE.md (project context) → temp dir
5. Snapshot директории (файлы + размеры + даты)
6. Запустить Claude Code (cwd = temp dir)
7. Удалить CLAUDE.md
8. Сравнить с snapshot: собрать новые/изменённые файлы
9. Загрузить изменённые файлы обратно в артефакты проекта
10. Удалить temp dir
```

Этот паттерн позволяет Claude Code:
- Читать существующие файлы проекта
- Создавать новые файлы
- Модифицировать файлы
- Результаты автоматически синхронизируются обратно в проект

---

## Структура новых файлов

```
backend/src/AgenticIT.Agent/ClaudeCode/
├── IClaudeCodeRunner.cs     # Интерфейс
├── ClaudeCodeRunner.cs      # Основная реализация (Process + NDJSON parsing)
└── ClaudeCodeRequest.cs     # Request DTO
```

---

## Порядок реализации

### Этап 1 — MVP (streaming text)
1. Frontend: "Ultra (Claude Code)" в dropdown
2. Backend: `ChatWebSocketHandler` распознаёт `"claude-code"`
3. Backend: `SmartOrchestrator` bypass planner
4. Backend: `ClaudeCodeRunner` — запуск процесса, парсинг stream-json
5. Трансляция `content_block_delta` (text_delta) → `token` WS events
6. `result` → `done` + `claude_code_cost`
7. DI registration
8. `ANTHROPIC_API_KEY` env var

### Этап 2 — Tool visibility
1. Трансляция `content_block_stop` (tool_use) → `claude_code_tool_use` с описанием
2. Трансляция `user` (tool_result) → `claude_code_tool_result`
3. Frontend: показ tool calls в UI (AgentStatusBar или аналог)
4. Thinking streaming (thinking_start/end + token)

### Этап 3 — Project integration
1. Копирование артефактов проекта в working directory
2. Snapshot + diff для сбора изменений
3. Загрузка изменённых файлов обратно в артефакты
4. CLAUDE.md injection с project context

### Этап 4 — Advanced
1. Session resume (`--resume <session_id>`)
2. OAuth credentials refresh (sync back)
3. MCP config injection
4. Configurable timeout / max-turns / max-budget

---

## Риски и ограничения

| Риск | Митигация |
|------|-----------|
| Claude Code не установлен | `EnsureInstalled()` с `npm install -g` при первом вызове |
| Node.js не установлен | Docker image с Node.js; проверка при старте |
| Стоимость (pay-per-token) | `--max-turns`, timeout, мониторинг `total_cost_usd` |
| OAuth запрещён для third-party | Основной путь — API Key; OAuth как опция |
| Timeout (долгие агентные циклы) | Configurable timeout + `process.Kill(true)` |
| Prompt injection через user input | `ArgumentList` вместо string concatenation |
| Большой stdout (много tool calls) | Truncation tool_result до 500 chars в WS |
| Параллельные запросы | Каждый в отдельном процессе + temp dir |
