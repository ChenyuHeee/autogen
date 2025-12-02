# 快速入门

本指南将帮助您快速开始使用AutoGen创建AI智能体。

## 前置条件

1. 已安装Python 3.10或更高版本
2. 已安装AutoGen包（参见 [安装指南](./installation.md)）
3. OpenAI API密钥（或其他支持的模型提供商）

## 设置环境变量

在运行示例之前，需要设置OpenAI API密钥:

```bash
export OPENAI_API_KEY="sk-..."
```

## 第一个智能体：Hello World

让我们创建一个简单的智能体:

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main() -> None:
    # 创建模型客户端
    model_client = OpenAIChatCompletionClient(model="gpt-4o")
    
    # 创建智能体
    agent = AssistantAgent("assistant", model_client=model_client)
    
    # 运行任务
    result = await agent.run(task="Say 'Hello World!'")
    print(result)
    
    # 关闭模型客户端
    await model_client.close()

asyncio.run(main())
```

## 带工具的智能体

智能体可以使用工具来执行特定任务。以下是一个带天气查询工具的示例:

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient

# 创建模型客户端
model_client = OpenAIChatCompletionClient(
    model="gpt-4o",
    # api_key="YOUR_API_KEY",  # 如果未设置环境变量，在此处提供
)

# 定义一个工具函数
async def get_weather(city: str) -> str:
    """获取指定城市的天气信息。"""
    return f"{city}的天气是23摄氏度，晴朗。"

# 创建带工具的智能体
agent = AssistantAgent(
    name="weather_agent",
    model_client=model_client,
    tools=[get_weather],
    system_message="你是一个有帮助的助手。",
    reflect_on_tool_use=True,
    model_client_stream=True,  # 启用模型客户端的流式输出
)

# 运行智能体
async def main() -> None:
    await Console(agent.run_stream(task="北京的天气怎么样？"))
    await model_client.close()

asyncio.run(main())
```

## 多智能体协作

您可以创建多个智能体协同工作。以下是使用 `AgentTool` 的示例:

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.tools import AgentTool
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4o")

    # 创建数学专家智能体
    math_agent = AssistantAgent(
        "math_expert",
        model_client=model_client,
        system_message="你是一位数学专家。",
        description="一个数学专家助手。",
        model_client_stream=True,
    )
    math_agent_tool = AgentTool(math_agent, return_value_as_last_message=True)

    # 创建物理专家智能体
    physics_agent = AssistantAgent(
        "physics_expert",
        model_client=model_client,
        system_message="你是一位物理专家。",
        description="一个物理专家助手。",
        model_client_stream=True,
    )
    physics_agent_tool = AgentTool(physics_agent, return_value_as_last_message=True)

    # 创建主智能体
    agent = AssistantAgent(
        "assistant",
        system_message="你是一个通用助手。在需要时使用专家工具。",
        model_client=model_client,
        model_client_stream=True,
        tools=[math_agent_tool, physics_agent_tool],
        max_tool_iterations=10,
    )
    
    # 询问数学问题
    await Console(agent.run_stream(task="计算圆周率π的前10位小数。"))
    
    # 询问物理问题
    await Console(agent.run_stream(task="解释牛顿第三定律。"))
    
    await model_client.close()

asyncio.run(main())
```

## 使用AutoGen Studio

如果您更喜欢无代码方式，可以使用AutoGen Studio:

```bash
# 启动AutoGen Studio
autogenstudio ui --port 8080 --appdir ./my-app
```

然后在浏览器中访问 http://localhost:8080 即可使用图形界面创建和管理智能体。

## 使用其他模型

### Azure OpenAI

```python
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

# 使用AAD身份验证
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default"
)

model_client = AzureOpenAIChatCompletionClient(
    model="gpt-4o",
    api_version="2024-02-01",
    azure_endpoint="https://your-endpoint.openai.azure.com/",
    azure_ad_token_provider=token_provider,
)
```

### Anthropic Claude

```python
from autogen_ext.models.anthropic import AnthropicChatCompletionClient

model_client = AnthropicChatCompletionClient(
    model="claude-3-5-sonnet-20241022",
    # api_key="your-api-key",  # 或设置 ANTHROPIC_API_KEY 环境变量
)
```

### Ollama（本地模型）

```python
from autogen_ext.models.ollama import OllamaChatCompletionClient

model_client = OllamaChatCompletionClient(
    model="llama3.2",
    # host="http://localhost:11434",  # Ollama默认端口
)
```

### DeepSeek

DeepSeek是一个强大的中文AI模型，提供与OpenAI兼容的API接口：

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

# 使用DeepSeek对话模型
model_client = OpenAIChatCompletionClient(
    model="deepseek-chat",
    base_url="https://api.deepseek.com",
    api_key="your-deepseek-api-key",  # 从 platform.deepseek.com 获取
    model_info=ModelInfo(
        vision=False,
        function_calling=True,
        json_output=True,
        family=ModelFamily.UNKNOWN,
        structured_output=True,
    ),
)

# 或使用DeepSeek推理模型（R1）
model_client = OpenAIChatCompletionClient(
    model="deepseek-reasoner",
    base_url="https://api.deepseek.com",
    api_key="your-deepseek-api-key",
    model_info=ModelInfo(
        vision=False,
        function_calling=True,
        json_output=True,
        family=ModelFamily.R1,  # R1模型支持思维链
        structured_output=True,
    ),
)
```

更多DeepSeek使用详情，请参阅 [Extensions指南中的DeepSeek部分](./user-guide/extensions.md#deepseek)。

## 下一步

现在您已经了解了基础知识，可以继续探索:

- [用户指南](./user-guide/index.md) - 更详细的教程
- [AgentChat文档](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html) - 完整的AgentChat指南
- [Core文档](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html) - 底层API文档
- [示例代码](https://github.com/microsoft/autogen/tree/main/python/samples) - 更多示例

## 常见问题

### 1. 如何处理API限流？

使用异步调用和适当的延迟来避免API限流。

### 2. 如何保存对话历史？

可以使用内置的状态管理功能保存和恢复智能体状态。

### 3. 如何调试智能体？

启用日志记录来查看详细的执行过程:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
```
