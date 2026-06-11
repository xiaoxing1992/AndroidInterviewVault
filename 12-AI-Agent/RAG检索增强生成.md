---
tags: [android, 面试, ai-agent, rag, S]
difficulty: S
frequency: high
---

# RAG 检索增强生成

## 问题

> RAG 的完整架构是什么？如何选择 Embedding 模型和向量数据库？Chunk 策略对效果有什么影响？如何评估 RAG 系统的质量？

## 核心答案

RAG（Retrieval-Augmented Generation）通过检索外部知识库为 LLM 提供上下文，解决幻觉和知识时效性问题。完整流程：文档切分（Chunking）→ 向量化（Embedding）→ 存入向量数据库 → 用户查询向量化 → 相似度检索 → 检索结果作为 Context 注入 Prompt → LLM 生成回答。关键优化点在 Chunk 策略、Embedding 质量和 Reranking。

## 深入解析

### 原理层

**RAG 完整架构：**
```
离线索引阶段:
Documents → Chunking → Embedding Model → Vector DB
   │           │            │                │
   PDF/MD    按语义切分   text→vector     Chroma/Pinecone

在线查询阶段:
User Query → Embedding → Vector Search → Reranking → Top-K Context
                                                          │
                                                          ↓
                                              Prompt = System + Context + Query
                                                          │
                                                          ↓
                                                    LLM → Answer
```

**Chunking 策略对比：**

| 策略 | 方法 | 适用场景 |
|------|------|----------|
| Fixed Size | 固定 token 数切分 | 通用，简单 |
| Recursive | 按段落→句子→字符递归切分 | 结构化文档 |
| Semantic | 按语义相似度切分（embedding 距离） | 高质量需求 |
| Document-based | 按文档自然结构（标题/章节） | 技术文档 |
| Sliding Window | 重叠窗口（overlap） | 避免上下文断裂 |

```python
# LangChain Recursive Chunking
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", "，", " "]
)
chunks = splitter.split_documents(documents)
```

**Embedding 模型选型：**
```
模型                    维度    中文支持   性能
text-embedding-3-small  1536    好        快/便宜
text-embedding-3-large  3072    好        慢/贵
bge-large-zh-v1.5       1024    优秀      本地部署
m3e-base                768     优秀      轻量
```

**向量检索 + Reranking（两阶段）：**
```python
# 阶段1: 向量检索（召回，快但粗）
results = vector_db.similarity_search(query, k=20)

# 阶段2: Reranking（精排，慢但准）
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
scores = reranker.predict([(query, doc.text) for doc in results])
top_docs = sorted(zip(results, scores), key=lambda x: x[1], reverse=True)[:5]
```

**Hybrid Search（混合检索）：**
```
向量检索: 语义相似（"如何优化启动速度" ≈ "App 冷启动加速"）
关键词检索: 精确匹配（BM25，适合专有名词/代码）
混合: RRF (Reciprocal Rank Fusion) 合并两种结果
```

### 实战层

**RAG 评估指标：**
- **Faithfulness**：回答是否忠于检索到的上下文（不编造）
- **Answer Relevance**：回答是否切题
- **Context Relevance**：检索到的文档是否相关
- **Context Recall**：是否检索到了所有需要的信息

```python
# 使用 Ragas 评估
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset=eval_dataset,
    metrics=[faithfulness, answer_relevancy, context_precision]
)
```

**常见优化手段：**
1. **Query Rewriting**：用 LLM 改写用户查询，提高检索质量
2. **HyDE**：先让 LLM 生成假设性答案，用答案去检索
3. **Parent Document Retriever**：检索小 chunk，返回大 chunk（上下文更完整）
4. **Multi-Query**：生成多个查询变体，合并检索结果
5. **Metadata Filtering**：结合元数据（时间/来源/类别）过滤

**移动端 RAG 场景：**
- App 内帮助文档智能问答
- 本地知识库（SQLite + 向量扩展）
- 端侧 Embedding（ONNX Runtime）+ 云端 LLM 生成

### 延伸问题

- [[Agent Memory系统设计]]
- [[LLM应用性能优化与成本控制]]
- [[Prompt Engineering与思维链]]

## 记忆锚点

RAG = 开卷考试。LLM 自己答题容易编（幻觉），给它参考资料（检索结果）就靠谱多了。核心是检索质量：切得好、找得准、排得对。
