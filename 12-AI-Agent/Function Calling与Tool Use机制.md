---
tags: [android, 面试, ai-agent, function-calling, A]
difficulty: A
frequency: high
---

# Function Calling 与 Tool Use 机制

## 问题

> LLM 的 Function Calling 是如何工作的？如何定义和编排多个工具？MCP 协议是什么？如何在 Android App 中集成 Tool Use？

## 核心答案

Function Calling 让 LLM 输出结构化的函数调用请求（而非自然语言），由应用层执行后将结果返回 LLM。工具通过 JSON Schema 定义参数，LLM 根据用户意图选择合适的工具并填充参数。MCP（Model Context Protocol）是 Anthropic 提出的标准化协议，统一了 Tool/Resource/Prompt 的定义和通信方式。移动端通过 API 调用云端 LLM 实现 Tool Use。

## 深入解析

### 原理层

**Function Calling 流程：**
```
1. 定义工具 (JSON Schema)
2. 用户发送消息 + 工具列表 → LLM
3. LLM 决定是否调用工具，输出:
   { "name": "get_weather", "arguments": {"city": "北京"} }
4. 应用层执行函数，获取结果
5. 将结果作为 tool_result 发回 LLM
6. LLM 基于结果生成最终回答（或继续调用其他工具）
```

**工具定义（OpenAI 格式）：**
```json
{
  "type": "function",
  "function": {
    "name": "search_products",
    "description": "搜索商品信息，当用户询问商品价格、库存时使用",
    "parameters": {
      "type": "object",
      "properties": {
        "query": { "type": "string", "description": "搜索关键词" },
        "category": { "type": "string", "enum": ["电子", "服装", "食品"] },
        "max_price": { "type": "number", "description": "最高价格" }
      },
      "required": ["query"]
    }
  }
}
```

**Claude Tool Use 格式：**
```json
{
  "name": "search_products",
  "description": "搜索商品信息",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": { "type": "string" },
      "category": { "type": "string" }
    },
    "required": ["query"]
  }
}
```

**MCP（Model Context Protocol）：**
```
MCP 架构:
┌─────────┐     stdio/SSE     ┌────────────┐
│ MCP Host │ ←──────────────→ │ MCP Server │
│ (Claude) │                   │ (工具提供者) │
└─────────┘                   └────────────┘

三大原语:
- Tools: 可执行的函数（搜索、计算、API 调用）
- Resources: 可读取的数据源（文件、数据库）
- Prompts: 预定义的 Prompt 模板

优势: 标准化协议，一个 Server 可被多个 Host 复用
```

**多工具编排模式：**
```python
# 并行调用（独立工具）
response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    tools=tools,
    messages=[{"role": "user", "content": "查北京和上海的天气"}]
)
# LLM 可能返回两个并行的 tool_use:
# [tool_use: get_weather(city="北京"), tool_use: get_weather(city="上海")]

# 顺序调用（依赖关系）
# LLM 自动识别依赖：先搜索航班 → 再查价格 → 最后预订
```

### 实战层

**Android 中集成 Tool Use：**
```kotlin
// 定义本地工具
data class ToolDefinition(
    val name: String,
    val description: String,
    val parameters: JsonSchema,
    val handler: suspend (JsonObject) -> String
)

val tools = listOf(
    ToolDefinition(
        name = "get_device_info",
        description = "获取设备信息（电量、存储、网络状态）",
        parameters = jsonSchema { },
        handler = { _ ->
            json {
                "battery" to batteryManager.level
                "storage_free" to storageManager.freeSpace
                "network" to connectivityManager.activeNetwork?.type
            }.toString()
        }
    ),
    ToolDefinition(
        name = "search_contacts",
        description = "搜索通讯录联系人",
        parameters = jsonSchema { "name" to StringType },
        handler = { args -> contactsRepository.search(args["name"].asString) }
    )
)

// 调用 LLM API 并处理 tool_use
suspend fun chat(userMessage: String): String {
    var messages = listOf(Message.user(userMessage))
    while (true) {
        val response = anthropicClient.messages(messages, tools)
        if (response.stopReason == "end_turn") return response.text
        
        // 处理工具调用
        val toolResults = response.toolUses.map { toolUse ->
            val handler = tools.find { it.name == toolUse.name }!!.handler
            val result = handler(toolUse.input)
            ToolResult(toolUse.id, result)
        }
        messages = messages + response.asAssistantMessage() + toolResults.asUserMessage()
    }
}
```

**工具设计最佳实践：**
- 工具描述要精确（LLM 靠描述决定何时调用）
- 参数用 enum 约束可选值
- 返回结构化数据（JSON），不要返回大段文本
- 敏感操作加确认步骤（"确定要删除吗？"）
- 设置工具调用次数上限，防止无限循环

### 延伸问题

- [[LLM Agent架构设计-ReAct与Plan-and-Execute]]
- [[Retrofit动态代理原理]]
- [[AI Agent在移动端的落地]]

## 记忆锚点

Function Calling = LLM 输出"我要调用什么函数+什么参数"，应用层执行后把结果喂回去。MCP 是标准化的工具协议，让工具可以跨 Host 复用。
