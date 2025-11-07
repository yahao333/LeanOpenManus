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

