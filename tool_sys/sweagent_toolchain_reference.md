# SWE-agent 工具链管理参考文档

> 本文档总结了SWE-agent的工具链管理范式，供开发自定义Code Agent时参考。
>
> 涵盖内容：工具架构设计、调用流程、权限系统、Bundle系统、实现细节

---

## 目录

1. [架构概览](#1-架构概览)
2. [Bundle工具系统](#2-bundle工具系统)
3. [工具配置系统](#3-工具配置系统)
4. [完整调用链](#4-完整调用链)
5. [权限控制系统](#5-权限控制系统)
6. [内置工具参考](#6-内置工具参考)
7. [解析器系统](#7-解析器系统)
8. [实现参考代码](#8-实现参考代码)
9. [最佳实践总结](#9-最佳实践总结)

---

## 1. 架构概览

### 1.1 核心设计原则

```
┌─────────────────────────────────────────────────────────────────┐
│                    SWE-agent 工具链核心原则                       │
├─────────────────────────────────────────────────────────────────┤
│ 1. Bundle化组织：工具以自包含Bundle形式组织，便于分发和复用        │
│ 2. 配置驱动：通过YAML配置定义工具，无需修改代码即可扩展            │
│ 3. 多格式支持：支持多种LLM输出格式（Function Calling、XML、JSON等）│
│ 4. 状态管理：内置Registry系统支持跨操作的状态持久化               │
│ 5. 安全第一：命令过滤机制防止危险操作                             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Agent 层                                           │
│  - 对话管理                                                   │
│  - 任务规划                                                   │
│  - 工具选择决策（由LLM完成）                                    │
│  - 支持多种模型格式（OpenAI、Anthropic、LiteLLM等）              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 工具管理层 (Tool Management)                        │
│  - Bundle注册与发现 (ToolHandler)                             │
│  - 配置解析 (config.yaml → Command对象)                       │
│  - 参数验证 (Pydantic + Argument定义)                         │
│  - 命令过滤 (ToolFilter)                                      │
│  - 解析器选择 (Parser Strategy)                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 执行层 (Execution)                                  │
│  - Bundle安装 (install.sh + bin/上传)                         │
│  - 命令执行 (Environment运行)                                 │
│  - 状态收集 (_state命令)                                      │
│  - 输出处理 (截断、格式化)                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Bundle工具系统

### 2.1 Bundle目录结构

```
tools/
├── edit_anthropic/              # 文件编辑Bundle示例
│   ├── bin/                     # 可执行脚本目录（必需）
│   │   ├── str_replace_editor   # 主工具脚本
│   │   └── _state               # 状态命令（可选）
│   ├── config.yaml             # Bundle配置（必需）
│   ├── install.sh              # 安装脚本（可选）
│   ├── lib/                    # Python库（可选）
│   └── README.md              # 文档（可选）
│
├── registry/                   # 状态持久化Bundle
│   ├── bin/
│   │   ├── register           # 键值存储命令
│   │   ├── registry_query     # 查询命令
│   │   └── _state             # 返回注册表状态
│   └── config.yaml
│
├── windowed/                   # 窗口化文件浏览
│   ├── bin/
│   │   ├── open               # 打开文件
│   │   ├── goto               # 跳转到行
│   │   ├── scroll             # 滚动浏览
│   │   └── _state             # 返回窗口状态
│   └── config.yaml
│
└── submit/                     # 提交Bundle
    ├── bin/
    │   └── submit             # 提交解决方案
    └── config.yaml
```

### 2.2 Bundle核心概念

**Bundle是SWE-agent工具管理的基本单元：**

```python
# sweagent/tools/bundle.py
class Bundle:
    """Represents a tool bundle on disk"""

    def __init__(self, path: Path):
        self.path = path
        self.config = self._load_config()

    def _load_config(self) -> BundleConfig:
        """Load and validate config.yaml"""
        config_path = self.path / "config.yaml"
        raw = yaml.safe_load(config_path.read_text())
        return BundleConfig(
            tools=raw.get("tools", {}),
            state_command=raw.get("state_command")
        )

    @property
    def bin_path(self) -> Path:
        """Path to executable scripts"""
        return self.path / "bin"

    @property
    def install_script(self) -> Path | None:
        """Optional install.sh path"""
        script = self.path / "install.sh"
        return script if script.exists() else None
```

### 2.3 创建自定义Bundle

```yaml
# tools/my_bundle/config.yaml
tools:
  my_tool:
    signature: "my_tool <required_arg> [<optional_arg>]"
    docstring: |
      Description shown to LLM explaining what this tool does.
      Can be multiline and include usage examples.
    arguments:
      - name: required_arg
        type: string
        description: "This argument is required"
        required: true
      - name: optional_arg
        type: integer
        description: "This argument is optional"
        required: false
        default: 10

# Optional: State command for persistent state
state_command: "_state"
```

```python
# tools/my_bundle/bin/my_tool
#!/usr/bin/env python3
import argparse
import sys

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("required_arg", help="Required argument")
    parser.add_argument("optional_arg", nargs="?", type=int, default=10)
    args = parser.parse_args()

    # Tool logic here
    result = process(args.required_arg, args.optional_arg)

    # Output to stdout (captured by agent)
    print(result)
    return 0

def process(required, optional):
    return f"Processed {required} with {optional}"

if __name__ == "__main__":
    sys.exit(main())
```

---

## 3. 工具配置系统

### 3.1 工具配置结构

```python
# sweagent/tools/tools.py
class ToolConfig(BaseModel):
    """Main configuration for all tools"""

    # Bundle configurations
    bundles: list[BundleConfig] = Field(default_factory=list)

    # Environment variables for tool execution
    env_variables: dict[str, str] = Field(default_factory=dict)

    # Registry variables for state management
    registry_variables: dict[str, str] = Field(default_factory=dict)

    # Enable built-in bash tool
    enable_bash_tool: bool = True

    # Parser configuration
    parse_function: ParseFunction = Field(default_factory=ParseFunction)

    # Command filtering
    filter: ToolFilterConfig = Field(default_factory=ToolFilterConfig)
```

### 3.2 Command定义系统

```python
# sweagent/tools/commands.py
class Command(BaseModel):
    """
    Represents an executable tool command.
    Created from config.yaml tool definitions.
    """
    name: str
    docstring: str
    signature: str
    end_name: str | None = None  # For multi-line commands
    arguments: list[Argument]
    invoke_format: str  # Generated format string

class Argument(BaseModel):
    """Command parameter definition"""
    name: str
    type: str  # "string", "integer", "array", etc.
    description: str
    required: bool = True
    enum: list[str] | None = None  # Allowed values
    argument_format: str | None = None  # Jinja template
```

### 3.3 配置示例

```yaml
# Agent配置中的tools部分
agent:
  tools:
    # 启用哪些Bundles
    bundles:
      - path: tools/registry
      - path: tools/edit_anthropic
      - path: tools/windowed
      - path: tools/search

    # 环境变量
    env_variables:
      PAGER: cat
      GIT_PAGER: cat
      PYTHONUNBUFFERED: "1"

    # 注册表变量（持久化状态）
    registry_variables:
      USE_FILEMAP: "true"
      EDITOR_TYPE: "windowed"

    # 启用bash工具
    enable_bash_tool: true

    # 解析器配置
    parse_function:
      type: function_calling  # 或 thought_action, xml, json

    # 命令过滤
    filter:
      blocklist:
        - vim
        - vi
        - emacs
        - nano
      blocklist_standalone:
        - python
        - bash
```

---

## 4. 完整调用链

### 4.1 调用流程图

```
用户输入: "帮我修复这个bug"
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Agent构建System Prompt                               │
│                                                             │
│ System: 你是一个软件工程助手，可以使用以下工具：              │
│                                                             │
│ [从Bundle config.yaml生成的工具文档]                         │
│                                                             │
│ ## str_replace_editor                                       │
│ Signature: str_replace_editor <command> [<args>]            │
│ - view: View file content                                   │
│ - create: Create new file                                   │
│ - str_replace: Replace string in file                       │
│                                                             │
│ ## open                                                     │
│ Signature: open <file_path>                                 │
│ Open a file in the editor window                            │
│                                                             │
│ ## bash                                                     │
│ Signature: bash <command>                                   │
│ Run command in bash shell                                   │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: LLM决策并输出工具调用                                │
│                                                             │
│ 输出格式取决于parse_function.type:                           │
│                                                             │
│ Function Calling格式:                                       │
│ {                                                           │
│   "tool": "open",                                           │
│   "arguments": {"file_path": "/repo/src/main.py"}           │
│ }                                                           │
│                                                             │
│ Thought-Action格式:                                         │
│ Let me first look at the main file.                         │
│ <open>                                                      │
│ /repo/src/main.py                                           │
│ </open>                                                     │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Parser解析工具调用                                   │
│                                                             │
│ 根据parse_function.type选择解析器:                           │
│ - function_calling → FunctionCallingParser                  │
│ - thought_action → ThoughtActionParser                      │
│ - xml → XMLThoughtActionParser                              │
│ - json → JsonParser                                         │
│                                                             │
│ 解析结果:                                                   │
│ ParsedCommand(name="open", args={"file_path": "/repo/..."}) │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: 命令过滤检查                                         │
│                                                             │
│ ToolFilter.check(command):                                  │
│ - 检查是否在blocklist中 (vim, emacs等)                      │
│ - 检查是否在blocklist_standalone中                          │
│ - 检查是否匹配dangerous patterns                            │
│                                                             │
│ ❌ 被阻止: "vim main.py" → 返回错误                          │
│ ✅ 通过: "open main.py" → 继续执行                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: 命令执行                                             │
│                                                             │
│ ToolHandler.execute():                                      │
│ 1. 格式化命令: "open /repo/src/main.py"                     │
│ 2. 上传到Environment执行                                     │
│ 3. 捕获stdout/stderr                                        │
│ 4. 执行_state命令收集环境状态                                 │
│ 5. 返回执行结果                                              │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: 结果返回给LLM                                        │
│                                                             │
│ 结果包含:                                                   │
│ - 命令输出 (stdout/stderr)                                  │
│ - 退出码                                                    │
│ - 环境状态 (open_file, working_dir等)                       │
│                                                             │
│ LLM生成下一步行动...                                         │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 关键数据结构

```python
# 工具调用请求（LLM输出，解析后）
class ParsedCommand(BaseModel):
    name: str                    # 命令名称
    args: dict[str, Any]         # 解析后的参数
    raw_command: str | None      # 原始命令字符串

# 工具执行结果
class CommandResult(BaseModel):
    output: str                  # 标准输出
    error: str | None           # 标准错误
    exit_code: int              # 退出码
    state: dict[str, Any]       # 环境状态

# 注册表状态（跨操作持久化）
class RegistryState(BaseModel):
    variables: dict[str, str]   # 键值存储
    filemap: dict[str, Any]     # 文件摘要
```

---

## 5. 权限控制系统

### 5.1 命令过滤机制

```python
# sweagent/tools/tools.py
class ToolFilterConfig(BaseModel):
    """Configuration for command filtering"""

    # Commands blocked when appearing anywhere in the command
    blocklist: list[str] = Field(default_factory=list)
    # Example: ["vim", "emacs", "nano"]
    # Blocks: "vim file.py", "sudo vim file.py"

    # Commands blocked only when they are the entire command
    blocklist_standalone: list[str] = Field(default_factory=list)
    # Example: ["python", "bash"]
    # Blocks: "python", but not "python script.py"

    # Regex patterns for conditional blocking
    blocked_commands: list[Block] = Field(default_factory=list)

class Block(BaseModel):
    """Conditional block rule"""
    command: str           # Command pattern to match
    pattern: str | None   # Regex to check in full command
    message: str          # Error message if blocked
```

### 5.2 过滤配置示例

```yaml
agent:
  tools:
    filter:
      # 完全禁止交互式编辑器
      blocklist:
        - vim
        - vi
        - emacs
        - nano
        - less
        - more

      # 禁止单独运行python/bash（必须使用带参数的完整命令）
      blocklist_standalone:
        - python
        - python3
        - bash
        - sh

      # 条件性阻止危险命令
      blocked_commands:
        - command: "rm"
          pattern: "^-rf\s+/"  # 阻止 rm -rf /
          message: "Dangerous deletion command blocked"

        - command: ">"
          pattern: ".*"  # 阻止输出重定向
          message: "Use write tool instead of shell redirection"
```

### 5.3 过滤检查流程

```python
# sweagent/tools/tools.py
class ToolHandler:
    def should_block_action(self, command: str) -> BlockAction | None:
        """Check if command should be blocked"""

        # 1. Check blocklist (anywhere in command)
        for blocked in self.config.filter.blocklist:
            if blocked in command:
                return BlockAction(
                    blocked=True,
                    message=f"Command '{blocked}' is not allowed"
                )

        # 2. Check standalone blocklist
        cmd_only = command.strip().split()[0]
        for blocked in self.config.filter.blocklist_standalone:
            if cmd_only == blocked:
                return BlockAction(
                    blocked=True,
                    message=f"'{blocked}' cannot be run standalone"
                )

        # 3. Check regex patterns
        for block in self.config.filter.blocked_commands:
            if block.pattern and re.search(block.pattern, command):
                return BlockAction(
                    blocked=True,
                    message=block.message
                )

        return None  # Command allowed
```

---

## 6. 内置工具参考

### 6.1 工具概览

swe-agent的工具由**两大部分**组成：

| 类型 | 说明 | 代表工具 |
|------|------|---------|
| **内置Bash工具** | 核心基础工具，通过代码定义 | `bash` |
| **Bundle工具** | 通过YAML配置的模块化工具包 | `str_replace_editor`, `open`, `search_file`等 |

### 6.2 内置Bash工具

#### 6.2.1 bash命令

**定义位置**: `sweagent/tools/commands.py:206-220`

```python
BASH_COMMAND = Command(
    name="bash",
    signature="<command>",
    docstring="runs the given command directly in bash",
    arguments=[
        Argument(
            name="command",
            type="string",
            description="The bash command to execute.",
            required=True,
        )
    ],
)
```

**功能**: 在bash shell中执行任意命令

**配置控制**:
```yaml
agent:
  tools:
    enable_bash_tool: true  # 默认启用
```

**使用示例**:
```bash
# 执行git命令
bash "git status"

# 运行测试
bash "pytest tests/"

# 安装依赖
bash "pip install -r requirements.txt"
```

**安全限制**: 受`filter`配置控制，默认阻止:
- 交互式编辑器: `vim`, `vi`, `emacs`, `nano`
- 单独运行的解释器: `python`, `bash`（无参数）
- 危险命令: `tail -f`, `make`等

---

### 6.3 Bundle工具详解

#### 6.3.1 edit_anthropic - 文件编辑器

**路径**: `tools/edit_anthropic/`

**核心命令**: `str_replace_editor`

**完整功能列表**:

| 子命令 | 功能 | 关键参数 |
|--------|------|---------|
| `view` | 查看文件/目录内容 | `path`, `view_range` |
| `create` | 创建新文件 | `path`, `file_text` |
| `str_replace` | 字符串替换编辑 | `path`, `old_str`, `new_str` |
| `insert` | 在指定行后插入 | `path`, `insert_line`, `new_str` |
| `undo_edit` | 撤销上次编辑 | `path` |

**详细参数**:
```yaml
str_replace_editor:
  signature: "str_replace_editor <command> <path> [options]"
  arguments:
    - name: command
      type: string
      required: true
      enum: ["view", "create", "str_replace", "insert", "undo_edit"]

    - name: path
      type: string
      required: true
      description: "绝对路径，如 /testbed/file.py"

    - name: view_range
      type: array[integer]
      required: false
      description: "查看范围 [start_line, end_line]，1-indexed，-1表示到末尾"

    - name: file_text
      type: string
      required: false  # create命令必需
      description: "创建文件的内容"

    - name: old_str
      type: string
      required: false  # str_replace命令必需
      description: "要替换的字符串（必须精确匹配，包括缩进）"

    - name: new_str
      type: string
      required: false
      description: "替换后的新字符串"

    - name: insert_line
      type: integer
      required: false  # insert命令必需
      description: "插入位置的行号（在该行之后插入）"
```

**使用示例**:

```bash
# 查看整个文件
str_replace_editor view /repo/src/main.py

# 查看特定行范围（第10-20行）
str_replace_editor view /repo/src/main.py --view_range "10 20"

# 查看从第50行到文件末尾
str_replace_editor view /repo/src/main.py --view_range "50 -1"

# 查看目录（最多2层深度）
str_replace_editor view /repo/src

# 创建新文件
str_replace_editor create /repo/src/new.py --file_text "# New file\ndef hello():\n    pass"

# 字符串替换（必须精确匹配）
str_replace_editor str_replace /repo/src/main.py \
  --old_str "def old_function():" \
  --new_str "def new_function():"

# 在第10行后插入代码
str_replace_editor insert /repo/src/main.py \
  --insert_line 10 \
  --new_str "    print('debug')"

# 撤销上次编辑
str_replace_editor undo_edit /repo/src/main.py
```

**状态命令**: `_state_anthropic`
- 返回: `{"working_dir": "/current/path"}`

**设计特点**:
1. **精确匹配**: `old_str`必须完全匹配目标文本（包括空格和缩进）
2. **唯一性检查**: 如果`old_str`在文件中不唯一，替换会失败
3. **自动截断**: 长输出会自动截断并标记`<response clipped>`
4. **撤销支持**: 每次编辑自动备份，支持`undo_edit`

---

#### 6.3.2 windowed - 窗口化文件浏览

**路径**: `tools/windowed/`

**核心概念**: 通过"窗口"机制浏览大文件，只显示部分内容

**完整命令列表**:

| 命令 | 功能 | 参数 | 状态影响 |
|------|------|------|---------|
| `open` | 打开文件 | `path`, `line_number`(可选) | 设置`CURRENT_FILE` |
| `goto` | 跳转到指定行 | `line_number` | 移动窗口 |
| `scroll_up` | 向上滚动 | 无 | 窗口上移 |
| `scroll_down` | 向下滚动 | 无 | 窗口下移 |
| `create` | 创建新文件 | `filename` | 设置`CURRENT_FILE` |

**详细参数**:
```yaml
open:
  signature: 'open "<path>" [<line_number>]'
  arguments:
    - name: path
      type: string
      required: true
    - name: line_number
      type: integer
      required: false
      description: "跳转到的行号，不提供则从窗口起始显示"

goto:
  signature: "goto <line_number>"
  arguments:
    - name: line_number
      type: integer
      required: true

create:
  signature: "create <filename>"
  arguments:
    - name: filename
      type: string
      required: true

scroll_up:
  signature: "scroll_up"
  description: "向上滚动{WINDOW}行（默认100行）"

scroll_down:
  signature: "scroll_down"
  description: "向下滚动{WINDOW}行"
```

**使用示例**:

```bash
# 打开文件（从第1行开始显示）
open "/repo/src/main.py"

# 打开文件并跳转到第150行
open "/repo/src/main.py" 150

# 在当前文件内跳转到第200行
goto 200

# 向上滚动（查看前面的代码）
scroll_up

# 向下滚动（查看后面的代码）
scroll_down

# 创建新文件
create "/repo/src/utils.py"
```

**状态命令**: `_state`
```json
{
  "open_file": "/absolute/path/to/file.py",
  "working_dir": "/current/working/directory"
}
```

**窗口机制**:
- 默认窗口大小: 100行（可通过`WINDOW`环境变量配置）
- `open`不带行号时: 默认从窗口起始位置显示
- `goto`后: 目标行位于窗口顶部
- 滚动命令: 移动整个窗口，显示相邻内容

---

#### 6.3.3 windowed_edit_replace - 基于搜索替换的编辑

**路径**: `tools/windowed_edit_replace/`

**与windowed Bundle配合使用**，提供基于搜索替换的编辑功能

**完整命令列表**:

| 命令 | 功能 | 参数 |
|------|------|------|
| `edit` | 搜索替换编辑 | `search`, `replace`, `replace-all`(可选) |
| `insert` | 在指定位置插入 | `text`, `line`(可选) |

**详细参数**:
```yaml
edit:
  signature: "edit <search> <replace> [<replace-all>]"
  arguments:
    - name: search
      type: string
      required: true
      description: "要搜索的文本（必须包含正确的缩进）"
    - name: replace
      type: string
      required: true
      description: "替换文本（必须包含正确的缩进）"
    - name: replace-all
      type: boolean
      required: false
      default: false
      description: "是否替换所有匹配项"

insert:
  signature: "insert <text> [<line>]"
  arguments:
    - name: text
      type: string
      required: true
    - name: line
      type: integer
      required: false
      description: "在该行之后插入，不提供则在文件末尾插入"
```

**使用示例**:

```bash
# 先打开文件
open "/repo/src/main.py"

# 搜索替换（仅替换第一个匹配）
edit "print('Hello world')" "print('Hello')\nprint('world')"

# 搜索替换所有匹配
edit "self.debug = True" "self.debug = False" true

# 在文件末尾插入
insert "# End of file"

# 在第50行后插入
insert "    # TODO: optimize this" 50
```

**设计特点**:
1. **上下文保留**: 搜索文本需要包含足够上下文以确保唯一性
2. **缩进敏感**: 必须正确处理Python缩进
3. **块操作**: 支持多行搜索和替换
4. **作用域限制**: 仅在当前打开的文件中操作

**重要提示**:
- 当添加`if`/`with`/`try`语句时，需要在`replace`中包含后续块的缩进
- 如果包装代码在`try`块中，确保同时添加`except`或`finally`块

---

#### 6.3.4 windowed_edit_linting - 带Lint检查的编辑

**路径**: `tools/windowed_edit_linting/`

**核心命令**: `edit`

**功能**: 执行编辑后自动进行flake8 lint检查

```yaml
edit:
  signature: |
    edit <start_line>:<end_line>
    <replacement_text>
    end_of_edit
  end_name: "end_of_edit"
  description: "多行编辑命令，以end_of_edit结束"
```

**使用示例**:

```bash
# 多行编辑
edit 10:20
    def new_function():
        print("line 1")
        print("line 2")
end_of_edit
```

**特性**:
- 使用`end_of_edit`作为结束标记（多行命令）
- 编辑后自动运行flake8检查
- 如果引入lint错误，会显示警告

---

#### 6.3.5 windowed_edit_rewrite - 全量重写

**路径**: `tools/windowed_edit_rewrite/`

**核心命令**: `edit`

**功能**: 用新文本完全替换当前显示的所有行

```yaml
edit:
  signature: "edit <text>"
  arguments:
    - name: text
      type: string
      required: true
```

**使用场景**: 需要完全重写当前窗口内容时

---

#### 6.3.6 search - 代码搜索工具

**路径**: `tools/search/`

**完整命令列表**:

| 命令 | 功能 | 参数 |
|------|------|------|
| `search_file` | 在单个文件中搜索 | `search_term`, `file`(可选) |
| `search_dir` | 在目录中递归搜索 | `search_term`, `dir`(可选) |
| `find_file` | 按文件名查找 | `file_name`, `dir`(可选) |

**详细参数**:
```yaml
search_file:
  signature: "search_file <search_term> [<file>]"
  arguments:
    - name: search_term
      type: string
      required: true
    - name: file
      type: string
      required: false
      description: "目标文件，不提供则搜索当前打开的文件"

search_dir:
  signature: "search_dir <search_term> [<dir>]"
  arguments:
    - name: search_term
      type: string
      required: true
    - name: dir
      type: string
      required: false
      description: "目标目录，不提供则搜索当前目录"

find_file:
  signature: "find_file <file_name> [<dir>]"
  arguments:
    - name: file_name
      type: string
      required: true
      description: "文件名或模式（支持shell通配符如*.py）"
    - name: dir
      type: string
      required: false
```

**使用示例**:

```bash
# 在当前打开的文件中搜索
search_file "def main"

# 在指定文件中搜索
search_file "TODO" "/repo/src/main.py"

# 在当前目录递归搜索
search_dir "class.*Exception"

# 在指定目录搜索
search_dir "import os" "/repo/src"

# 查找所有Python文件
find_file "*.py"

# 在特定目录查找配置文件
find_file "*.yaml" "/repo/config"
```

**search_file输出格式**:
```
Found 3 matches for "def main" in /repo/src/main.py:
Line 15:def main():
Line 42:def main_wrapper():
Line 88:    def main(self):
End of matches for "def main" in /repo/src/main.py
```

**安全限制**:
- `search_file`: 如果匹配超过100行，会提示缩小搜索范围
- `search_dir`: 防止在全仓库搜索时产生过多输出

---

#### 6.3.7 filemap - Python文件摘要

**路径**: `tools/filemap/`

**核心命令**: `filemap`

**功能**: 使用tree-sitter解析Python文件，显示文件结构但跳过长的函数体

**实现原理**:
- 使用tree-sitter解析Python AST
- 识别`function_definition`节点
- 函数体超过5行的部分会被省略，显示为`... eliding lines X-Y ...`

**参数**:
```yaml
filemap:
  signature: "filemap <file_path>"
  arguments:
    - name: file_path
      type: string
      required: true
```

**使用示例**:

```bash
# 查看大型Python文件的结构
filemap "/repo/src/large_module.py"
```

**输出示例**:
```
     1 import os
     2 import sys
     3
     5 def long_function():
     6 ... eliding lines 6-50 ...
    51
    52 class MyClass:
    53     def method(self):
    54 ... eliding lines 54-100 ...
```

**适用场景**: 快速了解大型Python模块的结构，而无需查看完整实现

---

#### 6.3.8 registry - 状态持久化系统

**路径**: `tools/registry/`

**核心功能**: 提供跨操作的状态持久化（键值存储）

**实现组件**:
- `_read_env`: 读取注册表变量
- `_write_env`: 写入注册表变量
- `lib/registry.py`: Registry类实现

**使用方式**:

Registry不是直接的LLM工具，而是供其他Bundle使用的底层服务。

**其他Bundle使用Registry**:
```python
# windowed/bin/open
from registry import registry
current_file = registry.get("CURRENT_FILE")
registry.set("CURRENT_FILE", new_path)
```

**典型状态变量**:
| 变量名 | 用途 | 由谁设置 |
|--------|------|---------|
| `CURRENT_FILE` | 当前打开的文件路径 | windowed/open |
| `USE_FILEMAP` | 是否启用filemap | 配置 |
| `EDITOR_TYPE` | 编辑器类型 | 配置 |

**状态持久化机制**:
1. 存储位置: `/root/.registry.json`
2. 跨命令保持: 即使shell会话结束，状态仍然保留
3. 状态收集: 每个bundle的`_state`命令读取registry并返回

---

#### 6.3.9 submit - 解决方案提交

**路径**: `tools/submit/`

**核心命令**: `submit`

**功能**: 标记任务完成并提交解决方案

**参数**:
```yaml
submit:
  signature: "submit"
  arguments: []
```

**变体**:
- `tools/submit/`: 基础版本
- `tools/review_on_submit_m/`: 带审查功能的版本（可配置`-f`强制提交）

**使用场景**:
- 代码修复完成后提交
- 测试通过后标记完成
- 触发评估流程

---

#### 6.3.10 forfeit - 放弃任务

**路径**: `tools/forfeit/`

**核心命令**: `exit_forfeit`

**功能**: 放弃当前挑战并终止会话

```yaml
exit_forfeit:
  signature: "exit_forfeit"
  docstring: "Give up on the current challenge and terminate the session."
```

---

#### 6.3.11 web_browser - 浏览器自动化

**路径**: `tools/web_browser/`

**核心命令列表**:

| 类别 | 命令 | 功能 |
|------|------|------|
| **导航** | `open_site` | 打开URL或本地文件 |
| | `close_site` | 关闭浏览器 |
| | `navigate_back` | 后退 |
| | `navigate_forward` | 前进 |
| | `reload_page` | 刷新页面 |
| **鼠标** | `click_mouse` | 单击指定坐标 |
| | `double_click_mouse` | 双击 |
| | `move_mouse` | 移动鼠标 |
| | `drag_mouse` | 拖拽（路径JSON格式） |
| **键盘** | `type_text` | 输入文本 |
| | `press_keys_on_page` | 按键组合（JSON格式） |
| **查看** | `screenshot_site` | 截图 |
| | `scroll_on_page` | 滚动页面 |
| | `set_browser_window_size` | 设置窗口大小 |
| **高级** | `execute_script_on_page` | 执行JavaScript |
| | `get_console_output` | 获取控制台日志 |
| | `wait_time` | 等待指定毫秒 |

**详细参数示例**:

```yaml
click_mouse:
  signature: "click_mouse <x> <y> [<button>]"
  arguments:
    - name: x
      type: integer
    - name: y
      type: integer
    - name: button
      type: string
      enum: ["left", "right"]
      default: "left"

drag_mouse:
  signature: "drag_mouse <path>"
  arguments:
    - name: path
      type: string
      description: "JSON格式路径: [[x1,y1],[x2,y2],...]"

press_keys_on_page:
  signature: "press_keys_on_page <keys>"
  arguments:
    - name: keys
      type: string
      description: "JSON格式: [\"ctrl\", \"c\"]"
```

**使用示例**:

```bash
# 打开网站
open_site "https://example.com"

# 点击坐标(100, 200)
click_mouse 100 200

# 右键点击
click_mouse 100 200 right

# 输入文本
type_text "Hello World"

# 按键组合（Ctrl+C）
press_keys_on_page '["ctrl", "c"]'

# 截图
screenshot_site

# 执行JavaScript
execute_script_on_page "document.title"

# 获取控制台输出
get_console_output

# 等待2秒
wait_time 2000

# 关闭浏览器
close_site
```

**配套组件**:
- `run_web_browser_server`: 浏览器服务器进程
- `lib/browser_manager.py`: 浏览器管理
- `lib/web_browser_utils.py`: 工具函数

---

#### 6.3.12 image_tools - 图像查看

**路径**: `tools/image_tools/`

**核心命令**: `view_image`

**功能**: 在环境中查看图像文件

```yaml
view_image:
  signature: "view_image <image_file>"
  arguments:
    - name: image_file
      type: string
      required: true
```

**使用场景**: 查看截图、图表、生成的图像等

---

#### 6.3.13 diff_state - 状态对比

**路径**: `tools/diff_state/`

**核心功能**: 捕获和对比环境状态变化

**状态命令**: `_state_diff_state`

**用途**: 在执行操作前后对比文件系统状态变化

---

#### 6.3.14 multilingual_setup - 多语言支持

**路径**: `tools/multilingual_setup/`

**功能**: 为多种编程语言提供环境设置支持

**命令**: `do_nothing`（占位符，实际配置在install.sh中）

---

### 6.4 工具组合使用模式

#### 模式1: 代码修复工作流

```bash
# 1. 查找问题文件
find_file "*.py" | grep test

# 2. 打开文件查看
open "/repo/src/buggy.py"

# 3. 搜索相关代码
search_file "def problematic_function"

# 4. 使用edit_anthropic编辑
str_replace_editor view /repo/src/buggy.py --view_range "1 50"
str_replace_editor str_replace /repo/src/buggy.py \
  --old_str "def old_code():" \
  --new_str "def fixed_code():"

# 5. 运行测试验证
bash "pytest tests/test_buggy.py -v"

# 6. 提交
submit
```

#### 模式2: 窗口化浏览+编辑

```bash
# 1. 打开文件
open "/repo/src/large_file.py"

# 2. 滚动浏览
scroll_down
scroll_down

# 3. 跳转到特定行
goto 150

# 4. 在当前视图内编辑
edit "old_pattern" "new_pattern"

# 5. 继续浏览
scroll_up
```

#### 模式3: 全仓库搜索

```bash
# 1. 搜索所有包含TODO的文件
search_dir "TODO" "/repo"

# 2. 查看找到的文件
filemap "/repo/src/todo_file.py"

# 3. 打开详细查看
open "/repo/src/todo_file.py"

# 4. 精确定位
search_file "TODO: fix this"
```

---

### 6.5 工具选择决策树

```
需要操作文件？
├── 是 → 需要编辑？
│   ├── 是 → 精确替换？
│   │   ├── 是 → str_replace_editor (edit_anthropic)
│   │   └── 否 → windowed + edit (搜索替换)
│   └── 否 → 查看文件？
│       ├── 是 → 大文件？
│       │   ├── 是 → filemap 或 windowed
│       │   └── 否 → str_replace_editor view
│       └── 否 → 创建文件？
│           └── 是 → str_replace_editor create
│
├── 否 → 需要搜索？
│   ├── 是 → 搜索文件内容？
│   │   ├── 是 → 指定文件？
│   │   │   ├── 是 → search_file
│   │   │   └── 否 → search_dir
│   │   └── 否 → find_file
│   └── 否 → 需要执行命令？
│       └── 是 → bash
│
└── 否 → 浏览器操作？
    └── 是 → web_browser 工具集
```

---

## 7. 解析器系统

### 7.1 解析器架构

```python
# sweagent/tools/parsing.py
class ParseFunction(BaseModel):
    """Parser configuration"""
    type: str  # Parser type identifier

class HistoryProcessor(ABC):
    """Base class for all parsers"""

    @abstractmethod
    def __call__(
        self,
        model_response: str,
        command_manager: ToolHandler,
    ) -> list[ParsedCommand]:
        """Parse model output into executable commands"""
        pass
```

### 7.2 内置解析器

| 解析器 | 类型 | 用途 | 输出格式 |
|--------|------|------|---------|
| `FunctionCallingParser` | `function_calling` | OpenAI/Anthropic API原生格式 | JSON tool calls |
| `ThoughtActionParser` | `thought_action` | Markdown代码块格式 | ```action\ncommand\n``` |
| `XMLThoughtActionParser` | `xml` | XML标签格式 | <command>args</command> |
| `JsonParser` | `json` | JSON格式 | {"command": "...", "args": {...}} |
| `ActionParser` | `action` | 纯动作格式 | action command |
| `Identity` | `identity` | 透传原始输出 | 无修改 |

### 7.3 Function Calling解析器

```python
# sweagent/tools/parsing.py
class FunctionCallingParser(HistoryProcessor):
    """
    Primary parser for OpenAI/Anthropic function calling API.
    Expects LiteLLM format with tool_calls.
    """

    def __call__(self, model_response, command_manager):
        # Parse JSON function calls from model output
        tool_calls = parse_json(model_response)

        commands = []
        for call in tool_calls:
            # Convert to ParsedCommand
            cmd = ParsedCommand(
                name=call["name"],
                args=call["arguments"],
            )

            # Validate against command definition
            command_def = command_manager.get_command(cmd.name)
            self._validate_args(cmd.args, command_def.arguments)

            # Format as executable string
            cmd.raw_command = self._format_command(cmd, command_def)
            commands.append(cmd)

        return commands
```

### 7.4 Thought-Action解析器

```python
# sweagent/tools/parsing.py
class ThoughtActionParser(HistoryProcessor):
    """
    Parses markdown-style thought and action blocks.
    Common format for older models or custom fine-tunes.
    """

    pattern = re.compile(
        r"(.*?)"  # Thought (captured)
        r"<((?:output|command))>"  # Opening tag
        r"(.*?)"  # Command content
        r"</\2>",  # Closing tag
        re.DOTALL
    )

    def __call__(self, model_response, command_manager):
        matches = self.pattern.findall(model_response)
        commands = []

        for thought, tag, content in matches:
            # Parse command name and arguments
            lines = content.strip().split("\n")
            cmd_name = lines[0].strip()
            args = self._parse_args(lines[1:])

            commands.append(ParsedCommand(
                name=cmd_name,
                args=args,
                raw_command=content.strip()
            ))

        return commands
```

---

## 8. 实现参考代码

### 8.1 最小Bundle实现模板

```yaml
# tools/my_tool/config.yaml
tools:
  hello:
    signature: "hello <name>"
    docstring: |
      Greet a user by name.
      Example: hello "World"
    arguments:
      - name: name
        type: string
        description: "Name to greet"
        required: true

state_command: "_state"
```

```python
# tools/my_tool/bin/hello
#!/usr/bin/env python3
"""Simple greeting tool"""
import argparse
import sys

def main():
    parser = argparse.ArgumentParser(description="Greet a user")
    parser.add_argument("name", help="Name to greet")
    args = parser.parse_args()

    print(f"Hello, {args.name}!")
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

```python
# tools/my_tool/bin/_state
#!/usr/bin/env python3
"""Return current state"""
import json

state = {
    "last_greeting": "World",
    "greeting_count": 1
}
print(json.dumps(state))
```

```bash
# tools/my_tool/install.sh
#!/bin/bash
# Optional: Install dependencies
pip install -q my-dependency || true
echo "My tool bundle installed"
```

### 8.2 自定义解析器实现

```python
# my_parser.py
from sweagent.tools.parsing import HistoryProcessor, ParsedCommand

class MyCustomParser(HistoryProcessor):
    """Custom parser for specific model output format"""

    def __call__(self, model_response, command_manager):
        commands = []

        # Your parsing logic here
        lines = model_response.strip().split("\n")

        for line in lines:
            if line.startswith("CMD:"):
                parts = line[4:].strip().split()
                cmd_name = parts[0]
                args = {"arg": parts[1]} if len(parts) > 1 else {}

                commands.append(ParsedCommand(
                    name=cmd_name,
                    args=args
                ))

        return commands
```

### 8.3 ToolHandler核心实现

```python
# sweagent/tools/tools.py (simplified)
class ToolHandler:
    """Main interface for tool management"""

    def __init__(self, config: ToolConfig, env: Environment):
        self.config = config
        self.env = env
        self.bundles: list[Bundle] = []
        self.commands: dict[str, Command] = {}

    def install(self) -> None:
        """Install all configured bundles to environment"""
        for bundle_config in self.config.bundles:
            bundle = Bundle(bundle_config.path)
            self._install_bundle(bundle)
            self.bundles.append(bundle)

    def _install_bundle(self, bundle: Bundle) -> None:
        """Install single bundle"""
        # 1. Upload bin/ directory
        self.env.upload(
            src=bundle.bin_path,
            dst="/opt/tools/",
            recursive=True
        )

        # 2. Make executables
        for script in bundle.bin_path.iterdir():
            self.env.execute(f"chmod +x /opt/tools/{script.name}")

        # 3. Run install.sh if exists
        if bundle.install_script:
            self.env.execute("bash /opt/tools/install.sh")

        # 4. Register commands
        for name, tool_config in bundle.config.tools.items():
            cmd = Command.from_config(name, tool_config)
            self.commands[name] = cmd

    def parse_actions(self, model_response: str) -> list[ParsedCommand]:
        """Parse model output into commands"""
        parser = self._get_parser()
        return parser(model_response, self)

    def execute(self, command: ParsedCommand) -> CommandResult:
        """Execute a parsed command"""
        # Check filters
        if block := self.should_block_action(command.raw_command):
            return CommandResult(
                output="",
                error=block.message,
                exit_code=1
            )

        # Execute in environment
        result = self.env.execute(command.raw_command)

        # Collect state
        state = self.get_state()

        return CommandResult(
            output=result.output,
            error=result.error,
            exit_code=result.exit_code,
            state=state
        )

    def get_state(self) -> dict[str, Any]:
        """Collect state from all bundles"""
        state = {}
        for bundle in self.bundles:
            if bundle.config.state_command:
                result = self.env.execute(
                    f"/opt/tools/{bundle.config.state_command}"
                )
                if result.exit_code == 0:
                    state.update(json.loads(result.output))
        return state
```

---

## 9. 最佳实践总结

### 9.1 Bundle设计原则

1. **单一职责**：每个Bundle聚焦一个功能领域
   - ✅ `edit_anthropic`: 文件编辑操作
   - ✅ `search`: 搜索和查找
   - ❌ `utils`: 混杂多种不相关功能

2. **自包含**：Bundle应包含所有依赖
   ```
   my_bundle/
   ├── bin/           # 可执行脚本
   ├── lib/           # 依赖库
   ├── install.sh     # 安装脚本
   └── config.yaml    # 配置
   ```

3. **状态透明**：通过`_state`命令暴露内部状态
   ```python
   # _state 返回JSON
   {
     "current_file": "/repo/main.py",
     "cursor_line": 42
   }
   ```

### 9.2 命令设计原则

1. **明确的签名**：参数顺序和命名清晰
   ```yaml
   # Good
   signature: "search_file <pattern> [<path>]"

   # Bad
   signature: "search <args...>"  # 太模糊
   ```

2. **全面的文档**：docstring应包含使用示例
   ```yaml
   docstring: |
     Search for pattern in files.

     Examples:
       search_file "def main" src/
       search_file "TODO" --file_extension=py
   ```

3. **合理的默认值**：减少LLM决策负担
   ```yaml
   arguments:
     - name: lines
       type: integer
       default: 50  # 合理的默认值
   ```

### 9.3 安全最佳实践

1. **使用blocklist阻止危险命令**
   ```yaml
   filter:
     blocklist:
       - vim      # 交互式编辑器
       - sudo     # 特权提升
       - curl | sh  # 管道执行
   ```

2. **验证用户输入**
   ```python
   # 在工具脚本中验证路径
   if ".." in file_path or file_path.startswith("/"):
       print("Error: Invalid path")
       sys.exit(1)
   ```

3. **限制输出大小**
   ```python
   # 截断大输出
   MAX_OUTPUT = 10000
   if len(output) > MAX_OUTPUT:
       output = output[:MAX_OUTPUT] + "\n... [truncated]"
   ```

### 9.4 LLM体验优化

1. **提供清晰的错误信息**
   ```python
   # Good
   print("Error: File not found. Use 'search_file' to find files.")

   # Bad
   print("Error")  # 太模糊
   ```

2. **支持常见操作模式**
   - 文件编辑：提供`view`命令让LLM先查看内容
   - 搜索：提供`search_file`和`search_dir`两种粒度

3. **渐进式披露**：复杂功能通过多个简单命令组合
   ```
   # 而非单个复杂的edit命令
   open file.py    # 打开文件
   goto 50         # 定位到行
   scroll down     # 浏览内容
   ```

---

## 附录：核心文件结构

```
sweagent/
├── tools/                      # Bundle定义目录
│   ├── registry/
│   ├── edit_anthropic/
│   ├── windowed/
│   ├── search/
│   ├── submit/
│   └── ...
│
├── sweagent/tools/             # 工具管理代码
│   ├── __init__.py
│   ├── bundle.py              # Bundle类定义
│   ├── commands.py            # Command/Argument类
│   ├── parsing.py             # 解析器实现
│   ├── tools.py               # ToolHandler/ToolConfig
│   └── ...
│
├── config/                     # 配置示例
│   ├── default.yaml
│   ├── anthropic_claude.yaml
│   └── ...
│
└── sweagent/agent/
    ├── agents.py              # Agent集成
    └── models.py              # 模型集成
```

---

*文档版本：1.0*
*基于：SWE-agent main branch (2025)*
*用途：SWE-agent工具开发参考*
