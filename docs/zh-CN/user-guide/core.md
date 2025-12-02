# Core API 指南

Core API是AutoGen的底层框架，提供事件驱动的多智能体系统基础设施。

## 简介

Core API适用于：

- 需要精细控制的高级用户
- 构建自定义智能体行为
- 分布式智能体系统
- 跨语言应用（Python/.NET）

## 核心概念

### 运行时（Runtime）

运行时是智能体执行的环境。AutoGen提供两种运行时：

#### 单进程运行时

```python
from autogen_core import SingleThreadedAgentRuntime

runtime = SingleThreadedAgentRuntime()
```

#### 分布式运行时

使用gRPC进行分布式部署：

```python
from autogen_ext.runtimes.grpc import GrpcWorkerAgentRuntime

runtime = GrpcWorkerAgentRuntime(host_address="localhost:50051")
```

### 智能体类型（Agent Type）

定义智能体的类型和行为：

```python
from autogen_core import AgentType, TypeSubscription

# 注册智能体类型
await runtime.register_factory(
    type=AgentType("my_agent"),
    agent_factory=lambda: MyAgent(),
    expected_class=MyAgent,
)
```

### 消息（Message）

消息是智能体之间通信的载体：

```python
from dataclasses import dataclass

@dataclass
class MyMessage:
    content: str
    sender: str
```

### 订阅（Subscription）

订阅定义智能体如何接收消息：

```python
from autogen_core import TypeSubscription, DefaultTopicId

# 类型订阅
subscription = TypeSubscription(
    topic_type="my_topic",
    agent_type="my_agent"
)
await runtime.add_subscription(subscription)
```

## 创建自定义智能体

### 基础智能体

```python
from autogen_core import (
    BaseAgent,
    MessageContext,
    message_handler,
)
from dataclasses import dataclass

@dataclass
class TextMessage:
    content: str

@dataclass
class Response:
    content: str

class MyAgent(BaseAgent):
    def __init__(self) -> None:
        super().__init__("my_agent")
    
    @message_handler
    async def handle_text(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        # 处理消息
        return Response(content=f"收到: {message.content}")
```

### 注册和使用智能体

```python
from autogen_core import AgentId, SingleThreadedAgentRuntime

# 创建运行时
runtime = SingleThreadedAgentRuntime()

# 注册智能体工厂
await runtime.register_factory(
    type=AgentType("my_agent"),
    agent_factory=lambda: MyAgent(),
    expected_class=MyAgent,
)

# 启动运行时
runtime.start()

# 发送消息
agent_id = AgentId("my_agent", "default")
response = await runtime.send_message(
    message=TextMessage(content="你好！"),
    recipient=agent_id,
)
print(response.content)

# 停止运行时
await runtime.stop()
```

## 主题和发布/订阅

### 定义主题

```python
from autogen_core import TopicId, DefaultTopicId

# 默认主题
topic = DefaultTopicId()

# 自定义主题
topic = TopicId(type="my_topic", source="my_source")
```

### 发布消息

```python
from autogen_core import DefaultTopicId

# 发布到主题
await runtime.publish_message(
    message=TextMessage(content="广播消息"),
    topic_id=DefaultTopicId(),
)
```

### 订阅主题

```python
from autogen_core import TypeSubscription

# 添加订阅
await runtime.add_subscription(
    TypeSubscription(
        topic_type="news",
        agent_type="news_reader"
    )
)
```

## 消息处理

### 单一消息类型

```python
class SimpleAgent(BaseAgent):
    @message_handler
    async def handle_message(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        return Response(content=f"处理: {message.content}")
```

### 多消息类型

```python
@dataclass
class ImageMessage:
    url: str

@dataclass
class AudioMessage:
    data: bytes

class MultiModalAgent(BaseAgent):
    @message_handler
    async def handle_text(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        return Response(content=f"文本: {message.content}")
    
    @message_handler
    async def handle_image(
        self, message: ImageMessage, ctx: MessageContext
    ) -> Response:
        return Response(content=f"图片: {message.url}")
    
    @message_handler
    async def handle_audio(
        self, message: AudioMessage, ctx: MessageContext
    ) -> Response:
        return Response(content="音频已处理")
```

## 状态管理

### 智能体状态

```python
class StatefulAgent(BaseAgent):
    def __init__(self) -> None:
        super().__init__("stateful_agent")
        self._message_count = 0
        self._history: list[str] = []
    
    @message_handler
    async def handle_message(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        self._message_count += 1
        self._history.append(message.content)
        return Response(
            content=f"已处理 {self._message_count} 条消息"
        )
    
    async def save_state(self) -> dict:
        return {
            "message_count": self._message_count,
            "history": self._history,
        }
    
    async def load_state(self, state: dict) -> None:
        self._message_count = state.get("message_count", 0)
        self._history = state.get("history", [])
```

## 分布式运行时

### 设置gRPC运行时

```python
from autogen_ext.runtimes.grpc import GrpcWorkerAgentRuntime

# 工作进程
runtime = GrpcWorkerAgentRuntime(
    host_address="localhost:50051"
)

# 注册智能体
await runtime.register_factory(
    type=AgentType("worker_agent"),
    agent_factory=lambda: WorkerAgent(),
    expected_class=WorkerAgent,
)

# 启动运行时
runtime.start()
```

### 网关配置

```python
from autogen_ext.runtimes.grpc import GrpcGatewayRuntime

# 网关
gateway = GrpcGatewayRuntime(
    host_address="localhost:50051"
)
await gateway.start()
```

## 错误处理

### 消息处理错误

```python
from autogen_core import CancellationToken

class RobustAgent(BaseAgent):
    @message_handler
    async def handle_message(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        try:
            # 处理消息
            result = await self.process(message)
            return Response(content=result)
        except ValueError as e:
            return Response(content=f"验证错误: {e}")
        except Exception as e:
            return Response(content=f"处理错误: {e}")
```

### 取消操作

```python
from autogen_core import CancellationToken

# 创建取消令牌
cancellation_token = CancellationToken()

# 发送可取消的消息
response = await runtime.send_message(
    message=TextMessage(content="长时间任务"),
    recipient=agent_id,
    cancellation_token=cancellation_token,
)

# 取消操作
cancellation_token.cancel()
```

## 性能优化

### 批量处理

```python
class BatchAgent(BaseAgent):
    def __init__(self) -> None:
        super().__init__("batch_agent")
        self._buffer: list[TextMessage] = []
    
    @message_handler
    async def handle_message(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        self._buffer.append(message)
        
        if len(self._buffer) >= 10:
            # 批量处理
            await self.process_batch(self._buffer)
            self._buffer.clear()
        
        return Response(content="已加入队列")
```

### 并发控制

```python
import asyncio
from asyncio import Semaphore

class ConcurrentAgent(BaseAgent):
    def __init__(self, max_concurrent: int = 5) -> None:
        super().__init__("concurrent_agent")
        self._semaphore = Semaphore(max_concurrent)
    
    @message_handler
    async def handle_message(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        async with self._semaphore:
            # 限制并发
            result = await self.process(message)
            return Response(content=result)
```

## 最佳实践

### 1. 使用数据类定义消息

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class OrderMessage:
    order_id: str
    customer_id: str
    items: list[str]
    total: float
    notes: Optional[str] = None
```

### 2. 分离关注点

```python
# 消息验证器
class Validator:
    def validate_order(self, order: OrderMessage) -> bool:
        return len(order.items) > 0 and order.total > 0

# 业务逻辑处理器
class OrderProcessor:
    async def process(self, order: OrderMessage) -> str:
        # 处理订单
        return f"订单 {order.order_id} 已处理"

# 智能体使用组合模式
class OrderAgent(BaseAgent):
    def __init__(self) -> None:
        super().__init__("order_agent")
        self._validator = Validator()
        self._processor = OrderProcessor()
```

### 3. 实现适当的日志记录

```python
import logging

logger = logging.getLogger(__name__)

class LoggingAgent(BaseAgent):
    @message_handler
    async def handle_message(
        self, message: TextMessage, ctx: MessageContext
    ) -> Response:
        logger.info(f"收到消息: {message.content}")
        try:
            result = await self.process(message)
            logger.info(f"处理完成: {result}")
            return Response(content=result)
        except Exception as e:
            logger.error(f"处理失败: {e}")
            raise
```

## 下一步

- 查看 [Extensions指南](./extensions.md) 了解可用的扩展
- 查看 [官方API文档](https://microsoft.github.io/autogen/stable/reference/index.html) 获取完整API参考
