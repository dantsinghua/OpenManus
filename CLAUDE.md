# OpenManus 项目指南

OpenManus 是一个开源的通用人工智能代理框架，旨在让用户无需邀请码即可构建和运行强大的 AI 代理。该项目由 MetaGPT 团队成员开发，支持多种大语言模型（LLM）、浏览器自动化、代码执行以及 MCP（Model Context Protocol）协议集成。

## 项目结构

```
OpenManus/
├── app/                        # 主应用程序目录
│   ├── agent/                  # AI 代理实现
│   │   ├── base.py             # 基础代理类 BaseAgent
│   │   ├── react.py            # ReAct 代理
│   │   ├── toolcall.py         # 工具调用代理
│   │   ├── manus.py            # 主代理 Manus
│   │   ├── mcp.py              # MCP 协议代理
│   │   ├── browser.py          # 浏览器代理
│   │   ├── swe.py              # 软件工程代理
│   │   ├── data_analysis.py    # 数据分析代理
│   │   └── sandbox_agent.py    # 沙箱代理
│   ├── tool/                   # 工具模块（34+ 工具）
│   │   ├── base.py             # 工具基类 BaseTool
│   │   ├── python_execute.py   # Python 代码执行
│   │   ├── browser_use_tool.py # 浏览器操作
│   │   ├── str_replace_editor.py # 文件编辑
│   │   ├── web_search.py       # 网页搜索
│   │   └── ...
│   ├── mcp/                    # MCP 服务模块
│   ├── flow/                   # 工作流编排
│   ├── prompt/                 # 提示词配置
│   ├── sandbox/                # 沙箱执行环境
│   ├── daytona/                # Daytona 云沙箱集成
│   ├── config.py               # 配置管理
│   ├── llm.py                  # LLM 接口封装
│   ├── schema.py               # 数据模型定义
│   └── logger.py               # 日志系统
├── protocol/                   # 通信协议
│   └── a2a/                    # Agent-to-Agent 协议
├── config/                     # 配置文件目录
│   ├── config.example.toml     # 主配置示例
│   └── mcp.example.json        # MCP 配置示例
├── main.py                     # 主程序入口
├── run_mcp.py                  # MCP 代理运行器
├── run_flow.py                 # 多代理工作流
├── sandbox_main.py             # 沙箱模式入口
├── requirements.txt            # Python 依赖
└── Dockerfile                  # Docker 配置
```

## 核心功能实现

### 1. 代理层级架构

项目采用层级继承的代理架构：

```
BaseAgent          # 基础代理：状态管理、内存、执行循环
    ↓
ReActAgent         # ReAct 模式：思考-行动循环
    ↓
ToolCallAgent      # 工具调用：LLM 函数调用能力
    ↓
├── Manus          # 通用多功能代理
├── DataAnalysis   # 数据分析专用代理
└── SandboxManus   # 沙箱环境代理
```

**BaseAgent** (`app/agent/base.py`):
- 管理代理状态（`AgentState`: IDLE, RUNNING, FINISHED, ERROR）
- 提供 `Memory` 存储对话历史
- 实现步进式执行循环 `run()` 方法
- 检测并处理代理卡死状态 `is_stuck()`

**Manus** (`app/agent/manus.py`):
- 默认最大步数：20 步
- 观察窗口：10000 字符
- 内置工具：PythonExecute、BrowserUseTool、StrReplaceEditor、AskHuman、Terminate
- 支持 MCP 服务器连接

### 2. 工具系统

工具基类 **BaseTool** (`app/tool/base.py`) 定义了标准接口：

```python
class BaseTool(ABC, BaseModel):
    name: str           # 工具名称
    description: str    # 工具描述
    parameters: dict    # 参数 JSON Schema

    async def execute(self, **kwargs) -> Any:
        """执行工具"""

    def to_param(self) -> Dict:
        """转换为 OpenAI 函数调用格式"""
```

**ToolCollection** 用于管理多个工具，支持动态添加和移除。

主要工具类别：
- **代码执行**: `PythonExecute`, `Bash`
- **浏览器**: `BrowserUseTool`, `ComputerUseTool`
- **文件操作**: `StrReplaceEditor`, `FileOperators`
- **搜索引擎**: `GoogleSearch`, `BaiduSearch`, `DuckDuckGoSearch`, `BingSearch`
- **数据可视化**: `ChartVisualization`

### 3. LLM 封装

**LLM 类** (`app/llm.py`) 提供统一的大语言模型接口：

- 支持 OpenAI、Azure OpenAI、Anthropic Claude、Ollama、AWS Bedrock
- 实现 Token 计数和限制检查
- 提供重试机制（指数退避，最多 6 次）
- 支持流式和非流式响应

关键方法：
- `ask()`: 普通对话
- `ask_with_images()`: 图像理解
- `ask_tool()`: 工具调用

### 4. 工作流系统

**FlowFactory** (`app/flow/flow_factory.py`) 创建工作流：

```python
flow = FlowFactory.create_flow(
    flow_type=FlowType.PLANNING,
    agents={"manus": Manus(), "data_analysis": DataAnalysis()}
)
result = await flow.execute(prompt)
```

`PlanningFlow` 支持多代理协作，可自动进行任务分解和分配。

### 5. MCP 协议支持

项目实现了 Model Context Protocol (MCP)，支持：
- **stdio** 模式：通过标准输入输出通信
- **SSE** 模式：通过 Server-Sent Events 通信

配置示例（`config/mcp.example.json`）：
```json
{
  "servers": {
    "my_server": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "my_mcp_server"]
    }
  }
}
```

### 6. 沙箱执行

沙箱系统 (`app/sandbox/`) 提供隔离的代码执行环境：
- Docker 容器隔离
- 可配置内存和 CPU 限制
- 支持 Daytona 云沙箱

## 部署指南

### 环境要求

- Python 3.12+
- Docker（可选，用于沙箱功能）

### 方法一：使用 conda

```bash
# 1. 创建虚拟环境
conda create -n open_manus python=3.12
conda activate open_manus

# 2. 克隆仓库
git clone https://github.com/FoundationAgents/OpenManus.git
cd OpenManus

# 3. 安装依赖
pip install -r requirements.txt

# 4. 安装浏览器驱动（可选）
playwright install
```

### 方法二：使用 uv（推荐）

```bash
# 1. 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. 克隆仓库
git clone https://github.com/FoundationAgents/OpenManus.git
cd OpenManus

# 3. 创建并激活虚拟环境
uv venv --python 3.12
source .venv/bin/activate  # Unix/macOS
# .venv\Scripts\activate   # Windows

# 4. 安装依赖
uv pip install -r requirements.txt
```

### 方法三：使用 Docker

```bash
# 构建镜像
docker build -t openmanus .

# 运行容器
docker run -it --rm -v $(pwd)/config:/app/OpenManus/config openmanus python main.py
```

### 配置

1. 复制配置文件：
```bash
cp config/config.example.toml config/config.toml
```

2. 编辑 `config/config.toml`，填入你的 API 密钥：

```toml
[llm]
model = "gpt-4o"                      # 或 "claude-3-7-sonnet-20250219"
base_url = "https://api.openai.com/v1"
api_key = "your-api-key-here"
max_tokens = 8192
temperature = 0.0

[llm.vision]
model = "gpt-4o"
base_url = "https://api.openai.com/v1"
api_key = "your-api-key-here"
```

### 运行

```bash
# 基础模式
python main.py

# 指定提示词
python main.py --prompt "帮我搜索今天的新闻"

# MCP 工具模式
python run_mcp.py

# 多代理工作流模式
python run_flow.py

# 沙箱模式
python sandbox_main.py
```

### 启用数据分析代理

在 `config/config.toml` 中添加：
```toml
[runflow]
use_data_analysis_agent = true
```

然后运行：
```bash
python run_flow.py
```

## 常用命令

```bash
# 运行测试
pytest tests/

# 代码格式检查
pre-commit run --all-files

# 查看日志
# 日志文件位于 logs/ 目录
```

## 关键文件速查

| 功能 | 文件路径 |
|------|---------|
| 主程序入口 | `main.py` |
| 配置管理 | `app/config.py` |
| LLM 接口 | `app/llm.py` |
| 基础代理 | `app/agent/base.py` |
| Manus 代理 | `app/agent/manus.py` |
| 工具基类 | `app/tool/base.py` |
| 配置示例 | `config/config.example.toml` |

## 支持的 LLM 提供商

- OpenAI (GPT-4o, GPT-4o-mini)
- Anthropic Claude (claude-3-opus, claude-3-sonnet, claude-3-haiku)
- Azure OpenAI
- AWS Bedrock
- Ollama（本地模型）
- JiekouAI
- PPIO

## 注意事项

1. **API 密钥安全**: 请勿将包含真实 API 密钥的 `config.toml` 提交到版本控制
2. **浏览器自动化**: 使用 `BrowserUseTool` 前需要先运行 `playwright install`
3. **沙箱模式**: 需要安装 Docker 并确保当前用户有 Docker 权限
4. **Token 限制**: 注意 LLM 的 Token 限制，长对话可能触发 `TokenLimitExceeded` 异常
