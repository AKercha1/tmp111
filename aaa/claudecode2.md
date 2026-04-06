# Claude Code Integration — Gap Analysis & Recommendations

## Текущее состояние

Claude Code интеграция работает как отдельный subprocess: `ClaudeCodeRunner` запускает `claude -p <prompt> --output-format stream-json`, читает NDJSON-стрим и пересылает события в WebSocket. При выборе модели `claude-code` в `SmartOrchestrator` вся стандартная логика **полностью обходится** (строка 80–98 в `SmartOrchestrator.cs`).

---

## 1. Артефакты (save_note, save_script, generate_document)

### Что есть в обычном флоу

`SingleAgentLoop` (строка 157–178) перехватывает вызовы artifact-тулов (`save_note`, `save_script`, `generate_document`), исполняет их через `ArtifactService`, сохраняет файлы на диск + запись в БД, и отправляет `artifact_saved` событие в WebSocket.

### Что есть в Claude Code

**Ничего.** Claude Code не имеет доступа к artifact-тулам платформы. Он может создать файлы в своём temp-каталоге, но:
- Файлы удаляются при `Directory.Delete(workDir, true)` в `RunAsync` finally-блоке
- Нет записей в БД, нет `artifact_saved` событий
- Пользователь не видит созданные файлы

### Рекомендация

**Подход A — MCP-сервер для артефактов (рекомендуется):**

Создать MCP-сервер `agenticit-artifacts` с тулами:
```json
{
  "tools": [
    { "name": "save_note",     "description": "Save a markdown note to the project" },
    { "name": "save_script",   "description": "Save a script (bash/powershell/python)" },
    { "name": "list_artifacts", "description": "List project artifacts" }
  ]
}
```

Передать его через `--mcp-config` в `ClaudeCodeRunner`. MCP-сервер вызывает тот же `ArtifactService` на бэкенде.

**Подход B — Post-processing:**

После завершения Claude Code сканировать `workDir` на наличие созданных файлов (перед `Directory.Delete`), автоматически сохранять их как артефакты проекта.

**Подход C — CLAUDE.md инструкции:**

Добавить в append system prompt инструкцию: "Для сохранения артефактов используй формат `<artifact name="..." type="note|script">...</artifact>`, я обработаю их после ответа." Парсить structured output и сохранять через `ArtifactService`.

**Приоритет: Высокий** — без артефактов Claude Code не может создавать полезные файлы в контексте проекта.

---

## 2. Goal / Plan / Instructions (Memory Service)

### Что есть в обычном флоу

`SmartOrchestrator` (строка 196–221) после каждого ответа запускает fire-and-forget `MemoryService.UpdateAfterOrchestratorTurnAsync()`:
- Обновляет project memory (summary, key facts)
- Обновляет user memory (preferences, recurring topics)
- Генерирует/обновляет план (goals, todos)
- Отслеживает выполнение задач (auto-completion)
- Выполняет deep review каждые 5 ходов

### Что есть в Claude Code

**Ничего.** В Claude Code ветке (строка 80–98) `return` происходит ДО блока memory update (строка 196–221). Каждая сессия Claude Code полностью изолирована — нет накопления знаний.

### Рекомендация

Добавить memory update после `_claudeCodeRunner.RunAsync()` — точно так же, как для обычного флоу:

```csharp
// В SmartOrchestrator.RunAsync(), после строки 95:
if (req.ProjectId is not null)
{
    var responseText = ExtractLastAssistantText(req.ConversationHistory);
    var capturedProjectId = req.ProjectId;
    // ... (fire-and-forget memory update, как в строках 196-221)
}
```

Также: включать project memory (goals, summaries) в `ProjectContext`, который уже передаётся в Claude Code через `BuildProjectContextAsync`. Нужно убедиться что `BuildProjectContextAsync` включает memory.

**Приоритет: Средний** — без этого Claude Code "забывает" всё между ходами.

---

## 3. MCP (vBox Portal Tools)

### Что есть в обычном флоу

`IMcpClientManager` динамически загружает тулы из vBox Portal при `MCP_ENABLED=true`:
- Кеширует tool schemas (TTL 300s)
- Выполняет тулы через SSE-транспорт с Bearer token auth
- Нормализует и валидирует параметры (`McpParamNormalizer`, `McpParamValidator`)
- Передаёт subscription context

### Что есть в Claude Code

**Частично.** `ClaudeCodeRunner` поддерживает `--mcp-config` через `AgenticITOptions.ClaudeCodeMcpConfig`, но:
- Конфиг статический (JSON-строка в appsettings), не динамический
- Нет интеграции с vBox OAuth token management
- Нет per-user session pooling
- Claude Code использует свой MCP-клиент, не AgenticIT'шный
- Нет parameter normalization/validation

### Рекомендация

**Вариант 1 — Динамический MCP config (рекомендуется):**

Вместо статического `ClaudeCodeMcpConfig` из appsettings, генерировать MCP config динамически перед запуском:

```csharp
// В ClaudeCodeRunner.RunAsync:
var mcpConfig = await BuildDynamicMcpConfig(request.UserId, request.TenantId);
// Записать во временный файл и передать через --mcp-config
```

`BuildDynamicMcpConfig` берёт URL и свежий auth token из `IMcpClientManager`, формирует JSON:
```json
{
  "mcpServers": {
    "vbox": {
      "type": "sse",
      "url": "https://portal.vbox.dev/mcp/sse",
      "headers": { "Authorization": "Bearer <fresh-token>" }
    }
  }
}
```

**Вариант 2 — Proxy MCP server:**

Создать локальный MCP-прокси, который Claude Code подключает через stdio, а прокси вызывает `IMcpClientManager` на бэкенде. Это даёт полный контроль над auth, normalization, validation.

**Приоритет: Высокий** — без MCP Claude Code не может работать с Azure через vBox.

---

## 4. Azure Tools (70+ тулов)

### Что есть в обычном флоу

`ToolRegistry` определяет 70+ тулов по категориям:
- **Cost** (7): orphaned disks, idle VMs, cost breakdown, savings
- **Security** (7): open ports, public storage, Key Vault, Defender
- **Compliance** (6): missing tags, policy violations, RBAC
- **Reliability** (3): backup coverage, Site Recovery
- **Observability** (4): monitoring, diagnostics, alerts
- **Identity** (3): risky service principals, stale users
- **M365** (4): Secure Score, risky users, Teams/Exchange
- **Migration** (3): readiness, on-prem servers, TCO
- **Write/Destructive** (6): delete, resize, remediate, apply_tag
- **Inventory** (9): resources, VMs, VNets, storage

Все исполняются через `ToolExecutor` → MCP (vBox Portal).

### Что есть в Claude Code

**Ничего напрямую.** Azure тулы доступны только через MCP, который частично поддерживается (см. пункт 3).

### Рекомендация

Решается через пункт 3 (MCP интеграция). Когда Claude Code подключен к vBox MCP, все Azure тулы становятся доступны автоматически.

Дополнительно: включить skill `tool-selection` в system prompt Claude Code, чтобы он знал какие тулы доступны и как их использовать:

```csharp
private static string BuildAppendSystemPrompt(ClaudeCodeRequest request)
{
    var parts = new List<string>
    {
        "You are an AI assistant within the Agentic IT platform.",
        SkillLoader.LoadSkillBodySafe("core-advisor"),
        SkillLoader.LoadSkillBodySafe("tool-selection"),
    };
    if (hasMcp) parts.Add(SkillLoader.LoadSkillBodySafe("vbox-tools"));
    if (hasProject) parts.Add(SkillLoader.LoadSkillBodySafe("project-guidance"));
    // ...
}
```

**Приоритет: Высокий** — это основная функциональность платформы.

---

## 5. Specialist Agents (параллельное выполнение)

### Что есть в обычном флоу

`SmartOrchestrator` использует planner LLM для маршрутизации:
- `direct` → один `SingleAgentLoop`
- `specialist` → N агентов параллельно (cost, security, compliance, etc.) → synthesis

10 встроенных специалистов в `SpecialistRegistry`, каждый с:
- Выделенным набором тулов
- Своим system prompt из embedded skill
- Цветом для UI

### Что есть в Claude Code

**Ничего.** Claude Code всегда работает как единый агент. Нет planner, нет parallel execution, нет synthesis.

### Рекомендация

**Это не нужно реализовывать.** Claude Code — это альтернативный движок, а не замена полного флоу. Пользователь выбирает "Ultra (Claude Code)" когда ему нужна мощь Claude Code (длинный контекст, agentic coding, filesystem access). Для мультиагентного анализа Azure инфраструктуры остаётся стандартный флоу.

**Приоритет: Низкий** — намеренно другая архитектура.

---

## 6. Approval Workflow (destructive tools)

### Что есть в обычном флоу

`SingleAgentLoop` (строка 180–212):
- 6 destructive тулов (`delete_resource`, `resize_vm`, `remediate_nsg_rule`, `enable_storage_https`, `apply_tag`, `assign_policy`)
- Отправляет `approval_required` событие
- Ждёт ответ пользователя через `IApprovalManager`
- Логирует `approved`/`denied`

### Что есть в Claude Code

**Ничего.** Claude Code исполняет все тулы без подтверждения.

### Рекомендация

**Вариант 1 — Claude Code built-in permissions:**

Claude Code имеет встроенную систему permissions. Через `--allowedTools` и `--disallowedTools` CLI flags можно ограничить доступ:
```
--disallowedTools "mcp__vbox__delete_resource,mcp__vbox__resize_vm,..."
```

Но это просто блокирует, а не запрашивает approval.

**Вариант 2 — MCP Approval Proxy (рекомендуется):**

Обернуть destructive MCP тулы в прокси, который:
1. Получает вызов от Claude Code
2. Отправляет `approval_required` через WebSocket
3. Ждёт ответа пользователя
4. Выполняет или отклоняет

Это требует MCP-прокси из пункта 3, Вариант 2.

**Вариант 3 — Post-result injection:**

Если Claude Code вызывает destructive тул через MCP, бэкенд перехватывает вызов, возвращает "Awaiting approval..." и уведомляет фронтенд.

**Приоритет: Средний** — важно для production safety, но для MVP можно ограничить destructive tools через `--disallowedTools`.

---

## 7. Web Search

### Что есть в обычном флоу

`WebSearchExecutor` с domain filtering, result truncation, audit logging. Отправляет `tool_result` с `is_web_search=true` и sources metadata.

### Что есть в Claude Code

**Встроенно.** Claude Code имеет `WebSearch` и `WebFetch` тулы из коробки. Они работают автономно.

### Рекомендация

**Достаточно текущей реализации.** Claude Code's built-in web search работает хорошо. Единственное улучшение: передавать `claude_code_tool_use` событие с human-readable description (уже реализовано в `FormatToolDescription`).

**Приоритет: Низкий** — уже работает.

---

## 8. Schedule Tools

### Что есть в обычном флоу

3 тула: `schedule_action`, `list_scheduled_actions`, `cancel_scheduled_action`. Хранятся в `IScheduledJobRepository`. Поддерживают cron и ISO datetime.

### Что есть в Claude Code

**Ничего.**

### Рекомендация

Добавить через MCP-сервер артефактов (пункт 1):
```json
{
  "tools": [
    { "name": "schedule_action", "description": "Schedule a one-time or recurring action" },
    { "name": "list_scheduled_actions", "description": "List scheduled actions" },
    { "name": "cancel_scheduled_action", "description": "Cancel a scheduled action" }
  ]
}
```

**Приоритет: Низкий** — редко используемая функция.

---

## 9. System Prompt (Skills Composition)

### Что есть в обычном флоу

`SkillLoader.BuildSystemPrompt()` собирает из 5+ embedded markdown skills:
- `core-advisor` — базовый промпт, идентичность "Agentic IT"
- `tool-selection` — гайд по выбору тулов
- `vbox-tools` — условно, если MCP enabled
- `project-guidance` — условно, если projectId != null
- `document-generation` — генерация документов
- + `projectContext` + `extraSystem`

### Что есть в Claude Code

**Минимально.** `BuildAppendSystemPrompt` (строка 674–689) содержит только:
```
"You are an AI assistant within the Agentic IT platform."
"The user is interacting with you through a web chat interface."
+ projectContext
```

Нет core-advisor, нет tool-selection, нет vbox-tools, нет project-guidance, нет document-generation.

### Рекомендация

Обогатить `BuildAppendSystemPrompt` скиллами:

```csharp
private string BuildAppendSystemPrompt(ClaudeCodeRequest request, bool hasMcp, bool hasProject)
{
    var parts = new List<string>
    {
        SkillLoader.LoadSkillBodySafe("core-advisor"),
        SkillLoader.LoadSkillBodySafe("tool-selection"),
    };

    if (hasMcp) parts.Add(SkillLoader.LoadSkillBodySafe("vbox-tools"));
    if (hasProject) parts.Add(SkillLoader.LoadSkillBodySafe("project-guidance"));
    parts.Add(SkillLoader.LoadSkillBodySafe("document-generation"));

    if (request.ProjectContext is not null)
        parts.Add(request.ProjectContext);

    return string.Join("\n\n", parts.Where(p => !string.IsNullOrEmpty(p)));
}
```

Для этого `ClaudeCodeRunner` нужно будет получать `AgenticITOptions.McpEnabled` и `request.ProjectId != null` флаги.

**Приоритет: Высокий** — без этого Claude Code не знает контекст платформы.

---

## 10. Extended Thinking

### Что есть в обычном флоу

`SingleAgentLoop` (строка 314–317) устанавливает `thinking.budget_tokens = 3000` при `useExtendedThinking=true`.

### Что есть в Claude Code

**Частично.** Streaming `thinking_delta` уже обрабатывается в `ProcessContentBlockDeltaAsync` (строка 330–338) — отправляет `thinking_start` и стримит thinking tokens. Но:
- Нет CLI-флага `--thinking` для активации
- Нет UI-чекбокса (скрыт для claude-code модели)

### Рекомендация

Claude Code Opus включает extended thinking по умолчанию. Для Sonnet можно добавить CLI флаг если нужно. Текущая streaming обработка уже корректна.

**Приоритет: Низкий** — уже работает для Opus.

---

## 11. Audit Logging

### Что есть в обычном флоу

`IAuditLogger.LogToolCallAsync()` для каждого вызова тула: tool name, input, output, approval status, user ID, chat ID.

### Что есть в Claude Code

**Ничего.** Tool calls не логируются в audit system.

### Рекомендация

Логировать минимум:
- Start/end Claude Code сессии (уже есть через `_logger.LogInformation`)
- Каждый `claude_code_tool_use` event → `_auditLogger.LogToolCallAsync`
- Cost tracking из `claude_code_cost` event

```csharp
// В ProcessContentBlockStopAsync, case "tool_use":
await _auditLogger.LogToolCallAsync(chatId, toolName, inputDict, null, userId: userId);
```

Для этого нужно передать `IAuditLogger` и context (chatId, userId) в streaming методы.

**Приоритет: Средний** — нужно для compliance.

---

## 12. Conversation History & Attachments

### Что есть в обычном флоу

- Base64 inline images → `image` content block
- Text attachments с truncation at 32KB → `<attachment>` block
- Binary files → info text
- Deduplication of tool IDs

### Что есть в Claude Code

- Attachments записываются как файлы в temp dir
- Нет inline images в conversation
- Claude Code читает файлы через свой Read tool

### Рекомендация

**Текущий подход достаточен.** Claude Code's filesystem access компенсирует отсутствие inline embedding. Файлы в workDir доступны через Read/Glob/Grep.

Улучшение: для image attachments добавить `--image` CLI flag если он поддерживается, или конвертировать в base64 и вставить в промпт.

**Приоритет: Низкий** — работает через filesystem.

---

## Subscription / Org Context

### Что есть в обычном флоу

`SmartOrchestrator` (строка 64–77):
- Резолвит Azure subscriptions по `vbox_organization_id`
- Передаёт `projectSubscriptions` и `projectOrgId` в `AgentLoopRequest`
- Specialist agents получают org/subscription context

### Что есть в Claude Code

**Ничего.** `ClaudeCodeRequest` не содержит `ProjectOrgId` или `ProjectSubscriptions`.

### Рекомендация

Включить org/subscription info в `ProjectContext` (уже содержит `BuildProjectContextAsync` output). Дополнительно добавить subscription list в system prompt:

```csharp
if (projectSubscriptions.Count > 0)
{
    var subList = string.Join(", ", projectSubscriptions.Select(s => $"{s.Id} ({s.Name})"));
    mergedCcContext += $"\n\nAzure subscriptions: [{subList}]";
}
if (projectOrgId is not null)
    mergedCcContext += $"\nvBox Organization ID: {projectOrgId}";
```

**Приоритет: Средний** — нужно для Azure tool context.

---

## Сводная таблица

| Фича | Обычный флоу | Claude Code | Статус | Приоритет | Рекомендация |
|-------|-------------|-------------|--------|-----------|-------------|
| System Prompt (Skills) | 5+ skills | 2 строки | **MISSING** | Высокий | Добавить SkillLoader в BuildAppendSystemPrompt |
| Артефакты | save_note, save_script, generate_doc | Нет | **MISSING** | Высокий | MCP-сервер / post-processing / structured output |
| MCP Tools (vBox) | Динамические, с auth | Статический конфиг | **PARTIAL** | Высокий | Динамический MCP config с fresh token |
| Azure Tools (70+) | Через ToolExecutor+MCP | Нет | **MISSING** | Высокий | Решается через MCP интеграцию |
| Memory Service | Auto-update каждый ход | Нет | **MISSING** | Средний | Добавить memory update после RunAsync |
| Approval Workflow | 6 destructive tools | Нет | **MISSING** | Средний | --disallowedTools или MCP proxy |
| Audit Logging | Все tool calls | Нет | **MISSING** | Средний | Логировать claude_code_tool_use |
| Org/Subscriptions | OrgId + subs resolved | Нет | **MISSING** | Средний | Включить в ProjectContext |
| Specialist Agents | 10 parallel agents | Нет | N/A | Низкий | Не нужно — другая архитектура |
| Web Search | WebSearchExecutor | Built-in | **OK** | Низкий | Достаточно |
| Extended Thinking | budget_tokens=3000 | Streaming OK | **OK** | Низкий | Opus включает по умолчанию |
| Attachments | Inline base64 + text | Files in workDir | **OK** | Низкий | Достаточно |
| Schedule Tools | 3 tools | Нет | **MISSING** | Низкий | Через MCP-сервер |

---

## План реализации

### Этап 1 — System Prompt & Memory (1-2 дня)

1. Обогатить `BuildAppendSystemPrompt` скиллами из `SkillLoader`
2. Добавить org/subscription info в context
3. Добавить memory update после `_claudeCodeRunner.RunAsync()` в SmartOrchestrator
4. Передать `hasMcp` и `hasProject` флаги в ClaudeCodeRunner

### Этап 2 — Динамический MCP (2-3 дня)

1. Создать `BuildDynamicMcpConfig()` в ClaudeCodeRunner
2. Интегрировать с `IMcpClientManager` для получения URL и fresh token
3. Генерировать temp MCP config JSON перед каждым запуском
4. Добавить `--disallowedTools` для destructive tools

### Этап 3 — Артефакты (2-3 дня)

1. **Вариант A (MCP-сервер):** Создать `AgenticITMcpServer` (stdio transport) с тулами save_note, save_script, list_artifacts
2. **Вариант B (Post-processing):** Сканировать workDir, парсить structured output, сохранять через ArtifactService
3. Отправлять `artifact_saved` события в WebSocket

### Этап 4 — Audit & Compliance (1 день)

1. Передать `IAuditLogger` в `ClaudeCodeRunner`
2. Логировать каждый `claude_code_tool_use` через `_auditLogger.LogToolCallAsync`
3. Логировать cost/session info

---

## Архитектурная диаграмма (целевое состояние)

```
User → WebSocket → ChatWebSocketHandler
                         │
                    SmartOrchestrator
                    ┌──────┴──────┐
                    │             │
              model=claude-code   model=sonnet/opus/haiku
                    │             │
            ClaudeCodeRunner   Planner LLM
                    │          ┌──┴──┐
                    │       Direct  Specialist
                    │          │       │
               claude CLI   SingleAgentLoop  SpecialistRunner
                    │          │       │
              ┌─────┴────┐   ToolExecutor  (parallel)
              │          │     │
        Built-in    MCP config  ├─ MCP Tools (vBox)
        tools       (dynamic)   ├─ Web Search
        (Read,       │          ├─ Artifacts
        Write,      vBox MCP    ├─ Schedule
        Bash,       Server      └─ Azure (70+)
        WebSearch)    │
              │      (same tools!)
        AgenticIT
        MCP Server
        (artifacts,
        schedule)
              │
         ArtifactService
         ScheduleRepo
              │
           Database
```
