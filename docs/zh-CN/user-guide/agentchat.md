# AgentChat 指南

AgentChat是AutoGen的高级API，用于构建多智能体对话应用程序。它建立在Core API之上，提供更简单易用的接口。

## 简介

AgentChat适用于：

- 快速原型设计
- 构建对话式AI应用
- 多智能体协作场景

## 核心组件

### AssistantAgent

`AssistantAgent` 是最常用的智能体类型，它可以：

- 回答问题
- 使用工具
- 执行代码
- 与其他智能体协作

```python
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient

model_client = OpenAIChatCompletionClient(model="gpt-4o")

agent = AssistantAgent(
    name="assistant",
    model_client=model_client,
    system_message="你是一个有帮助的AI助手。",
    tools=[],  # 可选：添加工具
    reflect_on_tool_use=True,  # 工具使用后进行反思
)
```

### UserProxyAgent

`UserProxyAgent` 代表人类用户，用于人机交互场景：

```python
from autogen_agentchat.agents import UserProxyAgent

user = UserProxyAgent(name="user")
```

### CodeExecutorAgent

`CodeExecutorAgent` 用于执行代码：

```python
from autogen_agentchat.agents import CodeExecutorAgent
from autogen_ext.code_executors.local import LocalCommandLineCodeExecutor

executor = LocalCommandLineCodeExecutor(work_dir="./workspace")
code_agent = CodeExecutorAgent(
    name="code_executor",
    code_executor=executor,
)
```

## 团队模式

### RoundRobinGroupChat

轮流发言模式：

```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import TextMentionTermination

team = RoundRobinGroupChat(
    participants=[agent1, agent2],
    termination_condition=TextMentionTermination("TERMINATE"),
)

result = await team.run(task="讨论一下人工智能的未来。")
```

### SelectorGroupChat

选择器模式，由模型决定下一个发言者：

```python
from autogen_agentchat.teams import SelectorGroupChat

team = SelectorGroupChat(
    participants=[researcher, writer, reviewer],
    model_client=model_client,
    selector_prompt="""根据对话内容，选择最适合下一步的参与者：
    - researcher: 需要研究或收集信息时
    - writer: 需要撰写内容时  
    - reviewer: 需要审查或评估时
    """,
)
```

### Swarm

Swarm模式，通过工具调用切换智能体：

```python
from autogen_agentchat.teams import Swarm
from autogen_agentchat.agents import AssistantAgent

# 创建可以切换到其他智能体的工具
def transfer_to_sales():
    """将对话转移给销售智能体。"""
    return sales_agent

def transfer_to_support():
    """将对话转移给支持智能体。"""
    return support_agent

# 创建智能体
triage_agent = AssistantAgent(
    name="triage",
    model_client=model_client,
    system_message="你是一个分诊智能体。根据用户需求，将对话转移给合适的部门。",
    tools=[transfer_to_sales, transfer_to_support],
)

sales_agent = AssistantAgent(
    name="sales",
    model_client=model_client,
    system_message="你是销售智能体。帮助用户了解产品和定价。",
)

support_agent = AssistantAgent(
    name="support", 
    model_client=model_client,
    system_message="你是客户支持智能体。帮助用户解决技术问题。",
)

# 创建Swarm团队
team = Swarm(participants=[triage_agent, sales_agent, support_agent])
```

## 终止条件

AutoGen提供多种终止条件：

### MaxMessageTermination

限制最大消息数：

```python
from autogen_agentchat.conditions import MaxMessageTermination

termination = MaxMessageTermination(max_messages=10)
```

### TextMentionTermination

检测特定文本：

```python
from autogen_agentchat.conditions import TextMentionTermination

termination = TextMentionTermination("TERMINATE")
```

### TokenUsageTermination

限制Token使用量：

```python
from autogen_agentchat.conditions import TokenUsageTermination

termination = TokenUsageTermination(max_total_token=10000)
```

### 组合终止条件

使用 `|` (或) 和 `&` (与) 组合条件：

```python
from autogen_agentchat.conditions import (
    MaxMessageTermination,
    TextMentionTermination,
)

# 达到10条消息或出现TERMINATE时终止
termination = MaxMessageTermination(10) | TextMentionTermination("TERMINATE")
```

## 工具使用

### 定义工具

工具可以是任何Python函数：

```python
async def search_web(query: str) -> str:
    """在网上搜索信息。
    
    Args:
        query: 搜索查询
        
    Returns:
        搜索结果
    """
    # 实现搜索逻辑
    return f"搜索 '{query}' 的结果..."

async def calculate(expression: str) -> str:
    """计算数学表达式。
    
    Args:
        expression: 数学表达式
        
    Returns:
        计算结果
    """
    try:
        result = eval(expression)
        return str(result)
    except Exception as e:
        return f"计算错误: {e}"
```

### 使用MCP服务器

使用Model Context Protocol (MCP) 服务器：

```python
from autogen_ext.tools.mcp import McpWorkbench, StdioServerParams

server_params = StdioServerParams(
    command="npx",
    args=["@playwright/mcp@latest", "--headless"],
)

async with McpWorkbench(server_params) as mcp:
    agent = AssistantAgent(
        "browser_agent",
        model_client=model_client,
        workbench=mcp,
    )
    await agent.run(task="打开 https://example.com 并获取页面标题")
```

## 流式输出

使用流式输出实时显示结果：

```python
from autogen_agentchat.ui import Console

# 使用Console显示流式输出
await Console(agent.run_stream(task="写一首诗"))
```

自定义流式处理：

```python
async for message in agent.run_stream(task="写一首诗"):
    if hasattr(message, "content"):
        print(message.content, end="", flush=True)
```

## 状态管理

保存和恢复智能体状态：

```python
import json

# 保存状态
state = await agent.save_state()
with open("agent_state.json", "w") as f:
    json.dump(state, f)

# 恢复状态
with open("agent_state.json", "r") as f:
    state = json.load(f)
await agent.load_state(state)
```

## 日志记录

启用详细日志：

```python
import logging

# 设置日志级别
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("autogen_agentchat")
logger.setLevel(logging.DEBUG)
```

## 最佳实践

### 1. 清晰的系统消息

为每个智能体编写清晰、具体的系统消息：

```python
agent = AssistantAgent(
    name="data_analyst",
    model_client=model_client,
    system_message="""你是一位专业的数据分析师。
    
    你的职责：
    1. 分析数据集
    2. 发现数据模式
    3. 生成可视化报告
    4. 提供数据驱动的建议
    
    始终使用准确的统计方法，并清晰解释你的发现。
    """,
)
```

### 2. 合理的工具设计

设计小而专注的工具：

```python
# 好的设计：单一职责
async def read_file(path: str) -> str:
    """读取文件内容。"""
    ...

async def write_file(path: str, content: str) -> str:
    """写入文件内容。"""
    ...

# 避免：过于复杂的工具
async def file_operations(operation: str, path: str, content: str = None) -> str:
    """执行文件操作。"""  # 职责不清晰
    ...
```

### 3. 适当的错误处理

在工具中处理错误：

```python
async def fetch_data(url: str) -> str:
    """从URL获取数据。"""
    try:
        # 实现获取逻辑
        return data
    except ConnectionError:
        return "错误：无法连接到服务器"
    except TimeoutError:
        return "错误：请求超时"
    except Exception as e:
        return f"错误：{str(e)}"
```

## 下一步

- 查看 [Core指南](./core.md) 了解底层API
- 查看 [Extensions指南](./extensions.md) 了解可用的扩展
- 查看 [官方示例](https://github.com/microsoft/autogen/tree/main/python/samples) 获取更多代码示例
