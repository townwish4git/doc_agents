# Crush 工具链管理参考文档

> 本文档总结了Crush的Agent工具调用范式，供理解Crush架构和开发参考。
>
> 涵盖内容：工具架构设计、调用流程、权限系统、实现细节

---

## 目录

1. [架构概览](#1-架构概览)
2. [工具定义系统](#2-工具定义系统)
3. [完整调用链](#3-完整调用链)
4. [权限控制系统](#4-权限控制系统)
5. [内置工具参考](#5-内置工具参考)
6. [MCP工具集成](#6-mcp工具集成)
7. [子Agent工具设计](#7-子agent工具设计)
8. [实现参考代码](#8-实现参考代码)
9. [最佳实践总结](#9-最佳实践总结)

---

## 1. 架构概览

### 1.1 核心设计原则

```
┌─────────────────────────────────────────────────────────────────┐
│                     Crush 工具链核心原则                         │
├─────────────────────────────────────────────────────────────────┤
│ 1. 基于Fantasy框架：构建于charm.land/fantasy v0.11.0             │
│ 2. 安全第一：所有操作经过权限检查，用户有最终控制权               │
│ 3. 结构化交互：工具返回结构化数据，支持文本/图片/错误             │
│ 4. 可扩展性：支持MCP协议，动态加载外部工具                        │
│ 5. 并发控制：支持后台作业和并行子Agent执行                        │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Agent 层 (internal/agent/)                         │
│  - Session管理 (sessionAgent)                                │
│  - 对话流管理 (Run方法)                                       │
│  - 工具选择决策（由LLM通过fantasy框架完成）                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 协调层 (Coordinator)                               │
│  - 工具注册与发现 (buildTools)                                │
│  - Agent生命周期管理                                         │
│  - MCP工具动态加载                                           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 执行层 (internal/agent/tools/)                     │
│  - 具体工具实现 (33+ tools)                                  │
│  - 权限检查 (Permission System)                              │
│  - 输出处理 (截断、格式化)                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 工具定义系统

### 2.1 核心抽象：fantasy.AgentTool

Crush构建于`charm.land/fantasy`框架之上，使用其提供的工具接口：

```go
// fantasy.AgentTool - 工具接口定义
// 由fantasy框架提供，Crush实现具体工具

// AgentTool定义结构
fantasy.NewAgentTool(
    name string,                    // 工具名称
    description string,             // 工具描述（给LLM阅读）
    fn AgentToolFunc,               // 执行函数
)

// AgentToolFunc 签名
type AgentToolFunc func(
    ctx context.Context,
    params json.RawMessage,
    call fantasy.ToolCall,
) (fantasy.ToolResponse, error)
```

### 2.2 工具注册中心

**文件**: `/Users/townwish/Workspace/sources/crush/internal/agent/coordinator.go`

```go
// buildTools - 构建所有可用工具 (lines 404-494)
func (c *coordinator) buildTools(
    agent agent.Agent,
    allowList []string,
) []fantasy.AgentTool {
    var tools []fantasy.AgentTool

    // 定义所有内置工具
    allTools := map[string]fantasy.AgentTool{
        BashToolName:           tools.NewBashTool(c.permissions, c.bashTool, c.clock, c.opts.BashOpts...),
        ViewToolName:           tools.NewViewTool(c.permissions, c.lspService, c.opts.ReadMaxTokens),
        WriteToolName:          tools.NewWriteTool(c.permissions, c.opts.WriteMaxTokens),
        EditToolName:           tools.NewEditTool(c.permissions, c.lspService, c.opts.EditMaxTokens),
        MultiEditToolName:      tools.NewMultiEditTool(c.permissions, c.lspService),
        GlobToolName:           tools.NewGlobTool(),
        GrepToolName:           tools.NewGrepTool(),
        LSName:                 tools.NewLsTool(),
        FetchToolName:          tools.NewFetchTool(c.permissions),
        DownloadToolName:       tools.NewDownloadTool(c.permissions),
        TodoWriteToolName:      tools.NewTodosWriteTool(c.todoStore),
        WebSearchToolName:      tools.NewWebSearchTool(c.web),
        WebFetchToolName:       tools.NewWebFetchTool(c.web, c.permissions),
        SourcegraphToolName:    tools.NewSourcegraphTool(c.permissions),
        AgentToolName:          NewAgentTool(c, agent, c.subagentModel),  // 子Agent
        AgenticFetchToolName:   NewAgenticFetchTool(c, agent),            // 智能获取
        JobOutputToolName:      tools.NewJobOutputTool(c.shell),
        JobKillToolName:        tools.NewJobKillTool(c.shell),
        DiagnosticsToolName:    tools.NewDiagnosticsTool(c.lspService),
        ReferencesToolName:     tools.NewReferencesTool(c.lspService),
        LspRestartToolName:     tools.NewLspRestartTool(c.lspService),
        ListMcpResourcesToolName:   tools.NewListMcpResourcesTool(c.mcpToolProvider),
        ReadMcpResourceToolName:    tools.NewReadMcpResourceTool(c.mcpToolProvider),
    }

    // 根据allowList过滤工具
    for _, name := range allowList {
        if tool, ok := allTools[name]; ok {
            tools = append(tools, tool)
        }
    }

    // 添加MCP工具（动态加载）
    if mcpTools, err := c.mcpToolProvider.GetMCPTools(); err == nil {
        tools = append(tools, mcpTools...)
    }

    return tools
}
```

### 2.3 工具定义示例

**文件**: `/Users/townwish/Workspace/sources/crush/internal/agent/tools/view.go`

```go
package tools

import (
    "context"
    "encoding/json"
    "fmt"

    "charm.land/fantasy"
)

const ViewToolName = "view"

// ViewParams - 参数结构体
type ViewParams struct {
    FilePath string `json:"file_path"`
    Offset   *int   `json:"offset,omitempty"`
    Limit    *int   `json:"limit,omitempty"`
}

// NewViewTool - 创建工具
func NewViewTool(
    permissions PermissionService,
    lspService LSPService,
    maxTokens int64,
) fantasy.AgentTool {
    return fantasy.NewAgentTool(
        ViewToolName,
        viewDescription,  // 工具描述（给LLM看的说明）
        func(ctx context.Context, params json.RawMessage, call fantasy.ToolCall) (fantasy.ToolResponse, error) {
            // 1. 解析参数
            var viewParams ViewParams
            if err := json.Unmarshal(params, &viewParams); err != nil {
                return nil, fmt.Errorf("failed to parse view params: %w", err)
            }

            // 2. 权限检查
            if err := permissions.Request(ctx, PermissionRequest{
                Tool:      ViewToolName,
                Operation: "read",
                Path:      viewParams.FilePath,
            }); err != nil {
                return nil, err
            }

            // 3. 执行读取
            content, err := readFile(viewParams.FilePath, viewParams.Offset, viewParams.Limit)
            if err != nil {
                return fantasy.NewTextErrorResponse(err.Error()), nil
            }

            // 4. 返回结构化结果
            return fantasy.NewTextResponse(content), nil
        },
    )
}

// 工具描述（供LLM理解何时使用）
const viewDescription = `
Read a file from the local filesystem.

Parameters:
- file_path: Absolute path to the file (required)
- offset: Line number to start reading from (1-indexed, optional)
- limit: Maximum number of lines to read (optional)

Use this tool to read source code, configuration files, or any text files.
`
```

---

## 3. 完整调用链

### 3.1 调用流程图

```
用户输入: "帮我读取 main.go 的内容"
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Coordinator 构建Agent                                │
│                                                             │
│ 1. 调用buildTools()获取可用工具列表                           │
│ 2. 创建sessionAgent实例                                       │
│ 3. 设置工具：agent.SetTools(tools)                            │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Agent 构建 Prompt                                    │
│                                                             │
│ System: 你是Crush助手，可以使用以下工具：                      │
│ - view: 读取文件，参数：file_path, offset, limit             │
│ - write: 写入文件，参数：file_path, content                  │
│ - bash: 执行命令，参数：command, description                 │
│ ...                                                         │
│                                                             │
│ User: 帮我读取 main.go 的内容                                │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: LLM 决策 (通过fantasy框架)                           │
│                                                             │
│ LLM分析：                                                    │
│ - 意图：读取文件                                             │
│ - 目标文件：main.go                                          │
│ - 匹配工具：view                                             │
│ - 参数推导：file_path = "/path/to/main.go"                   │
│                                                             │
│ 输出工具调用：                                                │
│ {                                                           │
│   "tool": "view",                                           │
│   "params": {                                               │
│     "file_path": "/path/to/main.go"                         │
│   }                                                         │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: 参数验证与反序列化                                    │
│                                                             │
│ - json.Unmarshal 验证JSON格式                               │
│ - 检查必需字段                                               │
│ - 类型转换                                                   │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: 权限检查                                              │
│                                                             │
│ permissions.Request(ctx, PermissionRequest{                 │
│   Tool: "view",                                             │
│   Operation: "read",                                        │
│   Path: "/path/to/main.go"                                  │
│ })                                                          │
│                                                             │
│ 决策：                                                       │
│ - 规则允许 → 直接执行                                        │
│ - 需要确认 → 提示用户                                        │
│ - 规则拒绝 → 返回权限错误                                     │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: 工具执行                                              │
│                                                             │
│ 调用 view tool 的 AgentToolFunc：                             │
│ 1. 读取文件内容                                             │
│ 2. 应用offset和limit                                        │
│ 3. 格式化输出                                                │
│                                                             │
│ 结果：                                                       │
│ fantasy.NewTextResponse("package main\n...")               │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 7: 结果处理与回调                                        │
│                                                             │
│ agent.go中的回调：                                           │
│ - OnToolCall: 工具调用完成                                   │
│ - OnToolResult: 工具结果处理                                 │
│ - convertToToolResult: 转换为ToolResult                     │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 8: 结果返回给LLM                                         │
│                                                             │
│ LLM 生成自然语言回复：                                        │
│ "main.go 的内容如下：
 package main
 ..."                    │
└─────────────────────────────────────────────────────────────┘
           ↓
用户看到回复
```

### 3.2 关键数据结构

```go
// ToolCall - 工具调用请求（来自LLM）
type ToolCall struct {
    ID       string          `json:"id"`
    Function struct {
        Name      string          `json:"name"`
        Arguments json.RawMessage `json:"arguments"`
    } `json:"function"`
}

// ToolResponse - 工具响应接口
// 由fantasy框架定义，Crush实现：
// - NewTextResponse(content string) ToolResponse
// - NewTextErrorResponse(err string) ToolResponse
// - NewImageResponse(data []byte, mediaType string) ToolResponse
// - WithResponseMetadata(resp ToolResponse, key, value string) ToolResponse

// PermissionRequest - 权限请求
type PermissionRequest struct {
    Tool      string // 工具名称
    Operation string // 操作类型 (read/write/execute等)
    Path      string // 目标路径
    Metadata  map[string]interface{} // 附加信息
}

// SessionAgentCall - Agent调用记录
// internal/agent/agent.go: lines 138-145
type SessionAgentCall struct {
    Request  anthropic.Message
    Response anthropic.Message
    Results  []message.ToolResult
}
```

---

## 4. 权限控制系统

### 4.1 权限检查流程

**文件**: `/Users/townwish/Workspace/sources/crush/internal/permission/`

```go
// PermissionService 接口
// internal/agent/tools/tools.go
type PermissionService interface {
    Request(ctx context.Context, req PermissionRequest) error
}

// PermissionRequest 结构
type PermissionRequest struct {
    Tool      string
    Operation string
    Path      string
    Metadata  map[string]interface{}
}
```

**权限检查示例** (bash.go lines 217-235):

```go
func (t *bashTool) execute(ctx context.Context, params BashParams) (fantasy.ToolResponse, error) {
    // 1. 检查参数安全性
    if err := t.checkDangerousParams(params); err != nil {
        return fantasy.NewTextErrorResponse(err.Error()), nil
    }

    // 2. 请求权限
    if err := t.permissions.Request(ctx, PermissionRequest{
        Tool:      BashToolName,
        Operation: "execute",
        Path:      params.Command,
        Metadata: map[string]interface{}{
            "command": params.Command,
            "timeout": params.Timeout,
        },
    }); err != nil {
        return fantasy.NewTextErrorResponse(err.Error()), nil
    }

    // 3. 执行命令
    // ...
}
```

### 4.2 权限决策流程

```
┌─────────────────────────────────────────────┐
│            权限请求                          │
│  Tool: bash, Path: "git status"             │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│ 1. 检查安全规则                              │
│    - 危险命令黑名单                          │
│    - 破坏性操作检测                          │
└─────────────────────────────────────────────┘
                    ↓
            ┌───────┴───────┐
            ↓               ↓
        ┌───────┐      ┌─────────┐
        │ 拒绝  │      │ 继续检查 │
        └───────┘      └────┬────┘
                            ↓
            ┌───────────────────────────────┐
            │ 2. 检查配置规则                │
            │    - 读取opencode.json         │
            │    - 匹配allow/ask/deny规则    │
            └────────────────┬──────────────┘
                             ↓
            ┌───────┬────────┴────────┬───────┐
            ↓       ↓                 ↓       ↓
        ┌──────┐ ┌──────┐        ┌──────┐ ┌──────┐
        │allow │ │ ask  │        │ deny │ │默认ask│
        └──┬───┘ └──┬───┘        └──┬───┘ └──┬───┘
           ↓       ↓               ↓        ↓
        ┌──────┐ ┌──────┐      ┌────────┐ ┌──────┐
        │执行  │ │提示用户│      │拒绝执行│ │提示用户│
        └──────┘ └──────┘      └────────┘ └──────┘
```

---

## 5. 内置工具参考

### 5.1 文件操作工具

| 工具 | 用途 | 核心参数 | 文件位置 |
|------|------|---------|---------|
| `view` | 读取文件 | `file_path`, `offset`, `limit` | `tools/view.go` |
| `write` | 创建/覆盖文件 | `file_path`, `content` | `tools/write.go` |
| `edit` | 编辑文件（字符串替换） | `file_path`, `old_string`, `new_string` | `tools/edit.go` |
| `multi_edit` | 批量编辑多个文件 | `edits[]` | `tools/multiedit.go` |
| `glob` | 文件模式匹配 | `pattern` | `tools/glob.go` |
| `grep` | 内容搜索 | `pattern`, `path`, `output_mode` | `tools/grep.go` |
| `ls` | 列出目录 | `path` | `tools/ls.go` |

### 5.2 命令执行工具

| 工具 | 用途 | 核心参数 | 文件位置 |
|------|------|---------|---------|
| `bash` | 执行shell命令 | `command`, `description`, `timeout` | `tools/bash.go` |
| `job_output` | 获取后台作业输出 | `job_id` | `tools/job_output.go` |
| `job_kill` | 终止后台作业 | `job_id` | `tools/job_kill.go` |

### 5.3 任务管理工具

| 工具 | 用途 | 核心参数 | 文件位置 |
|------|------|---------|---------|
| `todo_write` | 待办事项管理 | `todos[]` | `tools/todos.go` |
| `agent_tool` | 创建子Agent | `description`, `prompt`, `subagent_type` | `agent_tool.go` |

### 5.4 LSP相关工具

| 工具 | 用途 | 核心参数 | 文件位置 |
|------|------|---------|---------|
| `diagnostics` | 获取诊断信息 | `file_path` | `tools/diagnostics.go` |
| `references` | 查找引用 | `file_path`, `line`, `offset` | `tools/references.go` |
| `lsp_restart` | 重启LSP服务器 | - | `tools/lsp_restart.go` |

### 5.5 Web与搜索工具

| 工具 | 用途 | 核心参数 | 文件位置 |
|------|------|---------|---------|
| `web_search` | 网络搜索 | `query` | `tools/web_search.go` |
| `web_fetch` | 获取网页内容 | `url` | `tools/web_fetch.go` |
| `fetch` | HTTP请求 | `url`, `method`, `body` | `tools/fetch.go` |
| `download` | 下载文件 | `url`, `file_path` | `tools/download.go` |
| `agentic_fetch` | 智能网页获取 | `url` | `agentic_fetch_tool.go` |
| `sourcegraph` | Sourcegraph搜索 | `query` | `tools/sourcegraph.go` |

### 5.6 MCP工具

| 工具 | 用途 | 文件位置 |
|------|------|---------|
| `list_mcp_resources` | 列出MCP资源 | `tools/list_mcp_resources.go` |
| `read_mcp_resource` | 读取MCP资源 | `tools/read_mcp_resource.go` |
| (动态工具) | MCP服务器提供的工具 | `tools/mcp-tools.go` |

---

## 6. MCP工具集成

### 6.1 MCP架构

MCP (Model Context Protocol) 允许Crush动态加载外部工具。

```
┌─────────────────────────────────────────────┐
│              Crush Core                      │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │      MCPToolProvider                  │   │
│  │  - GetMCPTools() []AgentTool          │   │
│  │  - RefreshTools()                     │   │
│  │  - ExecuteTool()                      │   │
│  └──────────────┬───────────────────────┘   │
│                 │                            │
│                 ↓ MCP Protocol               │
│  ┌──────────────────────────────────────┐   │
│  │      MCP Client (tools/mcp/)          │   │
│  │  - JSON-RPC通信                       │   │
│  │  - Tool Discovery                     │   │
│  │  - Tool Execution                     │   │
│  └──────────────┬───────────────────────┘   │
│                 │                            │
│                 ↓ Stdio/HTTP                 │
│  ┌──────────────────────────────────────┐   │
│  │      MCP Servers                      │   │
│  │  - Filesystem MCP                     │   │
│  │  - Git MCP                            │   │
│  │  - Database MCP                       │   │
│  │  - 自定义MCP服务器                     │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### 6.2 MCP工具包装器

**文件**: `/Users/townwish/Workspace/sources/crush/internal/agent/tools/mcp-tools.go`

```go
// WrapMCPTool - 将MCP工具包装为fantasy.AgentTool
func WrapMCPTool(
    serverName string,
    tool mcp.Tool,
    executeFn func(ctx context.Context, toolName string, args map[string]interface{}) (*mcp.CallToolResult, error),
) fantasy.AgentTool {
    return fantasy.NewAgentTool(
        tool.Name,
        tool.Description,
        func(ctx context.Context, params json.RawMessage, call fantasy.ToolCall) (fantasy.ToolResponse, error) {
            // 1. 解析参数
            var args map[string]interface{}
            if err := json.Unmarshal(params, &args); err != nil {
                return nil, err
            }

            // 2. 调用MCP服务器
            result, err := executeFn(ctx, tool.Name, args)
            if err != nil {
                return fantasy.NewTextErrorResponse(err.Error()), nil
            }

            // 3. 转换结果
            return convertMCPResultToResponse(result), nil
        },
    )
}
```

---

## 7. 子Agent工具设计

### 7.1 子Agent架构

**文件**: `/Users/townwish/Workspace/sources/crush/internal/agent/agent_tool.go`

```go
// agentTool - 创建子Agent执行并行任务
type agentTool struct {
    coordinator   *coordinator
    parentAgent   agent.Agent
    subagentModel *atomic.Pointer[Model]
}

// NewAgentTool - 创建子Agent工具
func NewAgentTool(
    coord *coordinator,
    parentAgent agent.Agent,
    subagentModel *atomic.Pointer[Model],
) fantasy.AgentTool {
    return fantasy.NewAgentTool(
        AgentToolName,
        agentToolDescription,
        func(ctx context.Context, params json.RawMessage, call fantasy.ToolCall) (fantasy.ToolResponse, error) {
            // 解析AgentToolParams
            // 创建新的子Agent会话
            // 在后台执行子任务
            // 返回结果或作业ID
        },
    )
}
```

### 7.2 子Agent调用流程

```
主Agent调用agent_tool
        ↓
创建子Agent (subagent_type决定类型)
        ↓
┌─────────────────┬─────────────────┐
↓                 ↓                 ↓
Bash Agent    Explore Agent    Plan Agent
(命令执行)     (代码探索)       (计划模式)
        ↓
子Agent独立执行，拥有自己的：
- 消息历史
- 工具集
- LLM模型
        ↓
返回结果给父Agent
```

---

## 8. 实现参考代码

### 8.1 最小工具实现模板

```go
// mytool.go
package tools

import (
    "context"
    "encoding/json"
    "fmt"

    "charm.land/fantasy"
)

const MyToolName = "my_tool"

type MyToolParams struct {
    Param1 string `json:"param1"`
    Param2 int    `json:"param2,omitempty"`
}

func NewMyTool(permissions PermissionService) fantasy.AgentTool {
    return fantasy.NewAgentTool(
        MyToolName,
        myToolDescription,
        func(ctx context.Context, params json.RawMessage, call fantasy.ToolCall) (fantasy.ToolResponse, error) {
            // 1. 解析参数
            var p MyToolParams
            if err := json.Unmarshal(params, &p); err != nil {
                return nil, fmt.Errorf("parse params: %w", err)
            }

            // 2. 权限检查
            if err := permissions.Request(ctx, PermissionRequest{
                Tool:      MyToolName,
                Operation: "execute",
                Path:      p.Param1,
            }); err != nil {
                return fantasy.NewTextErrorResponse(err.Error()), nil
            }

            // 3. 执行操作
            result, err := doSomething(p.Param1, p.Param2)
            if err != nil {
                return fantasy.NewTextErrorResponse(err.Error()), nil
            }

            // 4. 返回结果
            return fantasy.NewTextResponse(result), nil
        },
    )
}

const myToolDescription = `
Description of what this tool does.

Parameters:
- param1: Description of param1 (required)
- param2: Description of param2 (optional)

Usage guidelines for the LLM.
`
```

### 8.2 Agent执行流程核心代码

**文件**: `/Users/townwish/Workspace/sources/crush/internal/agent/agent.go`

```go
// Run - Agent主执行循环 (lines 167-393)
func (a *sessionAgent) Run(ctx context.Context, req AgentRunRequest) error {
    // 1. 获取模型和工具
    largeModel := a.largeModel.Get()
    tools := a.tools.Slice()

    // 2. 构建系统提示词
    systemPrompt := a.buildSystemPrompt(req)

    // 3. 创建fantasy Agent
    fantasyAgent := fantasy.NewAgent(
        largeModel.Model,
        fantasy.WithSystemPrompt(systemPrompt),
        fantasy.WithTools(tools...),
        fantasy.WithToolChoice(fantasy.ToolChoiceAuto),
    )

    // 4. 设置回调
    callbacks := fantasy.AgentCallbacks{
        OnToolInputStart: a.handleToolInputStart,
        OnToolCall:       a.handleToolCall,
        OnToolResult:     a.handleToolResult,
        OnMessage:        a.handleMessage,
    }

    // 5. 执行对话循环
    session := fantasyAgent.NewSession(req.SessionID)
    for {
        // 获取LLM响应
        resp, err := session.Run(ctx, messages, callbacks)
        if err != nil {
            return err
        }

        // 检查是否有工具调用
        if len(resp.ToolCalls) == 0 {
            // 没有工具调用，直接返回文本响应
            break
        }

        // 处理工具调用
        for _, tc := range resp.ToolCalls {
            result := a.executeTool(ctx, tc)
            messages = append(messages, result.ToMessage())
        }
    }

    return nil
}

// executeTool - 执行单个工具 (lines 1015-1022)
func (a *sessionAgent) executeTool(ctx context.Context, tc fantasy.ToolCall) fantasy.ToolResponse {
    tool := a.findTool(tc.Function.Name)
    if tool == nil {
        return fantasy.NewTextErrorResponse(fmt.Sprintf("unknown tool: %s", tc.Function.Name))
    }

    resp, err := tool.Execute(ctx, tc.Function.Arguments, tc)
    if err != nil {
        return fantasy.NewTextErrorResponse(err.Error())
    }
    return resp
}
```

### 8.3 工具结果转换

**文件**: `/Users/townwish/Workspace/sources/crush/internal/agent/agent.go` (lines 1024-1054)

```go
// convertToToolResult - 将fantasy.ToolResponse转换为内部ToolResult
func (a *sessionAgent) convertToToolResult(
    call ToolCall,
    resp fantasy.ToolResponse,
) message.ToolResult {
    result := message.ToolResult{
        ToolCallID: call.ID,
        ToolName:   call.Function.Name,
    }

    switch v := resp.(type) {
    case *fantasy.TextResponse:
        result.Content = v.Content
        result.IsError = false

    case *fantasy.TextErrorResponse:
        result.Content = v.Error
        result.IsError = true

    case *fantasy.ImageResponse:
        result.Content = "[Image]"
        result.ImageData = v.Data
        result.MediaType = v.MediaType
        result.IsError = false
    }

    return result
}
```

---

## 9. 最佳实践总结

### 9.1 工具设计原则

1. **单一职责**：每个工具只做一件事
   - ✅ `view`: 读取文件
   - ✅ `glob`: 查找文件
   - ❌ `fileOps`: 同时处理读、写、搜索

2. **参数自描述**：清晰的参数命名和描述
   ```go
   type ViewParams struct {
       FilePath string `json:"file_path"`  // 明确是文件路径
       Offset   *int   `json:"offset,omitempty"`
   }
   ```

3. **权限检查前置**：所有可能危险的操作先检查权限
   ```go
   func (t *tool) execute(ctx context.Context, params Params) (fantasy.ToolResponse, error) {
       // 先检查权限
       if err := t.permissions.Request(ctx, req); err != nil {
           return fantasy.NewTextErrorResponse(err.Error()), nil
       }
       // 再执行操作
   }
   ```

4. **错误处理**：返回结构化错误而非抛出异常
   ```go
   return fantasy.NewTextErrorResponse("file not found: " + path), nil
   ```

### 9.2 安全最佳实践

1. **所有修改操作需要权限检查**
2. **路径验证**：确保在允许的目录内
3. **命令白名单**：限制bash可执行的命令类型
4. **超时控制**：防止长时间运行的工具阻塞

### 9.3 与OpenCode的对比

| 特性 | Crush | OpenCode |
|------|-------|----------|
| 基础框架 | Go + charm.land/fantasy | TypeScript + 自定义 |
| 参数验证 | Go struct + json.Unmarshal | Zod Schema |
| 权限系统 | 服务接口 + 配置规则 | 规则引擎 + 配置 |
| MCP支持 | ✅ 原生支持 | 视版本而定 |
| 子Agent | ✅ Task tool | ✅ Task tool |
| 响应类型 | Text/Image/Error | 结构化JSON |
| 后台作业 | ✅ 内置支持 | 视版本而定 |
| LSP集成 | ✅ 原生支持 | 实验性功能 |

---

## 附录：核心文件结构

```
/Users/townwish/Workspace/sources/crush/internal/agent/
├── agent.go                    # 核心Agent实现 (sessionAgent)
├── agent_tool.go               # 子Agent工具
├── agentic_fetch_tool.go       # 智能获取工具
├── coordinator.go              # 协调器 (buildTools, 生命周期管理)
├── templates/
│   ├── agent_tool.md           # 子Agent工具描述模板
│   └── main_system_prompt.tmpl # 系统提示词模板
│
└── tools/                      # 工具实现目录
    ├── tools.go                # 工具基础定义和上下文管理
    ├── mcp-tools.go            # MCP工具包装器
    │
    ├── view.go                 # 文件读取
    ├── write.go                # 文件写入
    ├── edit.go                 # 文件编辑
    ├── multiedit.go            # 批量编辑
    ├── glob.go                 # 文件匹配
    ├── grep.go                 # 内容搜索
    ├── ls.go                   # 目录列表
    │
    ├── bash.go                 # 命令执行
    ├── job_output.go           # 作业输出
    ├── job_kill.go             # 终止作业
    │
    ├── todos.go                # 待办管理
    │
    ├── fetch.go                # HTTP请求
    ├── download.go             # 文件下载
    ├── web_search.go           # 网络搜索
    ├── web_fetch.go            # 网页获取
    ├── sourcegraph.go          # Sourcegraph搜索
    │
    ├── diagnostics.go          # LSP诊断
    ├── references.go           # LSP引用
    ├── lsp_restart.go          # LSP重启
    │
    ├── list_mcp_resources.go   # 列出MCP资源
    ├── read_mcp_resource.go    # 读取MCP资源
    │
    └── mcp/                    # MCP客户端实现
        ├── tools.go            # MCP工具执行
        └── ...

/Users/townwish/Workspace/sources/crush/internal/permission/
├── service.go                  # 权限服务接口
└── ...
```

---

*文档版本：1.0*
*基于：Crush main branch (2025)*
*用途：Crush架构理解和开发参考*
