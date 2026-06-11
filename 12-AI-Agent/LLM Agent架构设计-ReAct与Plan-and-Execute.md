---
tags: [android, 面试, ai-agent, 架构, A]
difficulty: A
frequency: high
---

# LLM Agent 架构设计 - ReAct 与 Plan-and-Execute

## 问题

> 什么是 LLM Agent？ReAct 和 Plan-and-Execute 两种 Agent 模式的区别是什么？如何选择？

## 核心答案

LLM Agent 是以大语言模型为"大脑"，通过感知环境、推理决策、调用工具来自主完成任务的系统。ReAct 模式交替进行推理（Thought）和行动（Action），逐步解决问题，适合探索性任务；Plan-and-Execute 先生成完整计划再逐步执行，适合结构化、多步骤任务。核心区别是"边想边做" vs "先想后做"。

## 深入解析

### 原理层

**Agent 核心循环：**
```
┌─────────────────────────────────────┐
│         Agent Loop                   │
│                                      │
│  Observe → Think → Act → Observe... │
│     ↑                        │       │
│     └────────────────────────┘       │
└─────────────────────────────────────┘

组成要素:
- LLM (大脑): 推理、决策、生成
- Tools (手脚): API 调用、搜索、代码执行
- Memory (记忆): 对话历史、长期知识
- Planning (规划): 任务分解、策略选择
```

**ReAct 模式（Reasoning + Acting）：**
```
User: 查询北京明天的天气并推荐穿搭

Thought 1: 我需要先查询北京明天的天气
Action 1: call weather_api(city="北京", date="tomorrow")
Observation 1: 晴天，15-25°C，微风

Thought 2: 天气晴朗温差较大，我来推荐穿搭
Action 2: generate_recommendation(weather="晴天15-25°C")
Observation 2: 建议外套+T恤...

Thought 3: 我已经有了完整信息，可以回复用户
Final Answer: 北京明天晴天...建议...
```

**Plan-and-Execute 模式：**
```
User: 帮我分析竞品 App 的功能并写一份报告

Plan:
1. 搜索竞品 App 列表
2. 逐个分析核心功能
3. 对比功能差异
4. 生成分析报告

Execute Step 1: call search_apps(category="...")
Execute Step 2: call analyze_app(app_id="...")
Execute Step 3: call compare_features(apps=[...])
Execute Step 4: call generate_report(data=...)

// 执行中可以 Replan（根据中间结果调整计划）
```

**两种模式对比：**

| 维度 | ReAct | Plan-and-Execute |
|------|-------|-----------------|
| 决策方式 | 逐步推理，边想边做 | 先规划全局，再逐步执行 |
| 适合任务 | 探索性、不确定性高 | 结构化、步骤明确 |
| Token 消耗 | 较少（每步只看当前） | 较多（需要维护计划） |
| 容错性 | 高（每步可调整） | 中（需要 Replan 机制） |
| 实现复杂度 | 低 | 中 |

**LangGraph 实现示例：**
```python
from langgraph.graph import StateGraph, END

# 定义状态
class AgentState(TypedDict):
    messages: list
    plan: list[str]
    current_step: int

# ReAct 节点
def react_node(state):
    response = llm.invoke(state["messages"])
    if response.tool_calls:
        return {"messages": [response], "next": "tools"}
    return {"messages": [response], "next": END}

# Plan-and-Execute
def planner_node(state):
    plan = llm.invoke("Create a plan for: " + state["task"])
    return {"plan": plan.steps}

def executor_node(state):
    step = state["plan"][state["current_step"]]
    result = execute_step(step)
    return {"current_step": state["current_step"] + 1}
```

### 实战层

**选型建议：**
- 简单问答 + 单工具调用 → 不需要 Agent，直接 Function Calling
- 多步骤但路径不确定 → ReAct
- 复杂任务、步骤可预知 → Plan-and-Execute
- 需要人工确认 → Plan-and-Execute + Human-in-the-loop

**Android/移动端集成：**
```kotlin
// Agent 通常运行在服务端，移动端作为 UI 层
// 通过 SSE/WebSocket 接收 Agent 的中间状态
sealed interface AgentEvent {
    data class Thinking(val thought: String) : AgentEvent
    data class ToolCall(val tool: String, val args: Map<String, Any>) : AgentEvent
    data class ToolResult(val result: String) : AgentEvent
    data class FinalAnswer(val answer: String) : AgentEvent
}
```

**常见问题：**
- Agent 陷入循环（反复调用同一工具）→ 设置最大迭代次数
- 幻觉导致错误工具调用 → 工具描述要精确、加 validation
- Token 超限 → 对话压缩、滑动窗口

### 延伸问题

- [[Function Calling与Tool Use机制]]
- [[Multi-Agent多智能体协作]]
- [[Prompt Engineering与思维链]]

## 记忆锚点

Agent = LLM + Tools + Memory + Loop。ReAct 是"走一步看一步"，Plan-and-Execute 是"先画地图再出发"。
