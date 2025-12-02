# DeepSeek 多智能体新手教程

本教程面向第一次接触 AutoGen 的开发者，带你从零搭建一个可迁移的 DeepSeek 多智能体项目。我们按照官方文档（<https://microsoft.github.io/autogen/stable/index.html>）的推荐流程，提供逐步操作说明，并补充 AutoGen Studio 的使用提示。照着做即可完成第一个多智能体 Demo。

## 1. 新建工程骨架

### 1.1 安装基础工具

1. 打开终端，确认当前 shell：

    ```zsh
    echo $SHELL
    ```

    输出包含 `zsh` 表示正在使用默认环境。

2. 安装 uv（统一管理 Python 依赖）：

    ```zsh
    python3 -m pip install --upgrade uv
    ```

3. 可选：安装 .NET SDK 和运行时，方便后续运行 .NET 示例或工具（每条命令执行完再运行下一条）：

    ```zsh
    wget https://dot.net/v1/dotnet-install.sh
    chmod +x dotnet-install.sh
    ./dotnet-install.sh --channel 9.0
    ./dotnet-install.sh --channel 8.0 --runtime dotnet
    ./dotnet-install.sh --channel 8.0 --runtime aspnetcore
    export PATH="$HOME/.dotnet:$PATH"
    ```

    提示：.NET 并非本教程必需，但一次性安装可以避免后续使用 AutoGen 生态工具时遇到缺少运行时的报错。

### 1.2 使用 uv 初始化项目

1. 选择一个空目录作为工作区，例如桌面：

    ```zsh
    cd ~/Desktop
    ```

2. 新建项目文件夹并进入：

    ```zsh
    mkdir deepseek-multi-agent
    cd deepseek-multi-agent
    ```

3. 初始化 Python 工程骨架（会生成 `pyproject.toml` 和 `.venv`）：

    ```zsh
    uv init --name deepseek_multi_agent
    ```

4. 安装 AutoGen 及其扩展依赖：

    ```zsh
    uv add "autogen-agentchat" "autogen-ext[openai]" "python-dotenv"
    ```

5. 如需图形化开发，再安装 AutoGen Studio：

    ```zsh
    uv add autogen-studio
    ```

6. 验证依赖是否安装完成：

    ```zsh
    uv pip list | grep autogen
    ```

    看到 `autogen-agentchat` 和 `autogen-ext` 等条目说明环境准备就绪。项目目录此时大致包含 `.venv/`、`pyproject.toml`、`uv.lock` 等文件。

## 2. 管理 DeepSeek 凭证

1. 登录 [DeepSeek 开放平台](https://platform.deepseek.com/)，创建 API Key。
2. 在项目根目录创建 `.env` 文件（复制整段命令执行即可）：

    ```zsh
    cat <<'ENV' > .env
    DEEPSEEK_API_KEY=你的-deepseek-api-key
    ENV
    ```

    如果更习惯使用编辑器，也可以在 VS Code 中新建 `.env` 并写入同样内容。请勿将 `.env` 提交到版本库。

3. 检查 `.env` 是否写入成功：

    ```zsh
    cat .env
    ```

4. 在当前 shell 中加载环境变量：

    ```zsh
    export $(cat .env | xargs)
    ```

    生产环境建议改用密钥管理服务或系统级环境变量，而不是直接在 shell 中导出。

## 3. 编写多智能体脚本

> 采用 `src/` 目录结构，便于后续迁移或打包。

1. 创建源代码目录：

    ```zsh
    mkdir -p src/deepseek_multi_agent
    ```

2. 创建并打开 `src/deepseek_multi_agent/app.py`（未配置 VS Code 命令行时，可以手动打开文件）：

    ```zsh
    code src/deepseek_multi_agent/app.py
    ```

3. 粘贴以下示例代码并保存：

```python
"""DeepSeek 多智能体项目的入口脚本。"""
import asyncio
import os
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.conditions import MaxMessageTermination
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.ui import Console
from autogen_core.models import ModelFamily, ModelInfo
from autogen_ext.models.openai import OpenAIChatCompletionClient


def build_model_client() -> OpenAIChatCompletionClient:
    """创建指向 DeepSeek 的模型客户端。"""
    api_key = os.environ.get("DEEPSEEK_API_KEY")
    if not api_key:
        raise RuntimeError("请先在环境变量中设置 DEEPSEEK_API_KEY")

    return OpenAIChatCompletionClient(
        model="deepseek-chat",
        base_url="https://api.deepseek.com",
        api_key=api_key,
        model_info=ModelInfo(
            vision=False,
            function_calling=True,
            json_output=True,
            family=ModelFamily.UNKNOWN,
            structured_output=True,
        ),
    )


def build_team(model_client: OpenAIChatCompletionClient) -> RoundRobinGroupChat:
    """组建由程序员与代码审查者构成的回合制团队。"""
    programmer = AssistantAgent(
        name="programmer",
        model_client=model_client,
        system_message="你是资深 Python 工程师，负责编写可靠、易维护的代码，并用代码块展示结果。",
    )

    reviewer = AssistantAgent(
        name="reviewer",
        model_client=model_client,
        system_message="你是严格的代码审查者，负责指出潜在问题并提出改进建议。",
    )

    return RoundRobinGroupChat(
        participants=[programmer, reviewer],
        termination_condition=MaxMessageTermination(max_messages=6),
    )


async def main() -> None:
    """运行多智能体协作流程。"""
    model_client = build_model_client()
    team = build_team(model_client)

    await Console(
        team.run_stream(task="请编写一个函数，判断字符串是否为回文，并提供测试样例。")
    )

    await model_client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

4. 在终端回到项目根目录，运行示例：

    ```zsh
    uv run python -m deepseek_multi_agent.app
    ```

    第一次运行会自动安装缺失依赖并启动会话，稍作等待即可看到 `programmer` 与 `reviewer` 两位智能体轮流对话、输出代码和反馈，表示多智能体流程已经跑通。

## 4. 深化与迁移

- **切换模型**：改用 `deepseek-reasoner` 时，把 `family` 设置为 `ModelFamily.R1` 以启用链式思考；迁移到其他 OpenAI 兼容服务时，调整 `base_url`、`model` 和 `model_info`。
- **接入工具**：在 `AssistantAgent` 中传入 `tools=[...]`，即可让 DeepSeek 调用自定义协程，扩展外部数据或动作能力。
- **扩展调度**：`RoundRobinGroupChat` 适合入门；复杂项目可以尝试 `HierarchicalGroupChat` 或自定义调度器实现多阶段协作。
- **部署迁移**：使用 `uv export --frozen --no-hashes > requirements.txt` 或直接携带 `uv.lock`，在目标环境中运行 `uv sync` 就能复现依赖。

## 5. 与 AutoGen Studio 并行使用

AutoGen Studio 提供可视化的多智能体编排界面，可与纯 Python 脚本共用同一套依赖和凭证。

```zsh
uv run autogenstudio ui
```

- 在 Studio 中快速搭建和验证工作流；
- 将验证过的配置导出为 Python 代码或 YAML，再整合进项目代码库；
- 共享 `.env` 与 `uv` 环境，避免重复维护密钥。

如果浏览器没有自动打开，可手动访问终端提示的地址（默认 `http://localhost:8080`）。首次使用可以选择 Guest 模式快速体验。

## 6. 开发质量自检

在提交代码或部署前，建议执行：

```zsh
uv run ruff check src
uv run ruff format --check src
uv run mypy src
```

若使用 GitHub CI，可直接在工作流中复用这些 `uv run` 命令。

## 7. 常见问题

| 场景 | 排查建议 |
| ---- | -------- |
| `ModuleNotFoundError` | 执行 `uv sync` 并使用 `uv run` 或 `source .venv/bin/activate` 后再运行脚本。 |
| DeepSeek 返回 401 | 检查 `.env` 中的密钥是否正确，确认账户余额充足，并留意调用频率限制。 |
| 首次调用耗时较长 | 第一次请求需要网络握手，等待几秒即可；若持续缓慢，检查网络代理或防火墙设置。 |
| 迁移到服务器/容器 | 携带 `.env` 和 `uv.lock`，在目标环境运行 `uv sync`；如需 .NET 功能，请提前安装对应运行时。 |

想要快速确认 DeepSeek 与网络配置是否可用，可以执行以下两步：

1. 在终端粘贴整段命令（包含末尾的 `PY` 行）：

    ```zsh
    uv run python - <<'PY'
    import asyncio
    import os
    from autogen_ext.models.openai import OpenAIChatCompletionClient
    from autogen_core.models import ModelFamily, ModelInfo


    async def main() -> None:
        client = OpenAIChatCompletionClient(
            model="deepseek-chat",
            base_url="https://api.deepseek.com",
            api_key=os.environ["DEEPSEEK_API_KEY"],
            model_info=ModelInfo(
                vision=False,
                function_calling=True,
                json_output=True,
                family=ModelFamily.UNKNOWN,
                structured_output=True,
            ),
        )
        result = await client.create_completion(
            messages=[{"role": "user", "content": "DeepSeek hello test"}]
        )
        print(result)
        await client.close()


    asyncio.run(main())
    PY
    ```

2. 若终端输出包含 DeepSeek 的回答，说明密钥和网络配置正常；如果报错，请重新检查 `.env` 或网络代理。

## 8. 下一步

- 阅读官方的 [AgentChat 用户指南](https://microsoft.github.io/autogen/stable/user-guide/agentchat.html)，了解更多智能体编排方式。
- 在团队中尝试使用 AutoGen Studio，将可视化工作流导出为代码，实现设计与实现并行。
- 根据业务场景接入数据库、工具调用或工作流编排，引导智能体完成更复杂的任务。

祝你顺利完成基于 AutoGen 与 DeepSeek 的多智能体项目！
