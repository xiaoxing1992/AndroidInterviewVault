---
tags: [android, 面试, ai-agent, memory, A]
difficulty: A
frequency: medium
---

# Agent Memory 系统设计

## 问题

> AI Agent 的记忆系统如何设计？短期记忆和长期记忆分别怎么实现？如何解决上下文窗口限制？

## 核心答案

Agent Memory 分三层：短期记忆（当前对话的上下文窗口，受 token 限制）、工作记忆（Scratchpad，存储当前任务的中间状态）、长期记忆（向量数据库/知识图谱，持久化存储跨会话知识）。解决上下文窗口限制的方法：对话摘要压缩、滑动窗口、重要信息提取存入长期记忆、RAG 按需检索。

## 深入解析

### 原理层

**三层记忆架构：**
```
┌─────────────────────────────────────────┐
│ 短期记忆 (Short-term / Buffer)           │
│ = 当前对话的 message 列表                 │
│ 容量: Context Window (128K tokens)       │
│ 生命周期: 单次会话                        │
├─────────────────────────────────────────┤
│ 工作记忆 (Working Memory / Scratchpad)   │
│ = 当前任务的中间结果、计划、状态           │
│ 容量: 几千 tokens                        │
│ 生命周期: 单次任务                        │
├─────────────────────────────────────────┤
│ 长期记忆 (Long-term)                     │
│ = 向量数据库 + 结构化存储                 │
│ 容量: 无限                               │
│ 生命周期: 永久                            │
│ 类型: 事实记忆 / 情景记忆 / 用户偏好      │
└─────────────────────────────────────────┘
```

**上下文窗口管理策略：**

| 策略 | 方法 | 适用场景 |
|------|------|----------|
| Buffer | 保留最近 N 条消息 | 简单对话 |
| Summary | 定期用 LLM 压缩历史为摘要 | 长对话 |
| Token Window | 保留最近 K tokens | 通用 |
| Entity Memory | 提取实体信息存储 | 需要记住用户信息 |
| Hybrid | 摘要 + 最近消息 + RAG 检索 | 复杂 Agent |

**对话压缩实现：**
```python
class ConversationMemory:
    def __init__(self, max_tokens=4000):
        self.messages = []
        self.summary = ""
        self.max_tokens = max_tokens
    
    def add_message(self, message):
        self.messages.append(message)
        if self.token_count() > self.max_tokens:
            self._compress()
    
    def _compress(self):
        # 将前半部分对话压缩为摘要
        old_messages = self.messages[:len(self.messages)//2]
        new_summary = llm.invoke(
            f"Summarize this conversation:\n{old_messages}\n"
            f"Previous summary: {self.summary}"
        )
        self.summary = new_summary
        self.messages = self.messages[len(self.messages)//2:]
    
    def get_context(self):
        return f"[Summary: {self.summary}]\n" + format_messages(self.messages)
```

**长期记忆存储：**
```python
class LongTermMemory:
    def __init__(self, vector_store, structured_store):
        self.vector_store = vector_store      # 语义检索
        self.structured_store = structured_store  # 精确查询
    
    def remember(self, content: str, metadata: dict):
        # 存入向量数据库
        embedding = embed(content)
        self.vector_store.add(content, embedding, metadata)
        
        # 提取结构化信息
        entities = extract_entities(content)
        self.structured_store.save(entities)
    
    def recall(self, query: str, k=5) -> list[str]:
        # 语义检索相关记忆
        return self.vector_store.search(query, k=k)
```

### 实战层

**移动端 Memory 方案：**
```kotlin
// Android 中的 Agent Memory 实现
class AgentMemory(
    private val db: AppDatabase,        // Room 存储结构化记忆
    private val vectorStore: LocalVectorStore,  // 本地向量存储
    private val prefs: DataStore<Preferences>   // 用户偏好
) {
    // 短期记忆：内存中的消息列表
    private val conversationBuffer = mutableListOf<Message>()
    
    // 长期记忆：持久化
    suspend fun saveMemory(content: String, type: MemoryType) {
        val embedding = embeddingModel.encode(content)
        db.memoryDao().insert(MemoryEntity(content, embedding, type, timestamp))
    }
    
    // 检索相关记忆
    suspend fun recall(query: String): List<String> {
        val queryEmbedding = embeddingModel.encode(query)
        return db.memoryDao().searchSimilar(queryEmbedding, limit = 5)
    }
    
    // 构建完整上下文
    suspend fun buildContext(currentQuery: String): String {
        val relevantMemories = recall(currentQuery)
        val recentMessages = conversationBuffer.takeLast(10)
        return """
            |[User Preferences]: ${getUserPreferences()}
            |[Relevant Memories]: ${relevantMemories.joinToString("\n")}
            |[Recent Conversation]: ${recentMessages.format()}
        """.trimMargin()
    }
}
```

**记忆的生命周期管理：**
- 重要性评分：根据被检索频率和用户反馈调整权重
- 遗忘机制：长期未访问的记忆降低优先级
- 冲突解决：新信息与旧记忆矛盾时，以新信息为准
- 隐私清理：用户要求"忘记"时彻底删除

### 延伸问题

- [[RAG检索增强生成]]
- [[LLM Agent架构设计-ReAct与Plan-and-Execute]]
- [[LLM应用性能优化与成本控制]]

## 记忆锚点

Agent Memory = 短期（对话窗口）+ 工作（任务草稿）+ 长期（向量库）。核心挑战是上下文窗口有限，解法是压缩+检索：不重要的压缩成摘要，需要时从长期记忆 RAG 检索。
