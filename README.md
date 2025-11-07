# OpenManus 源码分析与教程

## 目录
- [项目概述](#项目概述)
- [核心架构](#核心架构)
- [1. 项目结构](#1-项目结构)
- [2. 核心组件](#2-核心组件)
- [3. 执行流程](#3-执行流程)
- [4. 关键特性](#4-关键特性)
- [5. 扩展开发](#5-扩展开发)
- [6. 最佳实践](#6-最佳实践)
- [7. 故障排除](#7-故障排除)
- [总结](#总结)

## 项目概述

OpenManus 是一个开源的通用AI代理框架，旨在实现任何想法而无需邀请码。该项目由 MetaGPT 团队成员开发，具有强大的工具调用能力和灵活的架构设计。

## 核心架构

### 1. 项目结构

```
OpenManus/
├── app/                    # 核心应用代码
│   ├── agent/             # 代理实现
│   ├── tool/              # 工具系统
│   ├── config.py          # 配置管理
│   ├── llm.py             # LLM 接口
│   └── schema.py          # 数据模型
├── config/                # 配置文件
├── workspace/             # 工作目录
├── main.py               # 主入口
└── requirements.txt       # 依赖管理
```

### 2. 核心组件

#### 2.1 代理系统 (Agent System)

**BaseAgent (app/agent/base.py)**
- 所有代理的基类
- 提供状态管理、内存管理和执行循环
- 支持最大步数限制和卡顿检测

**Manus 代理 (app/agent/manus.py)**
- 主要的通用代理实现
- 继承自 ToolCallAgent
- 支持本地工具和 MCP 工具
- 浏览器上下文管理

```python
class Manus(ToolCallAgent):
    """A versatile general-purpose agent with support for both local and MCP tools."""
    
    name: str = "Manus"
    description: str = "A versatile agent that can solve various tasks..."
    
    # 工具集合
    available_tools: ToolCollection = Field(
        default_factory=lambda: ToolCollection(
            PythonExecute(),
            BrowserUseTool(),
            StrReplaceEditor(),
            AskHuman(),
            Terminate(),
        )
    )
```

#### 2.2 工具系统 (Tool System)

**BaseTool (app/tool/base.py)**
- 所有工具的基类
- 提供标准化的执行接口
- 支持参数验证和结果处理

```python
class BaseTool(ABC, BaseModel):
    """Consolidated base class for all tools combining BaseModel and Tool functionality."""
    
    name: str
    description: str
    parameters: Optional[dict] = None
    
    @abstractmethod
    async def execute(self, **kwargs) -> Any:
        """Execute the tool with given parameters."""
```

**工具类型:**
- `PythonExecute`: Python 代码执行
- `BrowserUseTool`: 浏览器自动化
- `StrReplaceEditor`: 文件编辑
- `AskHuman`: 人工交互
- `Terminate`: 终止执行

#### 2.3 配置系统 (Configuration System)

**Config (app/config.py)**
- 单例模式的配置管理
- 支持 TOML 配置文件
- 模块化配置结构

```toml
# config/config.toml
[llm]
model = "gpt-4o"
base_url = "https://api.openai.com/v1"
api_key = "sk-..."
max_tokens = 4096
temperature = 0.0

[browser]
headless = false
disable_security = true

[mcp]
server_reference = "app.mcp.server"
```

#### 2.4 LLM 接口 (LLM Interface)

**LLM (app/llm.py)**
- 支持多种 LLM 提供商 (OpenAI, Azure, AWS Bedrock)
- 令牌计数和限制管理
- 重试机制和错误处理

```python
class LLM:
    """Language model interface with token counting and retry mechanisms."""
    
    def __init__(self, config_name: str = "default", llm_config: Optional[LLMSettings] = None):
        # 初始化客户端和配置
        pass
    
    @retry(
        retry=retry_if_exception_type((RateLimitError, APIError)),
        wait=wait_random_exponential(min=1, max=60),
        stop=stop_after_attempt(3),
    )
    async def create_chat_completion(self, messages: List[Message], **kwargs) -> ChatCompletion:
        """Create chat completion with retry logic."""
```

### 3. 执行流程

#### 3.1 主执行流程

```python
# main.py
async def main():
    # 创建代理
    agent = await Manus.create()
    
    # 获取用户输入
    prompt = input("Enter your prompt: ")
    
    # 执行任务
    await agent.run(prompt)
    
    # 清理资源
    await agent.cleanup()
```

#### 3.2 代理执行循环

```python
# app/agent/base.py
async def run(self, request: Optional[str] = None) -> str:
    """Execute the agent's main loop asynchronously."""
    
    if request:
        self.update_memory("user", request)
    
    async with self.state_context(AgentState.RUNNING):
        while (self.current_step < self.max_steps and 
               self.state != AgentState.FINISHED):
            self.current_step += 1
            step_result = await self.step()
            
            # 卡顿检测
            if self.is_stuck():
                self.handle_stuck_state()
```

#### 3.3 工具调用流程

```python
# app/agent/toolcall.py
async def step(self) -> str:
    """Execute a single step with tool calling."""
    
    # 思考下一步行动
    response = await self.llm.create_chat_completion(
        messages=self.memory.messages,
        tools=self.available_tools.to_params(),
        tool_choice="auto",
    )
    
    # 处理工具调用
    if response.choices[0].message.tool_calls:
        await self._handle_tool_calls(response.choices[0].message.tool_calls)
```

### 4. 关键特性

#### 4.1 MCP 支持 (Model Context Protocol)

OpenManus 支持 MCP 协议，可以连接外部工具服务器：

```python
# MCP 服务器连接
await self.connect_mcp_server(server_config.url, server_id)

# 工具动态添加
new_tools = [tool for tool in self.mcp_clients.tools if tool.server_id == server_id]
self.available_tools.add_tools(*new_tools)
```

#### 4.2 浏览器自动化

基于 browser-use 库的浏览器自动化：

```python
class BrowserUseTool(BaseTool):
    """A powerful browser automation tool for web interaction."""
    
    async def execute(self, action: str, **kwargs):
        # 支持多种浏览器操作
        # go_to_url, click_element, input_text, scroll_down, etc.
```

#### 4.3 沙箱执行

安全的代码执行环境：

```python
class PythonExecute(BaseTool):
    """A tool for executing Python code with timeout and safety restrictions."""
    
    async def execute(self, code: str, timeout: int = 5):
        # 使用多进程隔离执行
        # 超时控制和输出捕获
```

### 5. 扩展开发

#### 5.1 自定义工具开发

```python
from app.tool.base import BaseTool

class CustomTool(BaseTool):
    name = "custom_tool"
    description = "A custom tool for specific tasks"
    parameters = {
        "type": "object",
        "properties": {
            "param1": {"type": "string", "description": "Parameter 1"},
        },
        "required": ["param1"],
    }
    
    async def execute(self, param1: str) -> ToolResult:
        # 实现工具逻辑
        result = await some_async_operation(param1)
        return self.success_response(result)
```

#### 5.2 自定义代理开发

```python
from app.agent.base import BaseAgent

class CustomAgent(BaseAgent):
    name = "CustomAgent"
    description = "A specialized agent for specific domains"
    
    async def step(self) -> str:
        # 实现自定义执行逻辑
        # 可以集成特定的工具和策略
        pass
```

### 6. 最佳实践

#### 6.1 配置管理
- 使用环境变量管理敏感信息
- 为不同环境创建配置文件
- 定期更新 API 密钥

#### 6.2 错误处理
- 实现适当的重试机制
- 添加详细的日志记录
- 处理网络超时和限流

#### 6.3 性能优化
- 合理设置最大步数限制
- 使用适当的超时设置
- 监控令牌使用情况

### 7. 故障排除

#### 7.1 常见问题

1. **工具调用失败**
   - 检查工具参数格式
   - 验证工具可用性
   - 查看错误日志

2. **LLM 连接问题**
   - 验证 API 密钥
   - 检查网络连接
   - 确认服务可用性

3. **内存不足**
   - 减少最大步数
   - 优化提示词
   - 清理历史消息

#### 7.2 调试技巧

```python
# 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)

# 检查工具状态
print(f"Available tools: {[tool.name for tool in agent.available_tools]}")

# 查看执行历史
for msg in agent.memory.messages:
    print(f"{msg.role}: {msg.content}")
```

## 总结

OpenManus 提供了一个强大而灵活的框架来构建通用AI代理。其模块化设计、丰富的工具生态系统和可扩展的架构使其成为开发智能代理应用的理想选择。通过理解其核心架构和执行流程，开发者可以有效地使用和扩展这个框架来满足各种应用需求。
