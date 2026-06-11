---
tags: [android, 面试, ai-agent, 性能, A]
difficulty: A
frequency: medium
---

# LLM 应用性能优化与成本控制

## 问题

> LLM 应用的主要成本来源是什么？如何优化 Token 消耗和响应延迟？Prompt Caching、模型路由、批处理各自的原理和适用场景？

## 核心答案

LLM 应用成本主要来自 Token 计费（输入+输出）和推理延迟。优化手段：Prompt Caching（缓存重复的 System Prompt 前缀，减少重复计费）、模型路由（简单任务用小模型，复杂任务用大模型）、批处理（Batch API 非实时场景降低 50% 成本）、流式输出（降低首 token 延迟）、KV Cache（推理加速）。

## 深入解析

### 原理层

**成本构成：**
```
总成本 = 输入 Token 费用 + 输出 Token 费用

Claude Sonnet 4.6 定价:
- 输入: $3 / 1M tokens
- 输出: $15 / 1M tokens
- Prompt Cache 写入: $3.75 / 1M tokens
- Prompt Cache 读取: $0.30 / 1M tokens (节省 90%)

一次典型 Agent 调用:
- System Prompt: ~2000 tokens (输入)
- 对话历史: ~3000 tokens (输入)
- 工具定义: ~1000 tokens (输入)
- 用户消息: ~100 tokens (输入)
- 模型回复: ~500 tokens (输出)
- 总计: ~6100 input + 500 output ≈ $0.026/次
- 日均 10000 次 ≈ $260/天
```

**Prompt Caching 原理：**
```
无缓存:
  每次请求都发送完整 System Prompt + 工具定义 (3000 tokens)
  → 每次都按输入价格计费

有缓存:
  首次请求: 标记 cache_control，写入缓存 (3000 tokens × $3.75/M)
  后续请求: 缓存命中，只计 $0.30/M (节省 90%)
  缓存 TTL: 5 分钟（Anthropic）

// Anthropic API 示例
{
  "system": [
    {
      "type": "text",
      "text": "你是一个 Android 技术助手...(很长的 system prompt)",
      "cache_control": {"type": "ephemeral"}  // 标记缓存
    }
  ]
}
```

**模型路由（大小模型分流）：**
```python
class ModelRouter:
    def route(self, query: str, context: dict) -> str:
        complexity = self.estimate_complexity(query)
        
        if complexity == "simple":
            return "haiku"      # 简单问答，$0.25/M input
        elif complexity == "medium":
            return "sonnet"     # 常规任务，$3/M input
        else:
            return "opus"       # 复杂推理，$15/M input
    
    def estimate_complexity(self, query: str) -> str:
        # 方法1: 规则判断（关键词/长度）
        # 方法2: 用小模型分类
        # 方法3: 历史数据训练分类器
        pass

# 效果: 70% 请求用 Haiku，25% 用 Sonnet，5% 用 Opus
# 成本降低 60-80%
```

**Batch API（批处理）：**
```python
# 非实时场景（日报生成、数据标注、批量翻译）
# 成本降低 50%，但延迟 24h 内返回

import anthropic
client = anthropic.Anthropic()

# 创建批处理任务
batch = client.batches.create(
    requests=[
        {"custom_id": "req_1", "params": {"model": "claude-sonnet-4-6-20250514", "messages": [...]}},
        {"custom_id": "req_2", "params": {"model": "claude-sonnet-4-6-20250514", "messages": [...]}},
        # ... 数千个请求
    ]
)

# 轮询结果
result = client.batches.retrieve(batch.id)
```

**推理延迟优化：**

| 技术 | 原理 | 效果 |
|------|------|------|
| Streaming | 逐 token 返回 | 首 token 延迟降低 |
| KV Cache | 缓存注意力计算 | 推理加速 2-5x |
| 量化 (INT4/INT8) | 降低精度 | 速度提升，质量略降 |
| Speculative Decoding | 小模型预测+大模型验证 | 加速 2-3x |
| Prompt 精简 | 减少输入 token | 直接降低延迟和成本 |

### 实战层

**Android 端成本控制实践：**
```kotlin
class LLMCostManager {
    private var dailyTokenUsage = 0L
    private val dailyBudget = 100_000L  // 每日 token 预算
    
    suspend fun chat(messages: List<Message>): Result<String> {
        if (dailyTokenUsage > dailyBudget) {
            return Result.failure(BudgetExceededException())
        }
        
        // 模型路由
        val model = when {
            messages.last().content.length < 50 -> "haiku"
            needsReasoning(messages) -> "sonnet"
            else -> "sonnet"
        }
        
        val response = apiClient.chat(model, messages)
        dailyTokenUsage += response.usage.totalTokens
        
        // 上报用量
        analytics.trackTokenUsage(model, response.usage)
        return Result.success(response.content)
    }
}
```

**Prompt 精简技巧：**
- 移除冗余指令（LLM 不需要"请"和"谢谢"）
- 用简短的 Few-shot 示例（1-2 个而非 5 个）
- 压缩对话历史（摘要替代完整历史）
- 工具描述精简（只保留必要参数说明）

**监控与告警：**
```kotlin
// 成本监控 Dashboard
data class UsageMetrics(
    val totalTokens: Long,
    val estimatedCost: Double,
    val cacheHitRate: Double,    // Prompt Cache 命中率
    val avgLatencyMs: Long,
    val modelDistribution: Map<String, Int>  // 各模型使用占比
)
```

**Rate Limiting 与重试：**
```kotlin
// 指数退避重试
suspend fun <T> retryWithBackoff(
    maxRetries: Int = 3,
    initialDelay: Long = 1000,
    block: suspend () -> T
): T {
    var delay = initialDelay
    repeat(maxRetries) { attempt ->
        try { return block() }
        catch (e: RateLimitException) {
            if (attempt == maxRetries - 1) throw e
            delay(delay)
            delay *= 2
        }
    }
    throw IllegalStateException()
}
```

### 延伸问题

- [[AI Agent在移动端的落地]]
- [[RAG检索增强生成]]
- [[网络优化-连接复用与协议升级]]

## 记忆锚点

LLM 成本优化三板斧：Prompt Cache（重复前缀缓存省 90%）、模型路由（小事用小模型）、Batch API（非实时任务半价）。监控 token 用量就像监控 API 调用量一样重要。
