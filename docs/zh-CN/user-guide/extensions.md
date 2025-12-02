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
