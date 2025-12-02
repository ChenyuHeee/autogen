# 安装指南

本文档介绍如何安装AutoGen框架。

## 系统要求

- **Python 3.10 或更高版本**

## 创建虚拟环境（推荐）

在本地安装AgentChat时，我们建议使用虚拟环境进行安装。这将确保AgentChat的依赖项与系统的其他部分隔离。

### 使用venv

**Linux/Mac:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

**Windows 命令行:**

```batch
# 根据您的设置，命令可能是 `python3` 而不是 `python`
python -m venv .venv
.venv\Scripts\activate.bat
```

要停用虚拟环境，运行:

```bash
deactivate
```

### 使用conda

首先 [安装Conda](https://docs.conda.io/projects/conda/en/stable/user-guide/install/index.html)（如果尚未安装）。

创建并激活环境:

```bash
conda create -n autogen python=3.12
conda activate autogen
```

要停用环境，运行:

```bash
conda deactivate
```

## 使用pip安装

使用pip安装 `autogen-agentchat` 包:

```bash
pip install -U "autogen-agentchat"
```

```{note}
需要Python 3.10或更高版本。
```

## 安装OpenAI模型客户端

要使用OpenAI和Azure OpenAI模型，您需要安装以下扩展:

```bash
pip install "autogen-ext[openai]"
```

如果您使用带有AAD身份验证的Azure OpenAI，需要安装以下内容:

```bash
pip install "autogen-ext[azure]"
```

## 其他模型支持

AutoGen支持多种模型提供商。以下是一些常用的安装命令:

### Anthropic Claude

```bash
pip install "autogen-ext[anthropic]"
```

### Google Gemini

```bash
pip install "autogen-ext[google]"
```

### Ollama（本地模型）

```bash
pip install "autogen-ext[ollama]"
```

## 安装AutoGen Studio

AutoGen Studio是一个无代码GUI界面，用于构建多智能体应用程序:

```bash
pip install -U "autogenstudio"
```

启动Studio:

```bash
autogenstudio ui --port 8080 --appdir ./myapp
```

## 验证安装

安装完成后，您可以通过以下Python代码验证安装:

```python
import autogen_agentchat
print(f"AutoGen AgentChat version: {autogen_agentchat.__version__}")
```

## 常见问题

### 1. 安装失败

确保您使用的是Python 3.10或更高版本:

```bash
python --version
```

### 2. 导入错误

确保您已激活虚拟环境，并且所有依赖项都已正确安装。

### 3. API密钥配置

使用OpenAI时，需要设置环境变量:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

## 下一步

安装完成后，请继续阅读 [快速入门](./quickstart.md) 来创建您的第一个智能体。
