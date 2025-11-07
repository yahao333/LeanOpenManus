# main.py

**功能定位与角色**
- main.py 的功能定位
  - 提供命令行入口，负责解析参数、初始化智能体、接收用户输入并驱动执行流程。
  - 使用 `asyncio` 管理异步生命周期，确保创建、运行与清理按顺序执行。
- 在系统中的角色
  - 作为“应用入口”，把外部输入（CLI 提示词）转换为内部调用（Agent 运行），并在退出前进行资源回收。
  - 把“框架能力”通过 `Manus` 智能体暴露为可运行的流程，是从用户到系统的第一落点。

**目录导图（本文件内）**
- 函数
  - `main`（异步）：解析参数、创建 `Manus`、获取提示词、执行 `agent.run`、清理资源。
- 运行入口
  - `if __name__ == "__main__": asyncio.run(main())`
- 类
  - 无（本文件不定义类；智能体类在 `app.agent.manus` 中）

**核心调用链（按执行顺序）**
- 入口执行
  - `if __name__ == "__main__":` → `asyncio.run(main())`
- 应用启动
  - `main()` 启动异步上下文，实例化 CLI 解析器 `argparse.ArgumentParser`
- 智能体初始化
  - `agent = await Manus.create()`：创建并初始化 `Manus` 智能体（加载配置、提示词、工具等，具体在 `app/agent/manus.py`）
- 获取提示词
  - 优先使用 `--prompt` 参数；否则通过 `input("Enter your prompt: ")` 交互获取
  - 空输入检测：`if not prompt.strip(): logger.warning(...) ; return`
- 执行请求
  - `await agent.run(prompt)`：把提示词交给智能体执行（可能触发规划、工具调用、沙盒等）
- 资源清理
  - `finally: await agent.cleanup()`：确保无论正常结束或被中断（如 `KeyboardInterrupt`），都进行清理

**重要代码段解读（含中文注释）**
- 入口与主函数
```
import argparse
import asyncio

from app.agent.manus import Manus
from app.logger import logger


# 当前功能：定义主函数
# 该函数是程序的入口点，负责解析命令行参数、创建和初始化智能体、处理用户输入和执行智能体的运行。
# 它使用 argparse 库来解析命令行参数，包括 --prompt 参数用于指定输入提示。
# 如果未提供 --prompt 参数，程序会提示用户输入。
# 主函数首先创建一个 Manus 智能体实例，并调用其 create 方法进行初始化。
# 然后，它根据命令行参数或用户输入，调用智能体的 run 方法来处理请求。
# 最后，在程序退出前，调用智能体的 cleanup 方法来释放资源。
async def main():
    # 1) 构建命令行解析器，声明可选参数 --prompt
    parser = argparse.ArgumentParser(description="Run Manus agent with a prompt")
    parser.add_argument(
        "--prompt", type=str, required=False, help="Input prompt for the agent"
    )
    args = parser.parse_args()

    # 2) 异步创建 Manus 智能体（内部会加载配置、提示词与工具）
    agent = await Manus.create()
    try:
        # 3) 获取提示词：优先使用命令行 --prompt，否则交互输入
        prompt = args.prompt if args.prompt else input("Enter your prompt: ")
        # 4) 基础校验：避免空白输入导致不必要调用
        if not prompt.strip():
            logger.warning("Empty prompt provided.")
            return

        # 5) 执行前告知状态（日志）
        logger.warning("Processing your request...")
        # 6) 调用智能体执行：核心“从 Prompt 到 Action”的入口
        await agent.run(prompt)
        logger.info("Request processing completed.")
    except KeyboardInterrupt:
        # 7) 捕获用户中断（Ctrl+C），给出提示
        logger.warning("Operation interrupted.")
    finally:
        # 8) 无论成功或异常，都进行资源清理，保证健壮性
        await agent.cleanup()


# 程序入口：通过 asyncio 在事件循环中运行 main
if __name__ == "__main__":
    asyncio.run(main())
```

**关系图（该模块与其他模块）**
```
[CLI 用户] 
   |
   v
[main.py]
   ├─ 解析参数: `argparse.ArgumentParser`
   ├─ 管理异步: `asyncio.run(...)`
   ├─ 日志输出: `app.logger.logger`
   └─ 调用智能体: `app.agent.manus.Manus`
        |
        v
[Manus Agent]
   ├─ 读取配置: `app.config` 与 `config/*.toml`
   ├─ 使用提示词: `app/prompt/manus.py`（注入 `workspace_root` 等上下文）
   ├─ 调用工具: `app/tool/*`（bash、python_execute、web_search、file_operators 等）
   ├─ 执行沙盒: `app/sandbox/*`（安全执行环境）
   ├─ 编排流程: `app/flow/*`（规划与流程控制）
   └─ LLM 抽象: `app.llm` / `app.bedrock`（统一模型接口）
```

- 要点总结
  - `main.py` 是最薄的一层：只负责“参数→Agent→清理”的控制。
  - 智能体的核心能力都在 `app/agent/manus.py` 及其依赖模块中实现；`main.py` 把用户输入交给它，并维护可靠的生命周期。
  - 使用 `finally` 保证 `cleanup` 一定执行，是健壮性与资源管理的关键。

# app/agent/manus.py

我先快速定位并阅读 Manus 智能体源码及其依赖的 prompt 与 config。随后提炼 create/run/cleanup 的执行流程与关键设计。
        
下面我们聚焦 `app/agent/manus.py` 的 `create/run/cleanup`，并结合其 `prompt` 与 `config`，从“如何构建一个可运行的 Agent”出发讲解。

**功能定位与角色**
- 功能定位
  - `Manus` 是通用多工具智能体，继承自 `ToolCallAgent`，支持本地工具与 MCP 远程工具，且能根据浏览器使用状态动态增强思考提示。
- 在系统中的角色
  - 作为主力执行代理，把来自 `main.py` 的用户请求转为“规划-工具调用-结果反馈”的闭环执行，并在运行前后管理 MCP 连接与资源回收。

**目录导图**
- class Manus(ToolCallAgent)
  - 字段与属性
    - `name` / `description`
    - `system_prompt`（来自 `app/prompt/manus.py`，注入 `config.workspace_root`）
    - `next_step_prompt`（来自 `app/prompt/manus.py`）
    - `max_observe` / `max_steps`
    - `mcp_clients: MCPClients`
    - `available_tools: ToolCollection`（含 `PythonExecute`、`BrowserUseTool`、`StrReplaceEditor`、`AskHuman`、`Terminate`）
    - `special_tool_names`（默认含 `Terminate`）
    - `browser_context_helper: Optional[BrowserContextHelper]`
    - `connected_servers: Dict[str, str]`
    - `_initialized: bool`
  - 方法
    - `initialize_helper`（模型校验后执行，同步初始化浏览器上下文辅助器）
    - `create`（类方法，异步工厂：初始化 MCP 并设置 `_initialized`）
    - `initialize_mcp_servers`（根据 `config.mcp_config.servers` 建立 SSE 或 stdio 连接）
    - `connect_mcp_server`（连接单个 MCP 并把该服务器提供的工具注入到 `available_tools`）
    - `disconnect_mcp_server`（断开 MCP 并从工具集中移除该服务器工具）
    - `cleanup`（清理浏览器、断开 MCP、重置 `_initialized`）
    - `think`（在 ReAct 思考前根据最近消息判断是否使用了浏览器工具，从而动态替换 `next_step_prompt`，之后调用父类 `think`）
  - 继承的方法（未在本文件显式定义）
    - `run`：继承自 `ToolCallAgent.run`，在运行结束时确保调用 `cleanup`（父类实现）

**执行顺序与核心调用链**
- 智能体创建阶段
  - `Manus.create()` → 构造实例 → `initialize_mcp_servers()`
    - 从 `config.mcp_config.servers` 读取配置，按 `type` 建立 SSE 或 stdio 连接
    - 每建立一个 MCP 连接，调用 `connect_mcp_server()` 把该服务器提供的工具（`MCPClientTool`）加入 `available_tools`
    - 设置 `_initialized = True`
- 请求执行阶段（由 `main.py` 调用）
  - `ToolCallAgent.run(request)` → 调用父类（`ReActAgent`）的运行逻辑
    - 若 `Manus.think()` 检测到最近的工具调用中包含浏览器工具，则通过 `BrowserContextHelper.format_next_step_prompt()` 注入浏览器上下文，临时替换 `next_step_prompt` 后进入思考/决策循环（ReAct）
    - 决策可能选择本地工具或 MCP 工具执行，并将消息与工具调用存入记忆（memory）
- 结束与清理阶段
  - `ToolCallAgent.run` 的 `finally` 中会调用 `cleanup()`，同时 `main.py` 的 `finally` 也会调用 `cleanup()`，实现双保险
  - `cleanup()` 会清理浏览器资源，并断开 MCP 服务器连接，重建工具集（移除 MCP 工具），重置 `_initialized`

**关键代码段解读（附中文注释）**
- 基本定义与提示词注入
```
class Manus(ToolCallAgent):
    """A versatile general-purpose agent with support for both local and MCP tools."""

    name: str = "Manus"
    description: str = "A versatile agent that can solve various tasks using multiple tools including MCP-based tools"

    # 使用系统提示模板，并注入工作目录
    system_prompt: str = SYSTEM_PROMPT.format(directory=config.workspace_root)
    # 下一步提示，指导工具选择与交互策略
    next_step_prompt: str = NEXT_STEP_PROMPT

    # 控制单次观察与总体步骤的上限
    max_observe: int = 10000
    max_steps: int = 20

    # MCP 客户端：用于与远程 MCP 服务器交互并加载工具
    mcp_clients: MCPClients = Field(default_factory=MCPClients)

    # 预置的一组通用工具：本地执行、浏览器控制、字符串替换、人类介入、终止
    available_tools: ToolCollection = Field(
        default_factory=lambda: ToolCollection(
            PythonExecute(),
            BrowserUseTool(),
            StrReplaceEditor(),
            AskHuman(),
            Terminate(),
        )
    )

    # 特殊工具：终止（用于随时停止）
    special_tool_names: list[str] = Field(default_factory=lambda: [Terminate().name])
    # 浏览器上下文辅助器（用于在思考阶段加入浏览器状态）
    browser_context_helper: Optional[BrowserContextHelper] = None

    # 已连接的 MCP 服务器（server_id -> url/command）
    connected_servers: Dict[str, str] = Field(default_factory=dict)
    _initialized: bool = False
```
- 初始化辅助器与工厂创建
```
@model_validator(mode="after")
def initialize_helper(self) -> "Manus":
    """同步初始化基础组件"""
    self.browser_context_helper = BrowserContextHelper(self)
    return self

@classmethod
async def create(cls, **kwargs) -> "Manus":
    """工厂方法：创建并正确初始化 Manus 实例"""
    instance = cls(**kwargs)
    # 初始化 MCP 服务器连接并加载工具
    await instance.initialize_mcp_servers()
    instance._initialized = True
    return instance
```
- 初始化 MCP 服务器与连接
```
async def initialize_mcp_servers(self) -> None:
    """根据配置初始化 MCP 服务器连接"""
    for server_id, server_config in config.mcp_config.servers.items():
        try:
            if server_config.type == "sse":
                if server_config.url:
                    await self.connect_mcp_server(server_config.url, server_id)
                    logger.info(f"Connected to MCP server {server_id} at {server_config.url}")
            elif server_config.type == "stdio":
                if server_config.command:
                    await self.connect_mcp_server(
                        server_config.command, server_id, use_stdio=True, stdio_args=server_config.args
                    )
                    logger.info(f"Connected to MCP server {server_id} using command {server_config.command}")
        except Exception as e:
            logger.error(f"Failed to connect to MCP server {server_id}: {e}")

async def connect_mcp_server(
    self, server_url: str, server_id: str = "", use_stdio: bool = False, stdio_args: List[str] = None
) -> None:
    """连接 MCP 服务器并添加其工具"""
    if use_stdio:
        await self.mcp_clients.connect_stdio(server_url, stdio_args or [], server_id)
        self.connected_servers[server_id or server_url] = server_url
    else:
        await self.mcp_clients.connect_sse(server_url, server_id)
        self.connected_servers[server_id or server_url] = server_url

    # 只添加该服务器提供的新工具
    new_tools = [tool for tool in self.mcp_clients.tools if tool.server_id == server_id]
    self.available_tools.add_tools(*new_tools)
```
- 断开 MCP 与重建工具集
```
async def disconnect_mcp_server(self, server_id: str = "") -> None:
    """断开 MCP 连接并移除其工具"""
    await self.mcp_clients.disconnect(server_id)
    if server_id:
        self.connected_servers.pop(server_id, None)
    else:
        self.connected_servers.clear()

    # 只保留基础工具（移除 MCPClientTool），再重新添加仍存在的 MCP 工具
    base_tools = [tool for tool in self.available_tools.tools if not isinstance(tool, MCPClientTool)]
    self.available_tools = ToolCollection(*base_tools)
    self.available_tools.add_tools(*self.mcp_clients.tools)
```
- 清理资源
```
async def cleanup(self):
    """清理 Manus 资源"""
    if self.browser_context_helper:
        await self.browser_context_helper.cleanup_browser()
    # 仅在初始化过的情况下断开 MCP
    if self._initialized:
        await self.disconnect_mcp_server()
        self._initialized = False
```
- 思考阶段（动态注入浏览器上下文）
```
async def think(self) -> bool:
    """在思考前根据上下文调整提示"""
    if not self._initialized:
        # 惰性补齐：若未初始化则初始化 MCP（保障独立调用场景）
        await self.initialize_mcp_servers()
        self._initialized = True

    original_prompt = self.next_step_prompt
    # 查看最近 3 条消息中是否使用了浏览器工具
    recent_messages = self.memory.messages[-3:] if self.memory.messages else []
    browser_in_use = any(
        tc.function.name == BrowserUseTool().name
        for msg in recent_messages
        if msg.tool_calls
        for tc in msg.tool_calls
    )

    # 如果正在使用浏览器，则注入浏览器上下文增强提示
    if browser_in_use:
        self.next_step_prompt = await self.browser_context_helper.format_next_step_prompt()

    result = await super().think()

    # 恢复原提示，避免污染后续思考
    self.next_step_prompt = original_prompt
    return result
```

**模块关系图**
```
[main.py]
  └─ 创建与驱动: `Manus.create()` → `Manus.run()` → `Manus.cleanup()`
        |
        v
[Manus (ToolCallAgent)]
  ├─ 继承运行/清理: `ToolCallAgent.run/cleanup`（父类实现）
  ├─ 思考增强: `Manus.think()` 动态注入浏览器上下文
  ├─ 工具集: `ToolCollection`（本地工具 + MCP 工具）
  ├─ 浏览器辅助: `BrowserContextHelper`
  ├─ MCP 连接: `MCPClients`（SSE/stdio）← `config.mcp_config.servers`
  └─ 提示词: `app/prompt/manus.py`（`SYSTEM_PROMPT` + `NEXT_STEP_PROMPT`，注入 `config.workspace_root`）
        |
        v
[基础设施]
  ├─ `app/config.py`（加载 `config.toml` 与 `mcp.json`，提供 `workspace_root` 与 MCP 配置）
  ├─ `app/agent/toolcall.py`（工具调用与运行生命周期）
  ├─ `app/agent/react.py`（ReAct 推理循环）
  └─ `app/tool/*`（工具实现：本地与 MCP）
```

**要点与建议**
- `create` 是“带副作用的工厂”（连接 MCP），`think` 做“上下文感知的提示切换”，`cleanup` 做“资源回收与工具集重建”。
- `system_prompt` 把 `workspace_root` 注入到提示中，提升智能体对当前项目的上下文理解。
- 运行路径通常为 `main.py → Manus.create → Manus.run → cleanup`，其中 `run` 的 `finally` 会确保清理，即使上层未调用清理也能安全结束。
- 如需体验完整流程，建议在 venv（Python 3.12）中运行 `python main.py --prompt "在当前工程中搜索 README 并总结要点"` 观察工具调用与日志输出。

        