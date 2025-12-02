# AutoGen 中文文档

欢迎阅读AutoGen中文文档！本文档帮助您快速了解和使用AutoGen框架。

## 目录

- [安装指南](./installation.md)
- [快速入门](./quickstart.md)
- [用户指南](./user-guide/index.md)

## 什么是AutoGen？

AutoGen是一个用于创建多智能体AI应用程序的框架。这些应用可以自主运行或与人类协作完成任务。

## 主要特性

- **多智能体协作**: 创建多个AI智能体协同工作
- **事件驱动架构**: 基于消息传递的灵活设计
- **可扩展性**: 支持本地和分布式运行时
- **跨语言支持**: 支持Python和.NET

## 快速开始

### 安装

```bash
# 安装AgentChat和OpenAI客户端
pip install -U "autogen-agentchat" "autogen-ext[openai]"
```

### Hello World示例

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4o")
    agent = AssistantAgent("assistant", model_client=model_client)
    print(await agent.run(task="Say 'Hello World!'"))
    await model_client.close()

asyncio.run(main())
```

## 文档链接

- [官方英文文档](https://microsoft.github.io/autogen/)
- [GitHub仓库](https://github.com/microsoft/autogen)
- [Discord社区](https://aka.ms/autogen-discord)
