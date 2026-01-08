# A2A 协议模块开发指南

本文档详细介绍 OpenManus 中 A2A（Agent-to-Agent）协议的实现逻辑、数据流和二次开发指南。

## 概述

A2A 协议是 Google 推出的代理间通信标准协议，允许不同的 AI 代理之间进行互操作。本模块将 OpenManus 的 Manus 代理封装为符合 A2A 协议的服务，使其可以被其他遵循 A2A 协议的客户端或代理调用。

**官方文档**: https://google.github.io/A2A/#/documentation

## 目录结构

```
protocol/a2a/
├── __init__.py
├── CLAUDE.md                 # 本文档
└── app/
    ├── __init__.py
    ├── main.py               # 服务器入口和配置
    ├── agent.py              # A2AManus 代理封装
    ├── agent_executor.py     # 请求执行器
    ├── README.md             # 英文使用说明
    └── README_zh.md          # 中文使用说明
```

## 核心组件

### 1. A2AManus (`agent.py`)

A2AManus 是 Manus 代理的 A2A 协议封装层，继承自 `app.agent.manus.Manus`。

```python
class A2AManus(Manus):
    SUPPORTED_CONTENT_TYPES: ClassVar[List[str]] = ["text", "text/plain"]

    async def invoke(self, query, sessionId) -> str:
        """执行单次任务调用"""

    async def stream(self, query: str) -> AsyncIterable[Dict[str, Any]]:
        """流式响应（当前未实现）"""

    def get_agent_response(self, config, agent_response):
        """格式化代理响应"""
```

**关键方法说明**:

| 方法 | 功能 | 返回值 |
|------|------|--------|
| `invoke()` | 执行任务并返回结果 | 包含 `is_task_complete`, `require_user_input`, `content` 的字典 |
| `stream()` | 流式响应（未实现） | 抛出 `NotImplementedError` |
| `get_agent_response()` | 将代理执行结果转换为标准响应格式 | 格式化的响应字典 |

**响应格式**:
```python
{
    "is_task_complete": True,      # 任务是否完成
    "require_user_input": False,   # 是否需要用户输入
    "content": "..."               # 实际响应内容
}
```

### 2. ManusExecutor (`agent_executor.py`)

ManusExecutor 是 A2A 协议的请求执行器，负责处理传入的任务请求。

```python
class ManusExecutor(AgentExecutor):
    def __init__(self, agent_factory: Callable[[], Awaitable[A2AManus]]):
        """初始化执行器，接收代理工厂函数"""

    async def execute(self, context: RequestContext, event_queue: EventQueue) -> None:
        """执行任务请求"""

    def _validate_request(self, context: RequestContext) -> bool:
        """验证请求有效性"""

    async def cancel(self, request: RequestContext, event_queue: EventQueue) -> Task | None:
        """取消任务（未实现）"""
```

**执行流程**:
1. 验证请求参数
2. 从上下文中提取用户输入
3. 通过工厂函数创建代理实例
4. 调用代理的 `invoke()` 方法执行任务
5. 将结果封装为 A2A 协议格式的响应
6. 通过事件队列发送完成事件

### 3. 服务器入口 (`main.py`)

服务器入口负责配置和启动 A2A 服务。

**核心组件配置**:

```python
# 代理能力声明
capabilities = AgentCapabilities(
    streaming=False,           # 当前不支持流式
    pushNotifications=True     # 支持推送通知
)

# 技能列表
skills = [
    AgentSkill(id="Python Execute", ...),
    AgentSkill(id="Browser use", ...),
    AgentSkill(id="Replace String", ...),
    AgentSkill(id="Ask human", ...),
    AgentSkill(id="terminate", ...),
]

# 代理名片
agent_card = AgentCard(
    name="Manus Agent",
    description="...",
    url=f"http://{host}:{port}/",
    version="1.0.0",
    capabilities=capabilities,
    skills=skills,
)
```

## 数据流

### 请求处理流程

```
┌─────────────┐     HTTP/JSON-RPC      ┌──────────────────────┐
│  A2A Client │ ──────────────────────▶│ A2AStarletteApplication │
└─────────────┘                        └──────────┬───────────┘
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │ DefaultRequestHandler │
                                       └──────────┬───────────┘
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │   ManusExecutor      │
                                       │   execute()          │
                                       └──────────┬───────────┘
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │   A2AManus           │
                                       │   invoke()           │
                                       └──────────┬───────────┘
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │   Manus.run()        │
                                       │   (执行实际任务)       │
                                       └──────────┬───────────┘
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │   EventQueue         │
                                       │   (完成事件入队)       │
                                       └──────────┬───────────┘
                                                  │
                                                  ▼
┌─────────────┐     HTTP Response      ┌──────────────────────┐
│  A2A Client │ ◀──────────────────────│   Task Result        │
└─────────────┘                        └──────────────────────┘
```

### 详细数据流说明

1. **请求接收阶段**
   - 客户端发送 JSON-RPC 请求到 `http://host:port/`
   - `A2AStarletteApplication` 解析请求并路由到 `DefaultRequestHandler`

2. **请求处理阶段**
   - `DefaultRequestHandler` 创建 `RequestContext` 和 `EventQueue`
   - 调用 `ManusExecutor.execute()` 处理请求

3. **任务执行阶段**
   - `ManusExecutor` 验证请求参数
   - 通过工厂函数 `agent_factory()` 创建 `A2AManus` 实例
   - 调用 `A2AManus.invoke(query, sessionId)` 执行任务
   - `invoke()` 内部调用继承自 `Manus` 的 `run()` 方法

4. **响应生成阶段**
   - 将执行结果封装为 `TextPart`
   - 创建 `Artifact` 包含响应内容
   - 通过 `completed_task()` 生成完成任务事件
   - 将事件加入 `EventQueue`

5. **响应返回阶段**
   - `InMemoryTaskStore` 存储任务状态
   - `InMemoryPushNotifier` 处理推送通知
   - 返回 JSON-RPC 响应给客户端

## A2A 协议核心类型

### 从 `a2a-sdk` 导入的类型

```python
# 服务器组件
from a2a.server.apps import A2AStarletteApplication
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.tasks import InMemoryPushNotifier, InMemoryTaskStore
from a2a.server.agent_execution import AgentExecutor, RequestContext
from a2a.server.events import EventQueue

# 数据类型
from a2a.types import (
    AgentCapabilities,    # 代理能力声明
    AgentCard,            # 代理名片
    AgentSkill,           # 代理技能
    Part,                 # 消息部分
    TextPart,             # 文本消息
    Task,                 # 任务对象
    InvalidParamsError,   # 参数无效错误
    UnsupportedOperationError,  # 不支持的操作错误
)

# 工具函数
from a2a.utils import completed_task, new_artifact
from a2a.utils.errors import ServerError
```

## 二次开发指南

### 1. 添加新技能

在 `main.py` 中的 `skills` 列表添加新的 `AgentSkill`：

```python
skills = [
    # ... 现有技能 ...
    AgentSkill(
        id="my_new_skill",
        name="My New Skill",
        description="技能描述",
        tags=["tag1", "tag2"],
        examples=["使用示例"],
    ),
]
```

同时需要确保 Manus 代理中已经配置了对应的工具。

### 2. 实现流式响应

当前 `stream()` 方法未实现，如需支持流式响应：

```python
class A2AManus(Manus):
    async def stream(self, query: str) -> AsyncIterable[Dict[str, Any]]:
        """实现流式响应"""
        # 1. 修改 Manus 代理支持流式输出
        # 2. 逐步 yield 中间结果
        async for chunk in self.run_streaming(query):
            yield {
                "is_task_complete": False,
                "content": chunk,
            }
        yield {
            "is_task_complete": True,
            "content": "完成",
        }
```

同时需要修改 `AgentCapabilities`:
```python
capabilities = AgentCapabilities(streaming=True, pushNotifications=True)
```

### 3. 自定义请求验证

扩展 `ManusExecutor._validate_request()` 方法：

```python
def _validate_request(self, context: RequestContext) -> bool:
    """自定义请求验证逻辑"""
    user_input = context.get_user_input()

    # 添加自定义验证
    if not user_input or len(user_input.strip()) == 0:
        return True  # 返回 True 表示验证失败

    if len(user_input) > 10000:  # 限制输入长度
        return True

    return False  # 返回 False 表示验证通过
```

### 4. 添加任务取消支持

实现 `ManusExecutor.cancel()` 方法：

```python
async def cancel(
    self, request: RequestContext, event_queue: EventQueue
) -> Task | None:
    """取消正在执行的任务"""
    # 1. 获取任务 ID
    task_id = request.task_id

    # 2. 调用代理的清理方法
    if hasattr(self, 'agent') and self.agent:
        await self.agent.cleanup()

    # 3. 返回取消状态的任务
    return Task(
        id=task_id,
        status={"state": "canceled"},
        # ... 其他字段
    )
```

### 5. 扩展响应格式

修改 `A2AManus.get_agent_response()` 以支持更丰富的响应：

```python
def get_agent_response(self, config, agent_response):
    """扩展响应格式"""
    return {
        "is_task_complete": True,
        "require_user_input": False,
        "content": agent_response,
        # 添加额外字段
        "metadata": {
            "execution_time": "...",
            "steps_count": "...",
            "tools_used": ["..."],
        }
    }
```

### 6. 创建新的代理变体

基于 A2AManus 创建专用代理：

```python
# 在 agent.py 中添加
class A2ADataAnalysis(A2AManus):
    """数据分析专用 A2A 代理"""

    SUPPORTED_CONTENT_TYPES: ClassVar[List[str]] = [
        "text", "text/plain", "application/json"
    ]

    @classmethod
    async def create(cls, **kwargs) -> "A2ADataAnalysis":
        from app.agent.data_analysis import DataAnalysis
        # 使用数据分析代理而非通用 Manus
        instance = cls(**kwargs)
        # 自定义初始化...
        return instance
```

## API 端点

### 获取代理名片

```
GET /.well-known/agent.json
```

返回代理的能力、技能等元数据信息。

### 发送任务

```
POST /
Content-Type: application/json

{
    "id": 1,
    "jsonrpc": "2.0",
    "method": "message/send",
    "params": {
        "message": {
            "messageId": "",
            "role": "user",
            "parts": [{"text": "你的任务描述"}]
        }
    }
}
```

## 运行与测试

### 启动服务器

```bash
# 安装依赖
pip install a2a-sdk==0.2.5

# 启动服务（默认端口 10000）
python -m protocol.a2a.app.main

# 指定主机和端口
python -m protocol.a2a.app.main --host 0.0.0.0 --port 8080
```

### 测试请求

```bash
# 获取代理名片
curl http://localhost:10000/.well-known/agent.json

# 发送任务
curl --location 'http://localhost:10000' \
--header 'Content-Type: application/json' \
--data '{
    "id": 1,
    "jsonrpc": "2.0",
    "method": "message/send",
    "params": {
        "message": {
            "messageId": "",
            "role": "user",
            "parts": [{"text": "帮我写一个 Hello World 程序"}]
        }
    }
}'
```

## 注意事项

1. **依赖安装**: `a2a-sdk` 不在 `requirements.txt` 中，需要单独安装
2. **流式支持**: 当前仅支持非流式模式
3. **最大步数**: 默认设置 `max_steps=3`，可根据需要调整
4. **错误处理**: 任务执行失败会抛出 `ServerError`
5. **会话管理**: 使用 `InMemoryTaskStore`，重启服务会丢失状态

## 相关资源

- [A2A 协议官方文档](https://google.github.io/A2A/#/documentation)
- [A2A 示例仓库](https://github.com/google-a2a/a2a-samples)
- [OpenManus 主项目文档](../../CLAUDE.md)
