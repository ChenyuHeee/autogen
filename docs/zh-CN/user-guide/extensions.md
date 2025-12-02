# Extensions 指南

Extensions是AutoGen的扩展组件，提供与外部服务和工具的集成。

## 模型客户端

AutoGen支持多种大语言模型提供商。

### OpenAI

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient

# 基本用法
model_client = OpenAIChatCompletionClient(
    model="gpt-4o",
    # api_key="your-api-key",  # 或使用 OPENAI_API_KEY 环境变量
)

# 带配置的用法
model_client = OpenAIChatCompletionClient(
    model="gpt-4o",
    temperature=0.7,
    max_tokens=2000,
)
```

### Azure OpenAI

```python
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient

# 使用API密钥
model_client = AzureOpenAIChatCompletionClient(
    model="gpt-4o",
    api_version="2024-02-01",
    azure_endpoint="https://your-endpoint.openai.azure.com/",
    api_key="your-api-key",
)

# 使用AAD身份验证
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "https://cognitiveservices.azure.com/.default"
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
    # api_key="your-api-key",  # 或使用 ANTHROPIC_API_KEY 环境变量
)
```

### Google Gemini

```python
from autogen_ext.models.google import GoogleGenerativeModelClient

model_client = GoogleGenerativeModelClient(
    model="gemini-pro",
    # api_key="your-api-key",  # 或使用 GOOGLE_API_KEY 环境变量
)
```

### Ollama（本地模型）

```python
from autogen_ext.models.ollama import OllamaChatCompletionClient

model_client = OllamaChatCompletionClient(
    model="llama3.2",
    host="http://localhost:11434",  # Ollama默认地址
)
```

### DeepSeek

DeepSeek提供与OpenAI兼容的API接口，可以直接使用`OpenAIChatCompletionClient`通过设置`base_url`来使用DeepSeek模型。

#### 获取API密钥

1. 访问 [DeepSeek开放平台](https://platform.deepseek.com/)
2. 注册并登录账户
3. 在API密钥页面创建新的API密钥
4. 确保账户有足够余额（DeepSeek API需要预付费）

#### 基本用法

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

# 创建DeepSeek模型客户端
model_client = OpenAIChatCompletionClient(
    model="deepseek-chat",  # DeepSeek对话模型
    base_url="https://api.deepseek.com",  # DeepSeek API地址
    api_key="your-deepseek-api-key",  # 或设置 OPENAI_API_KEY 环境变量
    model_info=ModelInfo(
        vision=False,
        function_calling=True,
        json_output=True,
        family=ModelFamily.UNKNOWN,
        structured_output=True,
    ),
)
```

#### 使用DeepSeek-R1推理模型

DeepSeek-R1是一个强大的推理模型，特别适合需要深度思考的任务：

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

# 创建DeepSeek-R1推理模型客户端
model_client = OpenAIChatCompletionClient(
    model="deepseek-reasoner",  # DeepSeek推理模型
    base_url="https://api.deepseek.com",
    api_key="your-deepseek-api-key",
    model_info=ModelInfo(
        vision=False,
        function_calling=True,
        json_output=True,
        family=ModelFamily.R1,  # 使用R1模型族以支持思维链
        structured_output=True,
    ),
)
```

#### 完整示例：使用DeepSeek创建智能体

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

async def main():
    # 创建DeepSeek模型客户端
    model_client = OpenAIChatCompletionClient(
        model="deepseek-chat",
        base_url="https://api.deepseek.com",
        api_key="your-deepseek-api-key",  # 建议使用环境变量
        model_info=ModelInfo(
            vision=False,
            function_calling=True,
            json_output=True,
            family=ModelFamily.UNKNOWN,
            structured_output=True,
        ),
    )

    # 创建智能体
    agent = AssistantAgent(
        name="deepseek_assistant",
        model_client=model_client,
        system_message="你是一个有帮助的AI助手，使用中文回答问题。",
    )

    # 运行对话
    await Console(agent.run_stream(task="请解释什么是多智能体系统？"))
    
    # 关闭客户端
    await model_client.close()

asyncio.run(main())
```

#### 使用环境变量配置

推荐使用环境变量来管理API密钥：

```bash
# 设置环境变量
export DEEPSEEK_API_KEY="your-deepseek-api-key"
```

```python
import os
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

model_client = OpenAIChatCompletionClient(
    model="deepseek-chat",
    base_url="https://api.deepseek.com",
    api_key=os.environ.get("DEEPSEEK_API_KEY"),
    model_info=ModelInfo(
        vision=False,
        function_calling=True,
        json_output=True,
        family=ModelFamily.UNKNOWN,
        structured_output=True,
    ),
)
```

#### DeepSeek带工具的智能体

DeepSeek模型支持函数调用（Function Calling），可以使用工具：

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

# 定义工具函数
async def get_weather(city: str) -> str:
    """获取指定城市的天气信息。
    
    Args:
        city: 城市名称
        
    Returns:
        天气信息字符串
    """
    # 这里是模拟的天气数据
    weather_data = {
        "北京": "晴天，温度25°C",
        "上海": "多云，温度28°C",
        "深圳": "阵雨，温度30°C",
    }
    return weather_data.get(city, f"{city}的天气数据暂不可用")

async def calculate(expression: str) -> str:
    """计算数学表达式。
    
    Args:
        expression: 数学表达式字符串
        
    Returns:
        计算结果
    """
    import ast
    import operator
    
    # 安全的数学运算符映射
    operators = {
        ast.Add: operator.add,
        ast.Sub: operator.sub,
        ast.Mult: operator.mul,
        ast.Div: operator.truediv,
        ast.Pow: operator.pow,
        ast.USub: operator.neg,
    }
    
    def safe_eval(node):
        if isinstance(node, ast.Constant):
            return node.value
        elif isinstance(node, ast.BinOp):
            left = safe_eval(node.left)
            right = safe_eval(node.right)
            return operators[type(node.op)](left, right)
        elif isinstance(node, ast.UnaryOp):
            operand = safe_eval(node.operand)
            return operators[type(node.op)](operand)
        else:
            raise ValueError(f"不支持的表达式: {ast.dump(node)}")
    
    try:
        tree = ast.parse(expression, mode='eval')
        result = safe_eval(tree.body)
        return f"计算结果: {result}"
    except Exception as e:
        return f"计算错误: {e}"

async def main():
    model_client = OpenAIChatCompletionClient(
        model="deepseek-chat",
        base_url="https://api.deepseek.com",
        api_key="your-deepseek-api-key",
        model_info=ModelInfo(
            vision=False,
            function_calling=True,
            json_output=True,
            family=ModelFamily.UNKNOWN,
            structured_output=True,
        ),
    )

    # 创建带工具的智能体
    agent = AssistantAgent(
        name="deepseek_tool_agent",
        model_client=model_client,
        tools=[get_weather, calculate],
        system_message="你是一个有帮助的AI助手。可以查询天气和进行数学计算。",
        reflect_on_tool_use=True,
    )

    # 测试天气查询
    await Console(agent.run_stream(task="北京今天天气怎么样？"))
    
    # 测试数学计算
    await Console(agent.run_stream(task="请计算 123 * 456 + 789"))
    
    await model_client.close()

asyncio.run(main())
```

#### DeepSeek多智能体协作

使用DeepSeek模型创建多智能体系统：

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import MaxMessageTermination
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

async def main():
    # 创建DeepSeek模型客户端
    model_client = OpenAIChatCompletionClient(
        model="deepseek-chat",
        base_url="https://api.deepseek.com",
        api_key="your-deepseek-api-key",
        model_info=ModelInfo(
            vision=False,
            function_calling=True,
            json_output=True,
            family=ModelFamily.UNKNOWN,
            structured_output=True,
        ),
    )

    # 创建程序员智能体
    programmer = AssistantAgent(
        name="programmer",
        model_client=model_client,
        system_message="""你是一位资深的Python程序员。
        你的职责是编写高质量、可维护的代码。
        在回复中始终使用代码块展示代码。""",
    )

    # 创建代码审查智能体
    reviewer = AssistantAgent(
        name="reviewer",
        model_client=model_client,
        system_message="""你是一位代码审查专家。
        你的职责是审查代码，找出潜在问题并提出改进建议。
        关注代码的正确性、可读性和性能。""",
    )

    # 创建团队
    team = RoundRobinGroupChat(
        participants=[programmer, reviewer],
        termination_condition=MaxMessageTermination(max_messages=6),
    )

    # 运行团队对话
    await Console(
        team.run_stream(task="请编写一个Python函数，用于检查一个字符串是否是回文。")
    )
    
    await model_client.close()

asyncio.run(main())
```

#### DeepSeek可用模型

| 模型名称 | 描述 | 适用场景 |
|---------|------|---------|
| `deepseek-chat` | 通用对话模型 | 日常对话、问答、内容创作 |
| `deepseek-reasoner` | 推理模型(R1) | 复杂推理、数学问题、代码生成 |
| `deepseek-coder` | 代码专用模型 | 代码生成、代码解释、调试 |

#### 注意事项

1. **API限流**: DeepSeek有API调用频率限制，请注意合理使用
2. **费用**: DeepSeek API为预付费模式，请确保账户余额充足
3. **模型能力**: 使用`model_info`参数告诉AutoGen模型的能力，这对于非OpenAI模型是必需的
4. **网络连接**: 确保网络能够访问 `api.deepseek.com`

### 模型客户端接口

所有模型客户端都实现相同的接口：

```python
from autogen_core.models import ChatCompletionClient

async def use_model(client: ChatCompletionClient):
    from autogen_core.models import UserMessage
    
    response = await client.create(
        messages=[
            UserMessage(content="你好！", source="user"),
        ]
    )
    print(response.content)
```

## 代码执行器

AutoGen提供多种代码执行环境。

### 本地执行器

```python
from autogen_ext.code_executors.local import LocalCommandLineCodeExecutor

executor = LocalCommandLineCodeExecutor(
    work_dir="./workspace",
    timeout=60,  # 超时时间（秒）
)

# 执行代码
result = await executor.execute_code_blocks(
    code_blocks=[
        CodeBlock(language="python", code="print('Hello, World!')"),
    ],
)
print(result.output)
```

### Docker执行器

```python
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor

executor = DockerCommandLineCodeExecutor(
    image="python:3.11",
    work_dir="./workspace",
    timeout=120,
)

async with executor:
    result = await executor.execute_code_blocks(
        code_blocks=[
            CodeBlock(language="python", code="import numpy as np; print(np.pi)"),
        ],
    )
```

### Jupyter执行器

```python
from autogen_ext.code_executors.jupyter import JupyterCodeExecutor

executor = JupyterCodeExecutor()

async with executor:
    result = await executor.execute_code_blocks(
        code_blocks=[
            CodeBlock(language="python", code="x = 1 + 1\nprint(x)"),
        ],
    )
```

## MCP工具

Model Context Protocol (MCP) 是一种标准化的工具集成协议。

### 使用MCP服务器

```python
from autogen_ext.tools.mcp import McpWorkbench, StdioServerParams
from autogen_agentchat.agents import AssistantAgent

# 定义服务器参数
server_params = StdioServerParams(
    command="npx",
    args=["@playwright/mcp@latest", "--headless"],
)

# 使用MCP工作台
async with McpWorkbench(server_params) as mcp:
    agent = AssistantAgent(
        "browser_agent",
        model_client=model_client,
        workbench=mcp,
        max_tool_iterations=10,
    )
    
    await agent.run(task="访问 https://example.com 并获取标题")
```

### 多MCP服务器

```python
from autogen_ext.tools.mcp import McpWorkbench, StdioServerParams

# 定义多个服务器
browser_params = StdioServerParams(
    command="npx",
    args=["@playwright/mcp@latest"],
)

file_params = StdioServerParams(
    command="npx", 
    args=["@modelcontextprotocol/server-filesystem", "./workspace"],
)

# 作为列表传递
async with McpWorkbench(browser_params) as browser_mcp:
    async with McpWorkbench(file_params) as file_mcp:
        agent = AssistantAgent(
            "multi_tool_agent",
            model_client=model_client,
            workbench=[browser_mcp, file_mcp],
        )
```

## OpenAI Assistant

使用OpenAI的Assistants API：

```python
from autogen_ext.agents.openai import OpenAIAssistantAgent

# 创建Assistant智能体
agent = OpenAIAssistantAgent(
    name="my_assistant",
    model="gpt-4o",
    instructions="你是一个有帮助的助手。",
    tools=[{"type": "code_interpreter"}],
    # api_key="your-api-key",  # 或使用环境变量
)

# 使用智能体
result = await agent.run(task="用Python计算1到100的和")
```

## 网页浏览

### MultimodalWebSurfer

```python
from autogen_ext.agents.web_surfer import MultimodalWebSurfer

agent = MultimodalWebSurfer(
    name="web_surfer",
    model_client=model_client,
)

result = await agent.run(
    task="搜索AutoGen的最新版本号"
)
```

## 文件处理

### 文件系统工具

```python
from autogen_ext.tools.file import FileSystemTools

file_tools = FileSystemTools(
    root_dir="./workspace",
    allowed_extensions=[".txt", ".py", ".md"],
)

agent = AssistantAgent(
    "file_agent",
    model_client=model_client,
    tools=file_tools.get_tools(),
)
```

## 自定义扩展

### 创建自定义模型客户端

```python
from autogen_core.models import (
    ChatCompletionClient,
    CreateResult,
    LLMMessage,
    ModelInfo,
    RequestUsage,
)
from typing import Sequence

class CustomModelClient(ChatCompletionClient):
    def __init__(self, api_key: str) -> None:
        self._api_key = api_key
    
    async def create(
        self,
        messages: Sequence[LLMMessage],
        **kwargs,
    ) -> CreateResult:
        # 实现你的API调用逻辑
        response_content = await self._call_api(messages)
        
        return CreateResult(
            finish_reason="stop",
            content=response_content,
            usage=RequestUsage(
                prompt_tokens=0,
                completion_tokens=0,
            ),
        )
    
    @property
    def model_info(self) -> ModelInfo:
        return ModelInfo(
            vision=False,
            function_calling=True,
            json_output=True,
            family="custom",
        )
```

### 创建自定义工具

```python
from autogen_core.tools import FunctionTool

def my_custom_tool(query: str, limit: int = 10) -> str:
    """自定义工具的描述。
    
    Args:
        query: 查询字符串
        limit: 结果数量限制
        
    Returns:
        查询结果
    """
    # 实现工具逻辑
    return f"查询 '{query}' 的前 {limit} 个结果..."

# 创建工具
tool = FunctionTool(my_custom_tool, description="执行自定义查询")

# 使用工具
agent = AssistantAgent(
    "tool_agent",
    model_client=model_client,
    tools=[tool],
)
```

### 创建自定义代码执行器

```python
from autogen_core.code_executor import CodeExecutor, CodeBlock, CodeResult

class CustomCodeExecutor(CodeExecutor):
    async def execute_code_blocks(
        self,
        code_blocks: list[CodeBlock],
        **kwargs,
    ) -> CodeResult:
        outputs = []
        for block in code_blocks:
            if block.language == "python":
                # 自定义执行逻辑
                output = await self._execute_python(block.code)
                outputs.append(output)
        
        return CodeResult(
            exit_code=0,
            output="\n".join(outputs),
        )
```

## 安装扩展

各扩展可以单独安装：

```bash
# OpenAI支持
pip install "autogen-ext[openai]"

# Azure支持（包含AAD认证）
pip install "autogen-ext[azure]"

# Anthropic支持
pip install "autogen-ext[anthropic]"

# Google支持
pip install "autogen-ext[google]"

# Ollama支持
pip install "autogen-ext[ollama]"

# Docker代码执行器
pip install "autogen-ext[docker]"

# Jupyter代码执行器
pip install "autogen-ext[jupyter]"

# 全部安装
pip install "autogen-ext[all]"
```

## 最佳实践

### 1. 资源管理

使用上下文管理器确保资源正确释放：

```python
async with McpWorkbench(server_params) as mcp:
    # 使用mcp
    pass
# 资源自动释放
```

### 2. 错误处理

处理扩展可能抛出的异常：

```python
from autogen_core.exceptions import ModelError

try:
    response = await model_client.create(messages=messages)
except ModelError as e:
    print(f"模型错误: {e}")
except Exception as e:
    print(f"未知错误: {e}")
```

### 3. 配置管理

使用环境变量管理敏感配置：

```python
import os

model_client = OpenAIChatCompletionClient(
    model="gpt-4o",
    api_key=os.environ.get("OPENAI_API_KEY"),
)
```

### 4. 日志记录

启用扩展日志以便调试：

```python
import logging

logging.getLogger("autogen_ext").setLevel(logging.DEBUG)
```

## 下一步

- 查看 [官方文档](https://microsoft.github.io/autogen/stable/user-guide/extensions-user-guide/index.html) 了解更多扩展
- 查看 [GitHub仓库](https://github.com/microsoft/autogen/tree/main/python/packages/autogen-ext) 了解扩展源码
- 参与 [社区讨论](https://github.com/microsoft/autogen/discussions) 分享你的自定义扩展
