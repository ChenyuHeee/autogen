# 用户指南

欢迎来到AutoGen用户指南！本指南将帮助您深入了解AutoGen的各个组件和功能。

## 目录

- [AgentChat指南](./agentchat.md) - 高级多智能体对话API
- [Core指南](./core.md) - 底层核心API
- [Extensions指南](./extensions.md) - 扩展组件

## AutoGen架构

AutoGen采用分层架构设计，每一层都有明确的职责：

```
┌─────────────────────────────────────────┐
│           AutoGen Studio                │  <- 无代码GUI界面
├─────────────────────────────────────────┤
│           AgentChat API                 │  <- 高级多智能体API
├─────────────────────────────────────────┤
│           Core API                      │  <- 底层事件驱动API
├─────────────────────────────────────────┤
│           Extensions                    │  <- 扩展组件（模型、工具等）
└─────────────────────────────────────────┘
```

### Core API

Core API是AutoGen的基础层，提供：

- **消息传递**: 智能体之间的通信机制
- **事件驱动架构**: 基于事件的响应处理
- **本地和分布式运行时**: 支持不同的部署场景
- **跨语言支持**: Python和.NET

### AgentChat API

AgentChat API建立在Core API之上，提供：

- **预设智能体**: 即用型智能体类型
- **团队模式**: 常见的多智能体协作模式
- **简化接口**: 更容易上手的API

### Extensions

Extensions提供与外部服务的集成：

- **模型客户端**: OpenAI、Azure OpenAI、Anthropic等
- **工具**: MCP服务器、代码执行器等
- **运行时**: gRPC分布式运行时

## 核心概念

### 智能体（Agent）

智能体是AutoGen的核心组件，代表一个可以执行任务的实体。

```python
from autogen_agentchat.agents import AssistantAgent

agent = AssistantAgent(
    name="my_agent",
    model_client=model_client,
    system_message="你是一个有帮助的助手。",
)
```

### 消息（Message）

消息是智能体之间通信的基本单位。

```python
from autogen_core import MessageContext
from autogen_agentchat.messages import TextMessage

message = TextMessage(content="你好！", source="user")
```

### 工具（Tool）

工具允许智能体执行特定操作，如调用API、执行代码等。

```python
async def calculate(expression: str) -> str:
    """计算数学表达式。"""
    return str(eval(expression))

agent = AssistantAgent(
    name="calculator",
    model_client=model_client,
    tools=[calculate],
)
```

### 团队（Team）

团队是多个智能体的集合，可以协同完成复杂任务。

```python
from autogen_agentchat.teams import SelectorGroupChat

team = SelectorGroupChat(
    participants=[agent1, agent2, agent3],
    model_client=model_client,
)
```

## 常用模式

### 两智能体对话

两个智能体之间的对话：

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat

agent1 = AssistantAgent("agent1", model_client=model_client)
agent2 = AssistantAgent("agent2", model_client=model_client)

team = RoundRobinGroupChat([agent1, agent2])
```

### 选择器群聊

使用选择器决定下一个发言者：

```python
from autogen_agentchat.teams import SelectorGroupChat

team = SelectorGroupChat(
    participants=[agent1, agent2, agent3],
    model_client=model_client,
    selector_prompt="选择最适合回答当前问题的智能体。",
)
```

### Swarm模式

基于工具的智能体切换：

```python
from autogen_agentchat.teams import Swarm

team = Swarm(
    participants=[agent1, agent2, agent3],
)
```

## 最佳实践

### 1. 使用适当的系统消息

为智能体提供清晰、具体的系统消息：

```python
agent = AssistantAgent(
    name="code_reviewer",
    model_client=model_client,
    system_message="""你是一位专业的代码审查员。
    请仔细检查代码中的：
    - 语法错误
    - 逻辑问题
    - 性能优化机会
    - 代码风格问题
    """,
)
```

### 2. 合理使用工具

只提供智能体需要的工具：

```python
agent = AssistantAgent(
    name="data_analyst",
    model_client=model_client,
    tools=[read_csv, calculate_stats, create_chart],
)
```

### 3. 设置终止条件

为任务设置合适的终止条件：

```python
from autogen_agentchat.conditions import MaxMessageTermination

termination = MaxMessageTermination(max_messages=10)
```

### 4. 处理错误

实现适当的错误处理：

```python
try:
    result = await agent.run(task="执行任务")
except Exception as e:
    print(f"错误: {e}")
```

## 下一步

- 查看 [AgentChat指南](./agentchat.md) 了解更多多智能体模式
- 查看 [Core指南](./core.md) 了解底层API
- 查看 [Extensions指南](./extensions.md) 了解可用的扩展
