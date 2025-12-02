<a name="readme-top"></a>

<div align="center">
<img src="https://microsoft.github.io/autogen/0.2/img/ag.svg" alt="AutoGen Logo" width="100">

[![Twitter](https://img.shields.io/twitter/url/https/twitter.com/cloudposse.svg?style=social&label=Follow%20%40pyautogen)](https://twitter.com/pyautogen)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Company?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/company/105812540)
[![Discord](https://img.shields.io/badge/discord-chat-green?logo=discord)](https://aka.ms/autogen-discord)
[![Documentation](https://img.shields.io/badge/Documentation-AutoGen-blue?logo=read-the-docs)](https://microsoft.github.io/autogen/)
[![Blog](https://img.shields.io/badge/Blog-AutoGen-blue?logo=blogger)](https://devblogs.microsoft.com/autogen/)

[English](./README.md) | [中文](./README_zh-CN.md)

</div>

# AutoGen

**AutoGen** 是一个用于创建多智能体AI应用程序的框架，这些应用可以自主运行或与人类协作完成任务。

> **重要提示:** 如果您是AutoGen的新用户，请查看 [Microsoft Agent Framework](https://github.com/microsoft/agent-framework)。
> AutoGen仍将持续维护，并继续接收错误修复和关键安全补丁。
> 阅读我们的 [公告](https://github.com/microsoft/autogen/discussions/7066)。

## 安装

AutoGen 需要 **Python 3.10 或更高版本**。

```bash
# 从Extensions安装AgentChat和OpenAI客户端
pip install -U "autogen-agentchat" "autogen-ext[openai]"
```

当前稳定版本可以在 [releases](https://github.com/microsoft/autogen/releases) 页面找到。如果您正在从AutoGen v0.2升级，请参阅 [迁移指南](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/migration-guide.html) 了解如何更新您的代码和配置的详细说明。

```bash
# 安装AutoGen Studio（无代码GUI界面）
pip install -U "autogenstudio"
```

## 快速入门

以下示例调用OpenAI API，因此您首先需要创建一个账户并导出您的密钥：`export OPENAI_API_KEY="sk-..."`。

### Hello World

使用OpenAI的GPT-4o模型创建一个助手智能体。查看 [其他支持的模型](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/models.html)。

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4.1")
    agent = AssistantAgent("assistant", model_client=model_client)
    print(await agent.run(task="Say 'Hello World!'"))
    await model_client.close()

asyncio.run(main())
```

### MCP服务器

创建一个使用Playwright MCP服务器的网页浏览助手智能体。

```python
# 首先运行 `npm install -g @playwright/mcp@latest` 来安装MCP服务器。
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.tools.mcp import McpWorkbench, StdioServerParams


async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4.1")
    server_params = StdioServerParams(
        command="npx",
        args=[
            "@playwright/mcp@latest",
            "--headless",
        ],
    )
    async with McpWorkbench(server_params) as mcp:
        agent = AssistantAgent(
            "web_browsing_assistant",
            model_client=model_client,
            workbench=mcp, # 如果有多个MCP服务器，将它们放在一个列表中。
            model_client_stream=True,
            max_tool_iterations=10,
        )
        await Console(agent.run_stream(task="查找microsoft/autogen仓库有多少贡献者"))


asyncio.run(main())
```

> **警告**: 仅连接可信任的MCP服务器，因为它们可能在您的本地环境中执行命令或暴露敏感信息。

### 多智能体协调

您可以使用 `AgentTool` 创建一个基本的多智能体协调设置。

```python
import asyncio

from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.tools import AgentTool
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import OpenAIChatCompletionClient


async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4.1")

    math_agent = AssistantAgent(
        "math_expert",
        model_client=model_client,
        system_message="你是一位数学专家。",
        description="一个数学专家助手。",
        model_client_stream=True,
    )
    math_agent_tool = AgentTool(math_agent, return_value_as_last_message=True)

    chemistry_agent = AssistantAgent(
        "chemistry_expert",
        model_client=model_client,
        system_message="你是一位化学专家。",
        description="一个化学专家助手。",
        model_client_stream=True,
    )
    chemistry_agent_tool = AgentTool(chemistry_agent, return_value_as_last_message=True)

    agent = AssistantAgent(
        "assistant",
        system_message="你是一个通用助手。在需要时使用专家工具。",
        model_client=model_client,
        model_client_stream=True,
        tools=[math_agent_tool, chemistry_agent_tool],
        max_tool_iterations=10,
    )
    await Console(agent.run_stream(task="x^2的积分是多少？"))
    await Console(agent.run_stream(task="水的分子量是多少？"))


asyncio.run(main())
```

更多高级的多智能体协调和工作流程，请阅读 [AgentChat文档](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)。

### 使用DeepSeek模型

AutoGen支持使用DeepSeek等国产模型。DeepSeek提供OpenAI兼容的API接口：

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_core.models import ModelInfo, ModelFamily

async def main() -> None:
    # 使用DeepSeek模型
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
    agent = AssistantAgent("assistant", model_client=model_client)
    print(await agent.run(task="请介绍一下你自己"))
    await model_client.close()

asyncio.run(main())
```

更多DeepSeek使用详情，请参阅 [中文文档](./docs/zh-CN/user-guide/extensions.md#deepseek)。

### AutoGen Studio

使用AutoGen Studio无需编写代码即可原型设计和运行多智能体工作流程。

```bash
# 在 http://localhost:8080 运行AutoGen Studio
autogenstudio ui --port 8080 --appdir ./my-app
```

## 为什么使用AutoGen？

<div align="center">
  <img src="autogen-landing.jpg" alt="AutoGen Landing" width="500">
</div>

AutoGen生态系统提供了创建AI智能体所需的一切，特别是多智能体工作流程——包括框架、开发工具和应用程序。

_框架_ 采用分层和可扩展的设计。各层有明确的职责划分，并建立在下层之上。这种设计使您能够在不同的抽象级别使用框架，从高级API到低级组件。

- [Core API](./python/packages/autogen-core/) 实现了消息传递、事件驱动的智能体，以及用于灵活性和功能的本地和分布式运行时。它还支持.NET和Python的跨语言支持。
- [AgentChat API](./python/packages/autogen-agentchat/) 实现了一个更简单但有主见的API，用于快速原型设计。此API构建在Core API之上，与v0.2用户熟悉的方式最接近，支持常见的多智能体模式，如两智能体聊天或群聊。
- [Extensions API](./python/packages/autogen-ext/) 支持第一方和第三方扩展，不断扩展框架功能。它支持LLM客户端（如OpenAI、AzureOpenAI）的特定实现，以及代码执行等功能。

生态系统还支持两个重要的 _开发工具_：

<div align="center">
  <img src="https://media.githubusercontent.com/media/microsoft/autogen/refs/heads/main/python/packages/autogen-studio/docs/ags_screen.png" alt="AutoGen Studio Screenshot" width="500">
</div>

- [AutoGen Studio](./python/packages/autogen-studio/) 提供了一个无代码GUI，用于构建多智能体应用程序。
- [AutoGen Bench](./python/packages/agbench/) 提供了一个基准测试套件，用于评估智能体性能。

您可以使用AutoGen框架和开发工具为您的领域创建应用程序。例如，[Magentic-One](./python/packages/magentic-one-cli/) 是一个使用AgentChat API和Extensions API构建的最先进的多智能体团队，可以处理各种需要网页浏览、代码执行和文件处理的任务。

使用AutoGen，您可以加入并为一个蓬勃发展的生态系统做出贡献。我们每周举办与维护者和社区的办公时间和讲座。我们还有一个用于实时聊天的 [Discord服务器](https://aka.ms/autogen-discord)、用于问答的GitHub Discussions，以及用于教程和更新的博客。

## 接下来去哪里？

<div align="center">

|               | [![Python](https://img.shields.io/badge/AutoGen-Python-blue?logo=python&logoColor=white)](./python)                                                                                                                                                                                                                                                                                                                | [![.NET](https://img.shields.io/badge/AutoGen-.NET-green?logo=.net&logoColor=white)](./dotnet)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | [![Studio](https://img.shields.io/badge/AutoGen-Studio-purple?logo=visual-studio&logoColor=white)](./python/packages/autogen-studio)                        |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 安装           | [![Installation](https://img.shields.io/badge/Install-blue)](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/installation.html)                                                                                                                                                                                                                                                         | [![Install](https://img.shields.io/badge/Install-green)](https://microsoft.github.io/autogen/dotnet/dev/core/installation.html)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | [![Install](https://img.shields.io/badge/Install-purple)](https://microsoft.github.io/autogen/stable/user-guide/autogenstudio-user-guide/installation.html) |
| 快速入门        | [![Quickstart](https://img.shields.io/badge/Quickstart-blue)](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/quickstart.html#)                                                                                                                                                                                                                                                         | [![Quickstart](https://img.shields.io/badge/Quickstart-green)](https://microsoft.github.io/autogen/dotnet/dev/core/index.html)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | [![Usage](https://img.shields.io/badge/Quickstart-purple)](https://microsoft.github.io/autogen/stable/user-guide/autogenstudio-user-guide/usage.html#)      |
| 教程           | [![Tutorial](https://img.shields.io/badge/Tutorial-blue)](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/index.html)                                                                                                                                                                                                                                                          | [![Tutorial](https://img.shields.io/badge/Tutorial-green)](https://microsoft.github.io/autogen/dotnet/dev/core/tutorial.html)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | [![Usage](https://img.shields.io/badge/Tutorial-purple)](https://microsoft.github.io/autogen/stable/user-guide/autogenstudio-user-guide/usage.html#)        |
| API参考        | [![API](https://img.shields.io/badge/Docs-blue)](https://microsoft.github.io/autogen/stable/reference/index.html#)                                                                                                                                                                                                                                                                                                 | [![API](https://img.shields.io/badge/Docs-green)](https://microsoft.github.io/autogen/dotnet/dev/api/Microsoft.AutoGen.Contracts.html)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | [![API](https://img.shields.io/badge/Docs-purple)](https://microsoft.github.io/autogen/stable/user-guide/autogenstudio-user-guide/usage.html)               |
| 包             | [![PyPi autogen-core](https://img.shields.io/badge/PyPi-autogen--core-blue?logo=pypi)](https://pypi.org/project/autogen-core/) <br> [![PyPi autogen-agentchat](https://img.shields.io/badge/PyPi-autogen--agentchat-blue?logo=pypi)](https://pypi.org/project/autogen-agentchat/) <br> [![PyPi autogen-ext](https://img.shields.io/badge/PyPi-autogen--ext-blue?logo=pypi)](https://pypi.org/project/autogen-ext/) | [![NuGet Contracts](https://img.shields.io/badge/NuGet-Contracts-green?logo=nuget)](https://www.nuget.org/packages/Microsoft.AutoGen.Contracts/) <br> [![NuGet Core](https://img.shields.io/badge/NuGet-Core-green?logo=nuget)](https://www.nuget.org/packages/Microsoft.AutoGen.Core/) <br> [![NuGet Core.Grpc](https://img.shields.io/badge/NuGet-Core.Grpc-green?logo=nuget)](https://www.nuget.org/packages/Microsoft.AutoGen.Core.Grpc/) <br> [![NuGet RuntimeGateway.Grpc](https://img.shields.io/badge/NuGet-RuntimeGateway.Grpc-green?logo=nuget)](https://www.nuget.org/packages/Microsoft.AutoGen.RuntimeGateway.Grpc/) | [![PyPi autogenstudio](https://img.shields.io/badge/PyPi-autogenstudio-purple?logo=pypi)](https://pypi.org/project/autogenstudio/)                          |

</div>

有兴趣贡献？请参阅 [CONTRIBUTING.md](./CONTRIBUTING.md) 了解如何开始的指南。我们欢迎各种类型的贡献，包括错误修复、新功能和文档改进。加入我们的社区，帮助我们让AutoGen变得更好！

有问题？查看我们的 [常见问题解答（FAQ）](./FAQ.md) 了解常见问题的答案。如果您没有找到您要找的内容，请随时在我们的 [GitHub Discussions](https://github.com/microsoft/autogen/discussions) 中提问，或加入我们的 [Discord服务器](https://aka.ms/autogen-discord) 获取实时支持。您也可以阅读我们的 [博客](https://devblogs.microsoft.com/autogen/) 获取更新。

## 法律声明

Microsoft和任何贡献者授予您Microsoft文档和本仓库中其他内容的许可证，根据 [知识共享署名4.0国际公共许可证](https://creativecommons.org/licenses/by/4.0/legalcode)，
请参阅 [LICENSE](LICENSE) 文件，并授予您仓库中任何代码的许可证，根据 [MIT许可证](https://opensource.org/licenses/MIT)，请参阅
[LICENSE-CODE](LICENSE-CODE) 文件。

Microsoft、Windows、Microsoft Azure和/或文档中引用的其他Microsoft产品和服务
可能是Microsoft在美国和/或其他国家的商标或注册商标。
本项目的许可证不授予您使用任何Microsoft名称、徽标或商标的权利。
Microsoft的一般商标指南可以在 <http://go.microsoft.com/fwlink/?LinkID=254653> 找到。

隐私信息可以在 <https://go.microsoft.com/fwlink/?LinkId=521839> 找到

Microsoft和任何贡献者保留所有其他权利，无论是根据其各自的版权、专利、
或商标，无论是暗示的、禁止反言的还是其他方式。

<p align="right" style="font-size: 14px; color: #555; margin-top: 20px;">
  <a href="#readme-top" style="text-decoration: none; color: blue; font-weight: bold;">
    ↑ 返回顶部 ↑
  </a>
</p>
