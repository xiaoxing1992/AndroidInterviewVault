---
tags: [android, 面试, ai-agent, 移动端, A]
difficulty: A
frequency: high
---

# AI Agent 在移动端的落地

## 问题

> AI Agent 如何在 Android/移动端落地？端侧 LLM 和云端 LLM 如何选择？Streaming 响应如何与 UI 结合？有哪些隐私和性能挑战？

## 核心答案

移动端 AI Agent 主要有三种架构：纯云端（API 调用，功能强但依赖网络）、纯端侧（本地模型，隐私好但能力有限）、混合架构（端侧处理简单任务+云端处理复杂任务）。Streaming 通过 SSE/WebSocket 实现逐 token 显示。关键挑战是延迟、成本、隐私和离线能力的平衡。

## 深入解析

### 原理层

**三种架构模式：**
```
1. 纯云端 Agent
   App → API Call → Cloud LLM → Stream Response → App UI
   优点: 能力强、模型可更新
   缺点: 依赖网络、延迟高、隐私风险

2. 纯端侧 Agent
   App → Local LLM (Gemma/Phi) → Response → App UI
   优点: 离线可用、隐私安全、无 API 成本
   缺点: 模型小能力弱、占用设备资源

3. 混合架构（推荐）
   App → Router → 简单任务 → 端侧小模型
                → 复杂任务 → 云端大模型
   优点: 平衡性能/成本/隐私
```

**Android 端侧 LLM 部署：**
```kotlin
// 使用 MediaPipe LLM Inference API
val llmInference = LlmInference.createFromOptions(
    context,
    LlmInference.LlmInferenceOptions.builder()
        .setModelPath("/data/local/tmp/gemma-2b-it-q4.bin")
        .setMaxTokens(1024)
        .setTemperature(0.7f)
        .build()
)

// 推理
val response = llmInference.generateResponse("Explain coroutines in Kotlin")

// 流式输出
llmInference.generateResponseAsync(prompt) { partialResult, done ->
    runOnUiThread { textView.append(partialResult) }
}
```

**云端 Streaming 集成：**
```kotlin
// OkHttp SSE 接收流式响应
fun streamChat(messages: List<Message>): Flow<String> = callbackFlow {
    val request = Request.Builder()
        .url("https://api.anthropic.com/v1/messages")
        .post(buildRequestBody(messages))
        .header("Accept", "text/event-stream")
        .build()

    val eventSource = EventSources.createFactory(okHttpClient)
        .newEventSource(request, object : EventSourceListener() {
            override fun onEvent(es: EventSource, id: String?, type: String?, data: String) {
                val event = json.decodeFromString<StreamEvent>(data)
                if (event.type == "content_block_delta") {
                    trySend(event.delta.text)
                }
            }
            override fun onClosed(es: EventSource) { close() }
            override fun onFailure(es: EventSource, t: Throwable?, response: Response?) {
                close(t ?: Exception("Stream failed"))
            }
        })
    
    awaitClose { eventSource.cancel() }
}

// UI 层消费
viewModelScope.launch {
    repository.streamChat(messages).collect { token ->
        _uiState.update { it.copy(streamingText = it.streamingText + token) }
    }
}
```

**端侧模型选型：**

| 模型 | 大小 | 能力 | 适用设备 |
|------|------|------|----------|
| Gemma 2B | ~1.5GB | 基础对话/摘要 | 中高端手机 |
| Phi-3 Mini | ~2.3GB | 推理/代码 | 高端手机 |
| Llama 3.2 1B | ~700MB | 轻量对话 | 大部分手机 |
| Gemini Nano | 系统内置 | Google 设备专属 | Pixel 8+ |

### 实战层

**Compose UI 与 Streaming 结合：**
```kotlin
@Composable
fun ChatScreen(viewModel: ChatViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    
    LazyColumn {
        items(state.messages) { message ->
            MessageBubble(message)
        }
        // 流式输出中的消息
        if (state.isStreaming) {
            item {
                StreamingBubble(
                    text = state.streamingText,
                    showCursor = true  // 打字机光标效果
                )
            }
        }
    }
}

// 打字机效果
@Composable
fun StreamingBubble(text: String, showCursor: Boolean) {
    val cursorVisible by rememberInfiniteTransition().animateFloat(
        initialValue = 0f, targetValue = 1f,
        animationSpec = infiniteRepeatable(tween(500), RepeatMode.Reverse)
    )
    Text(
        text = text + if (showCursor && cursorVisible > 0.5f) "▊" else "",
        style = MaterialTheme.typography.bodyLarge
    )
}
```

**隐私与安全：**
- 敏感数据（通讯录、位置）不发送到云端 LLM
- 端侧模型处理隐私相关查询
- API Key 不硬编码，通过后端代理调用
- 用户数据加密存储，支持"删除我的数据"

**成本控制：**
- 简单意图识别用端侧模型（零成本）
- 复杂任务才调用云端 API
- Prompt Caching 减少重复 token 计费
- 批量请求合并（非实时场景）

### 延伸问题

- [[LLM应用性能优化与成本控制]]
- [[Function Calling与Tool Use机制]]
- [[OkHttp拦截器链与连接池]]

## 记忆锚点

移动端 AI Agent = 端侧小模型（快/免费/隐私）+ 云端大模型（强/贵/需网络）混合。Streaming 用 SSE 逐 token 推送，UI 用打字机效果展示。
