---
tags: [android, 面试, ai-agent, multi-agent, S]
difficulty: S
frequency: medium
---

# Multi-Agent 多智能体协作

## 问题

> 多 Agent 系统的架构模式有哪些？Agent 之间如何通信和协调？如何处理任务分配和冲突？

## 核心答案

多 Agent 架构主要有三种模式：层级式（Orchestrator 分配任务给 Worker）、对等式（Agent 间平等协商）、专家混合式（Router 根据任务类型分发给专家 Agent）。通信通过消息传递（共享状态/事件总线），协调通过任务分解、角色分工、投票/仲裁机制实现。关键挑战是避免冲突、减少冗余、保证一致性。

## 深入解析

### 原理层

**三种架构模式：**

```
1. 层级式 (Hierarchical)
   ┌──────────────┐
   │ Orchestrator │  ← 总指挥，分解任务
   └──────┬───────┘
     ┌────┼────┐
     ↓    ↓    ↓
   Agent Agent Agent  ← Worker，执行子任务
   (搜索) (分析) (写作)

2. 对等式 (Peer-to-Peer)
   Agent A ←→ Agent B ←→ Agent C
   (各自独立，通过消息协商)

3. 专家混合式 (Mixture of Experts)
   ┌────────┐
   │ Router │  ← 根据任务类型路由
   └───┬────┘
   ┌───┼───┐
   ↓   ↓   ↓
  Code  UI  Data  ← 各领域专家
  Agent Agent Agent
```

**LangGraph 多 Agent 实现：**
```python
from langgraph.graph import StateGraph

# 定义共享状态
class TeamState(TypedDict):
    task: str
    plan: list[str]
    results: dict[str, str]
    final_output: str

# Orchestrator 节点
def orchestrator(state: TeamState):
    plan = llm.invoke(f"Break down this task: {state['task']}")
    return {"plan": plan.steps}

# Worker 节点
def researcher(state: TeamState):
    step = get_current_step(state, "research")
    result = research_llm.invoke(step)
    return {"results": {**state["results"], "research": result}}

def writer(state: TeamState):
    result = writer_llm.invoke(
        f"Write based on research: {state['results']['research']}"
    )
    return {"results": {**state["results"], "writing": result}}

# 构建图
graph = StateGraph(TeamState)
graph.add_node("orchestrator", orchestrator)
graph.add_node("researcher", researcher)
graph.add_node("writer", writer)
graph.add_edge("orchestrator", "researcher")
graph.add_edge("researcher", "writer")
graph.add_edge("writer", END)
```

**Agent 间通信模式：**

| 模式 | 实现 | 适用场景 |
|------|------|----------|
| 共享状态 | 全局 State 对象 | 简单协作 |
| 消息传递 | Queue/Channel | 异步协作 |
| 黑板系统 | 共享知识库 | 知识密集型 |
| 发布订阅 | Event Bus | 松耦合 |

**冲突解决机制：**
```python
# 1. 投票机制（多个 Agent 给出答案，取多数）
answers = [agent.solve(problem) for agent in agents]
final = majority_vote(answers)

# 2. 仲裁者（专门的 Judge Agent 评估）
def judge(answers: list[str], criteria: str) -> str:
    return judge_llm.invoke(
        f"Evaluate these answers based on {criteria}: {answers}"
    )

# 3. 辩论（Agent 互相质疑，迭代改进）
for round in range(3):
    for agent in agents:
        agent.critique(other_agents.answers)
        agent.revise_answer()
```

### 实战层

**实际应用场景：**
- **代码生成**：Architect Agent 设计 → Coder Agent 实现 → Reviewer Agent 审查
- **研究报告**：Researcher 搜索 → Analyst 分析 → Writer 撰写 → Editor 校对
- **客服系统**：Router 分类 → 专家 Agent 处理 → QA Agent 质检

**框架对比：**

| 框架 | 特点 | 适用 |
|------|------|------|
| LangGraph | 图结构，灵活 | 复杂工作流 |
| CrewAI | 角色驱动，简单 | 快速原型 |
| AutoGen | 对话驱动 | 研究/探索 |
| Claude Agent SDK | Anthropic 原生 | Claude 生态 |

**成本控制：**
```python
# 大小模型混合使用
router_model = "haiku"      # 路由判断用小模型（便宜）
expert_model = "sonnet"     # 专家处理用中等模型
judge_model = "opus"        # 最终判断用大模型（贵但准）
```

**常见问题：**
- Agent 之间信息丢失 → 确保共享状态完整传递
- 无限循环（互相推诿）→ 设置最大轮次 + 超时
- 成本爆炸 → 限制每个 Agent 的 token 预算

### 延伸问题

- [[LLM Agent架构设计-ReAct与Plan-and-Execute]]
- [[Function Calling与Tool Use机制]]
- [[LLM应用性能优化与成本控制]]

## 记忆锚点

Multi-Agent = 团队协作。层级式像公司（老板分活），对等式像圆桌会议（平等讨论），专家式像医院（分诊到专科）。核心是分工明确 + 通信顺畅 + 冲突有仲裁。
