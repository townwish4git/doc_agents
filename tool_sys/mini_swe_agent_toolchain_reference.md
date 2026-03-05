# Mini-SWE-Agent 工具链管理参考文档

> 本文档总结了 Mini-SWE-Agent 的工具链管理范式，供开发自定义 Code Agent 时参考。
>
> 涵盖内容：工具架构设计、调用流程、配置系统、实现细节、最佳实践

---

## 目录

1. [架构概览](#1-架构概览)
2. [工具定义系统](#2-工具定义系统)
3. [完整调用链](#3-完整调用链)
4. [权限控制系统](#4-权限控制系统)
5. [内置工具参考](#5-内置工具参考)
6. [配置系统设计](#6-配置系统设计)
7. [环境执行层](#7-环境执行层)
8. [实现参考代码](#8-实现参考代码)
9. [最佳实践总结](#9-最佳实践总结)

---

## 1. 架构概览

### 1.1 核心设计原则

```
┌─────────────────────────────────────────────────────────────────┐
│              Mini-SWE-Agent 工具链核心原则                       │
├─────────────────────────────────────────────────────────────────┤
│ 1. 简化设计：单一 Bash 工具覆盖大部分需求                        │
│ 2. 标准兼容：OpenAI Function Calling 格式                        │
│ 3. 环境隔离：命令执行在独立环境中（Docker/Local/Swerex）          │
│ 4. 模板驱动：Jinja2 模板控制输入输出格式                          │
│ 5. 配置化：YAML 配置切换不同运行模式                              │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Agent 层 (agents/)                                 │
│  - 对话管理 (DefaultAgent)                                    │
│  - 任务循环 (run() → step() → query())                        │
│  - 动作执行 (execute_actions())                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 模型层 (models/)                                    │
│  - LLM 调用 (LiteLLM 封装)                                    │
│  - 工具定义 (BASH_TOOL)                                       │
│  - 响应解析 (parse_toolcall_actions)                          │
│  - 消息格式化 (format_toolcall_observation_messages)          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 环境执行层 (environments/)                           │
│  - 命令执行 (Environment.execute)                             │
│  - 进程管理 (subprocess)                                      │
│  - 沙箱隔离 (Docker/Singularity/Swerex)                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 工具定义系统

### 2.1 工具定义格式

Mini-SWE-Agent 采用 **OpenAI Function Calling** 格式定义工具：

```python
# models/utils/actions_toolcall.py
BASH_TOOL = {
    "type": "function",
    "function": {
        "name": "bash",
        "description": "Execute a bash command",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The bash command to execute",
                }
            },
            "required": ["command"],
        },
    },
}
```

**Responses API 格式**（扁平结构）：

```python
# models/utils/actions_toolcall_response.py
BASH_TOOL_RESPONSE_API = {
    "type": "function",
    "name": "bash",
    "description": "Execute a bash command",
    "parameters": {
        "type": "object",
        "properties": {
            "command": {
                "type": "string",
                "description": "The bash command to execute",
            }
        },
        "required": ["command"],
    },
}
```

### 2.2 工具硬编码策略

与动态注册不同，Mini-SWE-Agent 采用**硬编码工具定义**：

```python
# models/litellm_model.py
class LitellmModel:
    def query(self, history: list[dict], system_prompt: str | None = None):
        response = litellm.completion(
            model=self.model_name,
            messages=messages,
            tools=[BASH_TOOL],  # ← 硬编码工具
            ...
        )
```

### 2.3 支持的模型后端

| 模型类 | 工具格式 | 说明 |
|--------|----------|------|
| `LitellmModel` | `BASH_TOOL` | LiteLLM 统一接口 |
| `PortkeyModel` | `BASH_TOOL` | Portkey 网关 |
| `RequestyModel` | `BASH_TOOL` | Requesty 平台 |
| `OpenRouterModel` | `BASH_TOOL` | OpenRouter 路由 |
| `LiteLLMModelResponseAPI` | `BASH_TOOL_RESPONSE_API` | Responses API |

---

## 3. 完整调用链

### 3.1 调用流程图

```
用户任务: "修复代码中的 bug"
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Agent 初始化                                          │
│                                                             │
│ DefaultAgent.run()                                          │
│ ├── 加载配置 (mini.yaml / default.yaml)                      │
│ ├── 初始化模型 (LitellmModel)                                │
│ ├── 初始化环境 (LocalEnvironment)                            │
│ └── 进入主循环                                               │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: 构建 Prompt                                          │
│                                                             │
│ system_prompt:                                              │
│ "你是一名软件工程助手，可以使用 bash 工具执行命令..."          │
│                                                             │
│ instance_template (Jinja2):                                 │
│ "请解决以下问题：{{ problem_statement }}"                     │
│                                                             │
│ 历史消息: [{role: "user", content: ...},                     │
│          {role: "assistant", tool_calls: [...]},             │
│          {role: "tool", content: ...}]                       │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: LLM 决策 + 工具调用                                   │
│                                                             │
│ POST /v1/chat/completions                                   │
│ {                                                           │
│   "model": "gpt-4",                                         │
│   "messages": [...],                                        │
│   "tools": [{"type": "function", "function": {...}}]        │
│ }                                                           │
│                                                             │
│ 响应:                                                       │
│ {                                                           │
│   "tool_calls": [{                                          │
│     "id": "call_xxx",                                       │
│     "function": {                                           │
│       "name": "bash",                                       │
│       "arguments": '{"command": "ls -la"}'                  │
│     }                                                       │
│   }]                                                        │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: 解析工具调用                                          │
│                                                             │
│ parse_toolcall_actions()                                    │
│ ├── 验证 JSON 格式                                          │
│ ├── 验证工具名称 ("bash")                                   │
│ ├── 提取 command 参数                                       │
│ └── 返回 Action 列表                                        │
│                                                             │
│ 错误处理:                                                   │
│ └── FormatError → 返回格式错误提示给 LLM                      │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: 执行命令                                              │
│                                                             │
│ Environment.execute(cmd="ls -la")                           │
│ ├── subprocess.run()                                        │
│ ├── 捕获 stdout/stderr                                      │
│ └── 返回结果字典                                             │
│                                                             │
│ 结果:                                                       │
│ {                                                           │
│   "output": "total 32\ndrwx...",                            │
│   "exit_code": 0,                                           │
│   "exception_info": null                                    │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: 格式化观察消息                                        │
│                                                             │
│ format_toolcall_observation_messages()                      │
│ ├── 应用 observation_template                               │
│ ├── 设置 role="tool"                                        │
│ ├── 关联 tool_call_id                                       │
│ └── 添加到历史消息                                           │
│                                                             │
│ 消息格式:                                                   │
│ {                                                           │
│   "role": "tool",                                           │
│   "tool_call_id": "call_xxx",                               │
│   "content": "<observation>输出内容</observation>"           │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 7: 循环或结束                                            │
│                                                             │
│ 检测到完成命令 (submit/submit_summary):                      │
│ └── 抛出 Submitted 异常 → 结束循环                           │
│                                                             │
│ 未检测到:                                                   │
│ └── 返回 Step 2 继续对话                                     │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 关键数据结构

```python
# 工具调用请求（LLM 输出）
class ToolCall:
    id: str                    # 调用 ID
    function: Function         # 函数信息
    type: str = "function"     # 固定为 "function"

class Function:
    name: str                  # 工具名称（"bash"）
    arguments: str             # JSON 字符串参数

# Action（内部表示）
class Action:
    command: str               # 要执行的命令

# 执行结果
class ExecutionResult:
    output: str                # 命令输出
    exit_code: int             # 返回码
    exception_info: dict       # 异常信息

# 观察消息
class ObservationMessage:
    role: str = "tool"         # 固定为 "tool"
    tool_call_id: str          # 关联的工具调用 ID
    content: str               # 格式化后的内容
```

---

## 4. 权限控制系统

### 4.1 交互式权限模式

Mini-SWE-Agent 的权限控制通过 `InteractiveAgent` 实现：

```python
# agents/interactive.py
class InteractiveAgent(DefaultAgent):
    def __init__(self, mode: str = "confirm", ...):
        self.mode = mode  # "human" | "confirm" | "yolo"
```

### 4.2 三种权限模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `human` | 用户输入直接执行，LM 建议需确认 | 完全控制 |
| `confirm` | LM 命令需用户确认后执行 | 安全审查 |
| `yolo` | LM 命令自动执行（危险） | 自动化运行 |

### 4.3 权限检查流程

```python
# interactive.py 简化逻辑
async def step(self, observation: str):
    if self.mode == "human":
        # 用户输入直接执行
        user_input = await self.cli.ask("Command")
        return Action(command=user_input)

    elif self.mode == "confirm":
        # 获取 LM 建议
        actions = await self.query(observation)
        # 用户确认
        for action in actions:
            confirmed = await self.cli.confirm(f"Run: {action.command}?")
            if confirmed:
                return action

    elif self.mode == "yolo":
        # 直接执行 LM 建议
        return await self.query(observation)
```

### 4.4 与 OpenCode 权限系统对比

```
OpenCode:                      Mini-SWE-Agent:
┌─────────────────┐            ┌─────────────────┐
│ 配置文件规则     │            │ 交互式模式选择   │
│ (allow/deny/ask)│            │ (human/confirm/  │
│                 │            │  yolo)          │
│ 细粒度文件级控制 │            │ 命令级人工确认   │
└─────────────────┘            └─────────────────┘
```

---

## 5. 内置工具参考

### 5.1 当前工具集

Mini-SWE-Agent 目前主要提供 **单一 Bash 工具**：

| 工具 | 用途 | 核心参数 | 适用场景 |
|------|------|----------|----------|
| `bash` | 执行 shell 命令 | `command` | 文件操作、代码执行、git 等 |

### 5.2 Bash 工具设计

```python
BASH_TOOL = {
    "type": "function",
    "function": {
        "name": "bash",
        "description": "Execute a bash command",
        "parameters": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The bash command to execute",
                }
            },
            "required": ["command"],
        },
    },
}
```

**设计决策**：
- 单一工具覆盖所有需求（简化架构）
- 命令字符串形式（灵活，但需 LLM 正确构造）
- 无参数验证（依赖环境执行时的错误反馈）

### 5.3 与 OpenCode 工具对比

| 维度 | OpenCode | Mini-SWE-Agent |
|------|----------|----------------|
| 工具数量 | 15+ 个专用工具 | 1 个通用 Bash |
| 文件读取 | `read` 工具 | `bash: cat` |
| 文件搜索 | `glob`, `grep` | `bash: find`, `grep` |
| 文件编辑 | `edit`, `write` | `bash: sed`, `echo` |
| 参数验证 | Zod Schema | 无（字符串传递）|
| 安全性 | 细粒度权限 | 交互式确认 |

---

## 6. 配置系统设计

### 6.1 YAML 配置结构

```yaml
# config/mini.yaml (工具调用模式)
agent:
  name: "default"
  model:
    name: "gpt-4"
    tool_calls: true           # 启用工具调用
    observation_template: |    # 输出格式化模板
      <observation>
      {% if output %}{{ output }}{% endif %}
      {% if exception_info %}[Exception]: {{ exception_info }}{% endif %}
      <exit_code>{{ exit_code }}</exit_code>
      </observation>
    format_error_template: |   # 错误格式化模板
      <format_error>
      {{ error_message }}
      </format_error>

# config/default.yaml (文本模式)
agent:
  name: "default"
  model:
    name: "gpt-4"
    tool_calls: false          # 禁用工具调用
    action_template: "..."     # 动作解析模板（正则）
```

### 6.2 配置加载机制

```python
# __init__.py
_CONFIG_MAPPING = {
    "mini": "config/mini.yaml",
    "default": "config/default.yaml",
}

def load_config(name: str) -> dict:
    """加载指定配置文件"""
    config_path = _CONFIG_MAPPING[name]
    return yaml.safe_load(open(config_path))
```

### 6.3 模板系统

**Observation 模板变量**：
- `{{ output }}` - 命令标准输出
- `{{ exit_code }}` - 进程返回码
- `{{ exception_info }}` - 异常信息

**错误模板变量**：
- `{{ error_message }}` - 错误描述

---

## 7. 环境执行层

### 7.1 环境接口定义

```python
# __init__.py
class Environment(Protocol):
    def execute(
        self,
        action: Action,
        timeout: int | None = None,
        **kwargs
    ) -> dict:
        """
        Execute an action in the environment.

        Returns:
            {
                "output": str,           # stdout/stderr 合并输出
                "exit_code": int,        # 0 表示成功
                "exception_info": dict   # 异常详情
            }
        """
        ...
```

### 7.2 环境实现列表

| 环境类 | 文件 | 说明 | 适用场景 |
|--------|------|------|----------|
| `LocalEnvironment` | `local.py` | 本地 subprocess | 开发测试 |
| `DockerEnvironment` | `docker/` | Docker 容器 | 隔离执行 |
| `SwerexEnvironment` | `swerex/` | 远程执行 | 分布式 |
| `BubblewrapEnvironment` | `bubblewrap.py` | 沙箱 | 高安全 |
| `ContreeEnvironment` | `contree.py` | Contree 平台 | 云服务 |

### 7.3 LocalEnvironment 实现

```python
# environments/local.py
class LocalEnvironment:
    def execute(self, action: Action, timeout: int | None = None):
        try:
            result = subprocess.run(
                action.command,
                shell=True,
                capture_output=True,
                text=True,
                timeout=timeout,
                cwd=self.workdir,
            )
            return {
                "output": result.stdout + result.stderr,
                "exit_code": result.returncode,
                "exception_info": None,
            }
        except subprocess.TimeoutExpired:
            return {
                "output": "",
                "exit_code": -1,
                "exception_info": {"type": "Timeout"},
            }
        except Exception as e:
            return {
                "output": "",
                "exit_code": -1,
                "exception_info": {"type": type(e).__name__, "message": str(e)},
            }
```

---

## 8. 实现参考代码

### 8.1 最小工具调用实现模板

```python
# my_tool.py
from typing import TypedDict

# 1. 定义工具 Schema
MY_TOOL = {
    "type": "function",
    "function": {
        "name": "my_tool",
        "description": "Description for LLM",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {
                    "type": "string",
                    "description": "Parameter description",
                }
            },
            "required": ["param1"],
        },
    },
}

# 2. 解析函数
def parse_my_tool_actions(response) -> list[Action]:
    actions = []
    for tool_call in response.tool_calls:
        if tool_call.function.name == "my_tool":
            args = json.loads(tool_call.function.arguments)
            actions.append(Action(command=args["param1"]))
    return actions

# 3. 格式化观察
def format_my_tool_observation(result: dict, tool_call_id: str) -> dict:
    return {
        "role": "tool",
        "tool_call_id": tool_call_id,
        "content": f"<result>{result['output']}</result>"
    }
```

### 8.2 自定义模型类

```python
# my_model.py
from minisweagent.models.base import BaseModel

class MyModel(BaseModel):
    def query(self, history: list[dict], system_prompt: str | None = None):
        # 1. 构建消息
        messages = []
        if system_prompt:
            messages.append({"role": "system", "content": system_prompt})
        messages.extend(history)

        # 2. 调用 LLM API
        response = my_llm_api_call(
            messages=messages,
            tools=[BASH_TOOL],  # 使用工具
        )

        # 3. 解析响应
        if response.tool_calls:
            actions = parse_toolcall_actions(response)
            return actions

        # 4. 直接文本响应
        return [Action(command=response.content)]

    def format_observation(self, execution_result: dict) -> list[dict]:
        return format_toolcall_observation_messages(
            execution_result,
            self.config.get("observation_template")
        )
```

### 8.3 错误处理模式

```python
# exceptions.py
class InterruptAgentFlow(Exception):
    """Base exception for flow control"""
    pass

class FormatError(InterruptAgentFlow):
    """工具调用格式错误"""
    def __init__(self, message: str, raw_response: str):
        self.message = message
        self.raw_response = raw_response

class Submitted(InterruptAgentFlow):
    """任务提交完成"""
    pass

class UserInterruption(InterruptAgentFlow):
    """用户中断"""
    pass
```

---

## 9. 最佳实践总结

### 9.1 工具设计原则

1. **单一工具 vs 多工具权衡**
   - Mini-SWE-Agent 选择单一 Bash 工具（简化）
   - OpenCode 选择多专用工具（精确控制）
   - **建议**：根据安全需求和复杂度选择

2. **参数设计**
   ```python
   # 简单参数（Mini-SWE-Agent 风格）
   {"command": "cat file.txt"}

   # 结构化参数（OpenCode 风格）
   {"file_path": "file.txt", "offset": 1, "limit": 50}
   ```

3. **错误反馈**
   - 使用异常类控制流程（`InterruptAgentFlow`）
   - 格式化错误返回给 LLM 重试

### 9.2 安全最佳实践

1. **环境隔离**
   ```python
   # 生产环境使用 Docker
   env = DockerEnvironment(image="python:3.9")
   # 而非
   env = LocalEnvironment()  # 仅开发测试
   ```

2. **交互式确认**
   ```python
   # 关键操作前确认
   if self.mode == "confirm":
       confirmed = await self.cli.confirm(f"Execute: {cmd}?")
       if not confirmed:
           raise UserInterruption()
   ```

3. **超时控制**
   ```python
   # 所有命令设置超时
   result = env.execute(action, timeout=120)  # 2分钟
   ```

### 9.3 LLM 体验优化

1. **模板引导**
   ```yaml
   observation_template: |
     <observation>
     {{ output }}
     <exit_code>{{ exit_code }}</exit_code>
     </observation>
   ```

2. **格式错误恢复**
   ```python
   except FormatError as e:
       # 返回格式错误给 LLM，允许重试
       return self.format_error_message(e)
   ```

3. **完成检测**
   ```python
   # 检测完成命令
   if action.command.strip() in ["submit", "submit_summary"]:
       raise Submitted()
   ```

---

## 附录：核心文件结构

```
src/minisweagent/
├── __init__.py                    # 协议定义 (Model, Environment)
│
├── agents/
│   ├── base.py                    # BaseAgent 抽象类
│   ├── default.py                 # DefaultAgent 主实现
│   ├── interactive.py             # InteractiveAgent (权限控制)
│   └── repl.py                    # REPL 交互
│
├── models/
│   ├── base.py                    # BaseModel 抽象
│   ├── litellm_model.py           # LiteLLM 实现 (工具调用)
│   ├── litellm_response_model.py  # Responses API 实现
│   ├── portkey_model.py           # Portkey 网关
│   ├── requesty_model.py          # Requesty 平台
│   ├── openrouter_model.py        # OpenRouter
│   └── utils/
│       ├── actions_toolcall.py          # BASH_TOOL + 解析
│       ├── actions_toolcall_response.py # Responses API 版本
│       └── actions_text.py              # 文本模式解析
│
├── environments/
│   ├── base.py                    # 环境基类
│   ├── local.py                   # 本地执行
│   ├── docker/                    # Docker 环境
│   ├── swerex/                    # Swerex 远程执行
│   ├── bubblewrap.py              # 沙箱
│   └── contree.py                 # Contree 平台
│
├── config/
│   ├── mini.yaml                  # 工具调用配置
│   └── default.yaml               # 文本模式配置
│
└── exceptions.py                  # 异常定义
```

---

*文档版本：1.0*
*基于：Mini-SWE-Agent main branch (2025)*
*用途：自定义 Code Agent 开发参考*
