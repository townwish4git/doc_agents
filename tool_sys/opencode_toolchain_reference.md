# OpenCode 工具链管理参考文档

> 本文档总结了OpenCode的工具链管理范式，供开发自定义Code Agent时参考。
>
> 涵盖内容：工具架构设计、调用流程、权限系统、MCP集成、实现细节、最佳实践

---

## 目录

1. [架构概览](#1-架构概览)
2. [工具定义系统](#2-工具定义系统)
3. [完整调用链](#3-完整调用链)
4. [权限控制系统](#4-权限控制系统)
5. [内置工具参考](#5-内置工具参考)
6. [Bash工具特殊设计](#6-bash工具特殊设计)
7. [文件操作工具设计哲学](#7-文件操作工具设计哲学)
8. [MCP系统集成](#8-mcp系统集成)
9. [实现参考代码](#9-实现参考代码)
10. [最佳实践总结](#10-最佳实践总结)

---

## 1. 架构概览

### 1.1 核心设计原则

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenCode 工具链核心原则                       │
├─────────────────────────────────────────────────────────────────┤
│ 1. 意图分离：LLM表达意图 → 系统处理执行细节                      │
│ 2. 安全第一：所有操作经过权限检查，用户有最终控制权               │
│ 3. 结构化交互：工具返回结构化数据，非原始文本                     │
│ 4. 可扩展性：插件系统支持自定义工具                              │
│ 5. 渐进式披露：实验性工具通过feature flag控制                     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Agent 层                                           │
│  - 对话管理                                                   │
│  - 任务规划                                                   │
│  - 工具选择决策（由LLM完成）                                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 工具管理层 (Tool Management)                        │
│  - 工具注册与发现 (ToolRegistry)                              │
│  - 参数验证 (Zod Schema)                                      │
│  - 权限检查 (Permission System)                               │
│  - 执行调度                                                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 执行层 (Execution)                                  │
│  - 具体工具实现 (Tool.define)                                 │
│  - 系统调用 (spawn, fs, etc.)                                 │
│  - 输出处理 (截断、格式化)                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 工具定义系统

### 2.1 核心抽象：Tool 命名空间

```typescript
// 核心概念：Tool 不是类，而是接口+工厂函数
export namespace Tool {
  // 工具信息接口
  export interface Info<T = unknown> {
    id: string;                              // 唯一标识
    init: (ctx: InitContext) => Promise<T>;  // 初始化函数
  }

  // 工具定义结构（由 Tool.define 创建）
  export interface Definition<TParams, TResult> {
    name: string;
    description: string;
    parameters: ZodSchema<TParams>;          // 参数验证schema
    execute: (params: TParams, context: Context) => Promise<TResult>;
  }

  // 执行上下文
  export interface Context {
    sessionID: string;
    messageID: string;
    abort: AbortSignal;
    ask: (request: PermissionRequest) => Promise<PermissionResponse>;
    // ... 其他上下文
  }
}

// 工厂函数：创建工具定义
export function define<TParams, TResult>(
  name: string,
  config: {
    description: string;
    parameters: ZodSchema<TParams>;
    execute: (params: TParams, ctx: Tool.Context) => Promise<TResult>;
  }
): Tool.Info;
```

### 2.2 工具注册中心

```typescript
export namespace ToolRegistry {
  // 获取所有可用工具（仅内置工具）
  export async function tools(model, agent): Promise<Tool.Definition[]> {
    return [
      InvalidTool,
      QuestionTool,
      BashTool,
      ReadTool,
      GlobTool,
      GrepTool,
      EditTool,
      WriteTool,
      TaskTool,
      WebFetchTool,
      TodoWriteTool,
      WebSearchTool,
      CodeSearchTool,
      SkillTool,
      ApplyPatchTool,
      // 实验性工具（条件启用）
      ...(experimental ? [LspTool, BatchTool, PlanExitTool, PlanEnterTool] : []),
      // 自定义工具（文件/插件）
      ...custom,
    ];
  }

  // 动态加载自定义工具
  // 扫描路径：~/.opencode/tool/*.{js,ts}
  export async function loadCustomTools(): Promise<Tool.Info[]>;
}

// MCP 工具在 Session 层单独加载
// packages/opencode/src/session/prompt.ts
async function buildAllTools() {
  const tools: Record<string, Tool> = {};

  // 1. 内置工具
  const builtinTools = await ToolRegistry.tools(model, agent);
  for (const [key, item] of Object.entries(builtinTools)) {
    tools[key] = wrapWithHooks(item);
  }

  // 2. MCP 工具（运行时从 MCP Server 发现）
  const mcpTools = await MCP.tools();
  for (const [key, item] of Object.entries(mcpTools)) {
    tools[key] = wrapWithMcpHooks(item);  // 添加统一权限检查
  }

  return tools;
}
```

### 2.3 工具定义示例

```typescript
// read.ts - 文件读取工具实现
import { Tool } from "./tool";
import z from "zod";
import DESCRIPTION from "./read.txt";  // 加载使用指南

export const ReadTool = Tool.define("read", {
  // 描述会展示给LLM
  description: DESCRIPTION,

  // Zod schema 定义参数结构和验证
  parameters: z.object({
    file_path: z.string()
      .describe("The absolute path to the file to read"),
    offset: z.number().optional()
      .describe("Line number to start reading from (1-indexed)"),
    limit: z.number().optional()
      .describe("Maximum number of lines to read"),
  }),

  // 执行逻辑
  async execute(params, ctx) {
    const { file_path, offset, limit } = params;

    // 1. 权限检查
    await ctx.ask({
      permission: "read",
      patterns: [file_path],
      metadata: { filepath: file_path }
    });

    // 2. 执行读取
    const content = await fs.readFile(file_path, "utf-8");
    const lines = content.split("\n");

    // 3. 处理行范围
    const start = offset ? offset - 1 : 0;
    const end = limit ? start + limit : lines.length;
    const result = lines.slice(start, end).join("\n");

    // 4. 返回结构化结果
    return {
      content: result,
      metadata: {
        totalLines: lines.length,
        readLines: end - start,
        truncated: end < lines.length
      }
    };
  }
});
```

---

## 3. 完整调用链

### 3.1 调用流程图

```
用户输入: "帮我读取 config.ts 的内容"
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Agent 构建 Prompt                                     │
│                                                             │
│ System: 你是OpenCode助手，可以使用以下工具：                  │
│ - read: 读取文件，参数：file_path, offset, limit            │
│ - write: 写入文件，参数：file_path, content                 │
│ ...                                                         │
│                                                             │
│ [read.txt 内容]                                             │
│ "Read a file from the local filesystem..."                  │
│                                                             │
│ User: 帮我读取 config.ts 的内容                             │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: LLM 决策                                              │
│                                                             │
│ LLM分析：                                                    │
│ - 意图：读取文件                                             │
│ - 目标文件：config.ts                                        │
│ - 匹配工具：read                                             │
│ - 参数推导：file_path = "config.ts"                          │
│                                                             │
│ 输出工具调用：                                                │
│ {                                                           │
│   "tool": "read",                                           │
│   "params": {                                               │
│     "file_path": "config.ts"                                │
│   }                                                         │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: 参数验证                                              │
│                                                             │
│ Zod schema 验证：                                            │
│ - file_path: string ✅                                       │
│ - offset: number (optional) ✅                              │
│ - limit: number (optional) ✅                               │
│                                                             │
│ 验证通过 → 继续执行                                          │
│ 验证失败 → 返回错误给LLM                                      │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: 权限检查                                              │
│                                                             │
│ 检查规则：                                                   │
│ - read 操作是否在允许范围内？                                │
│ - config.ts 是否在项目目录内？                               │
│                                                             │
│ 决策：                                                       │
│ - 规则允许 → 直接执行                                        │
│ - 需要确认 → 提示用户："允许读取 config.ts 吗？"              │
│ - 规则拒绝 → 返回权限错误                                     │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: 工具执行                                              │
│                                                             │
│ 调用 ReadTool.execute():                                     │
│ 1. fs.readFile("config.ts")                                 │
│ 2. 处理内容（截断、格式化）                                   │
│ 3. 返回结果                                                  │
│                                                             │
│ 结果：                                                       │
│ {                                                           │
│   "content": "export const config = {...}",                 │
│   "metadata": { "truncated": false }                        │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: 结果返回给LLM                                         │
│                                                             │
│ LLM 生成自然语言回复：                                        │
│ "config.ts 的内容如下：
 export const config = {...}"                                │
└─────────────────────────────────────────────────────────────┘
           ↓
用户看到回复
```

### 3.2 关键数据结构

```typescript
// 工具调用请求（LLM 输出）
interface ToolCall {
  tool: string;           // 工具ID
  params: Record<string, any>;  // 参数
}

// 权限请求
interface PermissionRequest {
  permission: string;     // 权限类型
  patterns: string[];     // 影响的资源模式
  metadata: Record<string, any>;  // 附加信息
}

// 工具执行结果
interface ToolResult {
  output: any;            // 主要输出内容
  metadata?: Record<string, any>;  // 元数据（如truncated标记）
  title?: string;         // 结果标题
}
```

---

## 4. 权限控制系统

### 4.1 权限检查流程

```typescript
// permission/next.ts - 权限检查核心
async function checkPermission(
  toolCall: ToolCall,
  context: Tool.Context
): Promise<PermissionDecision> {
  const { tool, params } = toolCall;

  // 1. 查找匹配的规则
  const rules = await getPermissionRules();
  const matchingRule = findMatchingRule(rules, tool, params);

  if (matchingRule) {
    switch (matchingRule.action) {
      case "allow":
        return { allowed: true };  // 直接放行

      case "deny":
        return { allowed: false, reason: matchingRule.reason };

      case "ask":
        // 继续询问用户
        break;
    }
  }

  // 2. 默认行为：询问用户
  const userDecision = await promptUser({
    tool: tool.name,
    description: generateHumanReadableDescription(tool, params),
    params
  });

  return { allowed: userDecision === "allow" };
}
```

### 4.2 权限配置格式

```json
// opencode.json
{
  "permission": {
    "rules": [
      {
        "tool": "read",
        "pattern": "**/*.ts",
        "action": "allow"
      },
      {
        "tool": "write",
        "pattern": "**/*.env",
        "action": "ask"
      },
      {
        "tool": "bash",
        "pattern": "rm -rf /*",
        "action": "deny",
        "reason": "Dangerous command"
      },
      {
        "tool": "bash",
        "pattern": "git status",
        "action": "allow"
      }
    ]
  }
}
```

### 4.3 权限检查点

每个工具在 `execute` 函数内部调用 `ctx.ask()` 进行权限检查：

```typescript
async execute(params, ctx) {
  // 在执行前检查权限
  await ctx.ask({
    permission: "read",
    patterns: [params.file_path],
    metadata: { filepath: params.file_path }
  });

  // 权限通过后执行
  return fs.readFile(params.file_path);
}
```

---

## 5. 内置工具参考

### 5.1 标准工具（始终可用）

| 工具 | 用途 | 核心参数 |
|------|------|---------|
| `read` | 读取文件 | `file_path`, `offset`, `limit` |
| `write` | 创建/覆盖文件 | `file_path`, `content` |
| `edit` | 编辑文件（字符串替换） | `file_path`, `old_string`, `new_string` |
| `glob` | 文件模式匹配 | `pattern` |
| `grep` | 内容搜索 | `pattern`, `path`, `output_mode` |
| `list` | 列出目录 | `path`, `ignore` |
| `bash` | 执行shell命令 | `command`, `description`, `timeout` |
| `task` | 创建子Agent | `description`, `prompt`, `subagent_type` |
| `todowrite` | 待办管理 | `todos[]` |
| `webfetch` | 获取网页 | `url`, `prompt` |
| `websearch` | 网络搜索 | `query` |
| `codesearch` | 代码搜索 | `query`, `repos` |
| `skill` | 加载技能 | `skill` |
| `question` | 交互提问 | `questions[]` |

### 5.2 实验性工具

| 工具 | 用途 | 启用条件 |
|------|------|---------|
| `lsp` | LSP操作 | `OPENCODE_EXPERIMENTAL_LSP_TOOL` |
| `batch` | 批量执行 | `config.experimental.batch_tool: true` |
| `plan_enter/exit` | 计划模式 | `OPENCODE_EXPERIMENTAL_PLAN_MODE` |

---

## 6. Bash工具特殊设计

### 6.1 双文件结构

```
bash.ts   - 代码实现（机器执行）
bash.txt  - 使用指南（LLM阅读）
```

**为什么要分离？**

```typescript
// bash.ts - 实际执行逻辑
export const BashTool = Tool.define("bash", {
  description: DESCRIPTION,  // ← 这里加载 bash.txt
  parameters: z.object({
    command: z.string(),
    description: z.string(),
    timeout: z.number().optional()
  }),
  async execute(params, ctx) {
    // 权限检查
    await ctx.ask({ permission: "bash", ... });

    // 使用 spawn 执行
    const child = spawn(shell, ["-c", params.command], { ... });
    // ...
  }
});
```

```text
# bash.txt - 给LLM的指导
## Bash Tool Guidelines

### Allowed Operations
- git, npm, docker commands
- Build, test, deploy commands
- Run multiple independent commands in parallel

### Prohibited Operations
- File operations (use specialized tools): find, grep, cat, head, tail, sed, awk, echo
- cd commands (use workdir parameter)
- Destructive git operations
- File creation via redirection (>, >>)
```

### 6.2 安全限制

| 禁止项 | 原因 | 替代方案 |
|--------|------|---------|
| `cat`, `grep`, `find` | 应使用结构化工具 | `read`, `glob`, `grep` 工具 |
| `cd` | 难以跟踪工作目录 | `workdir` 参数 |
| `>`, `>>` 重定向 | 绕过权限检查 | `write` 工具 |
| `rm -rf /` | 危险操作 | 权限规则拒绝 |
| 管道 `|` | 难以分析 | 分步执行或使用专用工具 |

---

## 7. 文件操作工具设计哲学

### 7.1 为什么 `read` 优于 `cat`

| 对比维度 | `read` 工具 | `cat` 命令 |
|---------|------------|-----------|
| **输出控制** | 自动截断 + 元数据 | 无限制输出 |
| **结构化** | `{content, metadata}` | 原始文本 |
| **权限** | 精确到文件级别 | shell级别 |
| **LLM理解** | 明确的成功/失败 | 需解析错误信息 |
| **大文件** | ✅ 安全处理 | ❌ 可能溢出 |

### 7.2 核心设计原则

**原则1：意图优先于实现**

```
❌ LLM: "使用 cat -n file.ts | head -50"
   → 需要理解cat、管道、head命令

✅ LLM: "read file_path=file.ts limit=50"
   → 直接表达意图：读文件的前50行
```

**原则2：结构化胜于文本**

```typescript
// 工具返回结构化数据
{
  content: "...",
  metadata: {
    truncated: true,      // LLM知道内容不完整
    totalLines: 1000,     // 总行数
    readLines: 200        // 已读行数
  }
}

// 而非纯文本，需要LLM解析：
"...\n[Output truncated, 800 more lines]"
```

**原则3：权限最小化**

```typescript
// 每个操作单独检查权限
read file.ts    → 检查file.ts的读取权限
write config.ts → 检查config.ts的写入权限

// 而非 blanket permission:
bash "cat file.ts && echo x > config.ts"  → 难以分析风险
```

---

## 8. 实现参考代码

### 8.1 最小工具实现模板

```typescript
// mytool.ts
import { Tool } from "./tool";
import z from "zod";

export const MyTool = Tool.define("my_tool", {
  description: "Description for LLM to understand when to use this tool",

  parameters: z.object({
    param1: z.string()
      .describe("Description of param1 for LLM"),
    param2: z.number().optional()
      .describe("Optional parameter description")
  }),

  async execute(params, ctx) {
    const { param1, param2 } = params;

    // 1. 权限检查（如需要）
    await ctx.ask({
      permission: "my_permission",
      patterns: [param1],
      metadata: { target: param1 }
    });

    // 2. 执行操作
    const result = await doSomething(param1, param2);

    // 3. 返回结构化结果
    return {
      output: result,
      metadata: {
        processedAt: new Date().toISOString()
      }
    };
  }
});
```

### 8.2 工具注册

```typescript
// registry.ts
import { MyTool } from "./mytool";

export async function getAllTools() {
  return [
    // 内置工具
    ReadTool,
    WriteTool,
    BashTool,
    // ...

    // 自定义工具
    MyTool,

    // 实验性工具（条件加载）
    ...(isExperimentalEnabled() ? [ExperimentalTool] : [])
  ];
}
```

### 8.3 权限系统实现

```typescript
// permission.ts
interface PermissionRule {
  tool: string;
  pattern: string;
  action: "allow" | "deny" | "ask";
  reason?: string;
}

class PermissionManager {
  private rules: PermissionRule[];

  async check(
    tool: string,
    params: Record<string, any>
  ): Promise<{ allowed: boolean; reason?: string }> {
    // 1. 查找匹配规则
    const rule = this.rules.find(r =>
      r.tool === tool &&
      matchesPattern(params, r.pattern)
    );

    if (rule) {
      if (rule.action === "allow") return { allowed: true };
      if (rule.action === "deny") return { allowed: false, reason: rule.reason };
    }

    // 2. 默认询问用户
    return this.promptUser(tool, params);
  }

  private async promptUser(tool: string, params: any) {
    // UI提示用户确认
    // 返回用户选择
  }
}
```

---

## 9. 最佳实践总结

### 9.1 工具设计原则

1. **单一职责**：每个工具只做一件事，做好一件事
   - ✅ `read`: 读取文件
   - ✅ `glob`: 查找文件
   - ❌ `fileOps`: 同时处理读、写、搜索

2. **参数自描述**：使用 Zod schema + describe
   ```typescript
   z.string().describe("Absolute path to the file")
   ```

3. **明智的默认值**：减少LLM决策负担
   ```typescript
   timeout: z.number().default(120000)  // 2分钟默认超时
   ```

4. **错误处理**：返回结构化错误，非抛出异常
   ```typescript
   return {
     success: false,
     error: "File not found",
     suggestion: "Check the path or use glob to find the file"
   }
   ```

### 9.2 安全最佳实践

1. **所有修改操作需要权限检查**
2. **路径验证**：确保在允许的目录内
3. **输出截断**：防止LLM上下文溢出
4. **命令白名单**：限制bash可执行的命令类型

### 9.3 LLM体验优化

1. **提供使用指南**（如 bash.txt）
2. **人类可读的描述**：生成清晰的权限提示
3. **结果摘要**：帮助LLM快速理解输出
4. **渐进式披露**：复杂功能通过skill加载

---

## 8. MCP系统集成

### 8.1 MCP 是什么

MCP（Model Context Protocol）是 Anthropic 提出的开放协议，用于标准化 AI 模型与外部工具/服务的通信。OpenCode 将 MCP 作为**内置工具的扩展机制**，允许连接外部 MCP Server 来扩展工具生态。

**核心概念**：
- **MCP Server**：外部工具服务（可以是本地进程或远程服务）
- **MCP Client**：OpenCode 中与 MCP Server 通信的客户端
- **Tools**：MCP Server 提供的可调用工具
- **Prompts**：MCP Server 提供的预定义提示模板
- **Resources**：MCP Server 提供的可读资源

### 8.2 MCP 与内置工具的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenCode 工具系统架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────┐      ┌─────────────────────┐         │
│   │   内置 Tools 系统    │      │    MCP 系统         │         │
│   │   (Built-in)        │      │   (External)        │         │
│   ├─────────────────────┤      ├─────────────────────┤         │
│   │ • BashTool          │      │ • Remote MCP Server │         │
│   │ • ReadTool          │      │ • Local MCP Server  │         │
│   │ • WriteTool         │      │                     │         │
│   │ • EditTool          │      │ 通过 stdio/http/sse │         │
│   │ • GlobTool          │      │ 连接到外部服务      │         │
│   │ • ...               │      │                     │         │
│   │                     │      │ 提供:               │         │
│   │ 直接实现 execute()  │      │ - Tools             │         │
│   │                     │      │ - Prompts           │         │
│   │                     │      │ - Resources         │         │
│   └─────────┬───────────┘      └─────────┬───────────┘         │
│             │                            │                      │
│             └────────────┬───────────────┘                      │
│                          │                                      │
│                          ▼                                      │
│              ┌─────────────────────┐                           │
│              │   统一集成层         │                           │
│              │   (Session/Prompt)  │                           │
│              ├─────────────────────┤                           │
│              │ • 所有工具统一暴露   │                           │
│              │   给 LLM            │                           │
│              │ • 统一权限检查       │                           │
│              │ • 统一执行钩子       │                           │
│              │ • 统一输出格式化     │                           │
│              └─────────────────────┘                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**关键设计原则**：
- **无差别化**：对 LLM 来说，MCP 工具和内置工具完全一致
- **统一包装**：MCP 工具通过 `dynamicTool` 包装，获得相同的权限控制和插件支持
- **运行时发现**：MCP 工具在运行时动态发现，而非编译时确定

### 8.3 MCP 工具转换流程

```typescript
// packages/opencode/src/mcp/index.ts
async function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Promise<Tool> {
  const inputSchema = mcpTool.inputSchema

  // 将 MCP 的 JSON Schema 转换为 AI SDK 格式
  const schema: JSONSchema7 = {
    ...(inputSchema as JSONSchema7),
    type: "object",
    properties: (inputSchema.properties ?? {}) as JSONSchema7["properties"],
    additionalProperties: false,
  }

  // 使用 dynamicTool 包装，使其看起来像内置 Tool
  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),
    execute: async (args: unknown) => {
      // 调用 MCP 服务器
      return client.callTool(
        {
          name: mcpTool.name,
          arguments: (args || {}) as Record<string, unknown>,
        },
        CallToolResultSchema,
        { resetTimeoutOnProgress: true, timeout }
      )
    },
  })
}
```

### 8.4 MCP 配置格式

```json
// opencode.json
{
  "mcp": {
    // 本地 MCP Server（通过 stdio 通信）
    "my-local-server": {
      "type": "local",
      "command": ["node", "./mcp-server.js"],
      "environment": {
        "API_KEY": "xxx"
      }
    },

    // 远程 MCP Server（通过 HTTP/SSE 通信）
    "my-remote-server": {
      "type": "remote",
      "url": "https://mcp.example.com/v1",
      "oauth": true,  // 启用 OAuth 认证
      "headers": {
        "X-Custom": "header"
      },
      "timeout": 30000
    }
  }
}
```

### 8.5 MCP 与内置工具对比

| 特性 | 内置 Tools | MCP Tools |
|------|-----------|-----------|
| **实现位置** | 本地代码 (`src/tool/*.ts`) | 外部进程 (本地/远程) |
| **通信方式** | 直接函数调用 | stdio / HTTP / SSE |
| **协议** | OpenCode 内部协议 | Model Context Protocol |
| **权限检查** | 每个工具内部调用 `ctx.ask()` | 统一在 Session 层包装 |
| **输出格式** | 结构化对象 | MCP `CallToolResult` |
| **发现方式** | 编译时注册 | 运行时通过 MCP 协议发现 |
| **配置方式** | 代码硬编码 | `opencode.json` 配置 |
| **性能** | 高（无 IPC 开销） | 较低（有网络/进程通信） |
| **扩展性** | 需修改代码 | 动态添加配置即可 |

### 8.6 MCP 执行流程

```
用户配置 MCP Server → OpenCode 启动时连接 → 发现可用工具
                                                    ↓
用户输入 → LLM 分析 → 调用 MCP 工具
                        ↓
                ┌───────────────┐
                │ Session 包装层 │ ← 统一权限检查
                └───────┬───────┘
                        ↓
                ┌───────────────┐
                │  MCP Client   │ ← 协议转换
                └───────┬───────┘
                        ↓
                ┌───────────────┐
                │  MCP Server   │ ← 外部执行
                │ (stdio/http)  │
                └───────┬───────┘
                        ↓
                返回结果 → 格式化 → 返回给 LLM
```

### 8.7 Session 层统一集成

```typescript
// packages/opencode/src/session/prompt.ts
async function buildTools() {
  const tools: Record<string, Tool> = {}

  // 1. 添加内置工具
  for (const [key, item] of Object.entries(await ToolRegistry.tools())) {
    tools[key] = wrapWithHooks(item)  // 包装插件钩子和权限检查
  }

  // 2. 添加 MCP 工具（关键集成点）
  for (const [key, item] of Object.entries(await MCP.tools())) {
    // MCP 工具名称格式：{serverName}_{toolName}
    const transformed = ProviderTransform.schema(input.model, asSchema(item.inputSchema).jsonSchema)
    item.inputSchema = jsonSchema(transformed)

    // 包装 execute 添加统一权限检查和插件钩子
    item.execute = async (args, opts) => {
      const ctx = context(args, opts)

      // 统一权限检查
      await ctx.ask({
        permission: key,
        metadata: {},
        patterns: ["*"],
        always: ["*"],
      })

      // 执行前钩子
      await Plugin.trigger("tool.execute.before", { tool: key, ... }, { args })

      // 执行 MCP 工具
      const result = await execute(args, opts)

      // 执行后钩子
      await Plugin.trigger("tool.execute.after", { tool: key, ... }, result)

      // 格式化输出（MCP CallToolResult → 内部格式）
      return formatMcpResult(result)
    }

    tools[key] = item
  }

  return tools
}
```

### 8.8 MCP 状态管理

```typescript
// MCP 连接状态
export const Status = z.discriminatedUnion("status", [
  z.object({ status: z.literal("connected") }),
  z.object({ status: z.literal("disabled") }),
  z.object({ status: z.literal("failed"), error: z.string() }),
  z.object({ status: z.literal("needs_auth") }),
  z.object({ status: z.literal("needs_client_registration"), error: z.string() }),
])

// 获取所有 MCP Server 状态
const status = await MCP.status()
// {
//   "my-local-server": { status: "connected" },
//   "my-remote-server": { status: "needs_auth" }
// }
```

### 8.9 MCP 管理 API

```typescript
// 连接 MCP Server
await MCP.connect("server-name")

// 断开连接
await MCP.disconnect("server-name")

// 添加新的 MCP Server
await MCP.add("new-server", {
  type: "local",
  command: ["node", "server.js"]
})

// OAuth 认证流程
await MCP.authenticate("server-name")  // 打开浏览器完成 OAuth
await MCP.finishAuth("server-name", authorizationCode)
await MCP.removeAuth("server-name")
```

---

## 附录：核心文件结构

```
packages/opencode/src/tool/
├── tool.ts              # Tool 命名空间定义
├── registry.ts          # 工具注册中心
│
├── bash.ts              # Bash工具实现
├── bash.txt             # Bash使用指南
│
├── read.ts              # 文件读取
├── write.ts             # 文件写入
├── edit.ts              # 文件编辑
├── glob.ts              # 文件匹配
├── grep.ts              # 内容搜索
├── ls.ts                # 目录列表
│
├── task.ts              # 子任务创建
├── todo.ts              # 待办管理
│
├── webfetch.ts          # 网页获取
├── websearch.ts         # 网络搜索
├── codesearch.ts        # 代码搜索
│
├── skill.ts             # 技能加载
├── question.ts          # 交互提问
├── apply_patch.ts       # 补丁应用
├── batch.ts             # 批量执行（实验性）
├── lsp.ts               # LSP操作（实验性）
├── plan.ts              # 计划模式（实验性）
│
├── truncation.ts        # 输出截断处理
└── invalid.ts           # 无效工具处理

packages/opencode/src/permission/
├── next.ts              # 权限检查主逻辑
├── arity.ts             # 命令参数数量定义
└── types.ts             # 类型定义

packages/opencode/src/mcp/
├── index.ts             # MCP 核心实现（Client管理、工具转换）
├── auth.ts              # MCP 认证管理
├── oauth-provider.ts    # OAuth Provider 实现
└── oauth-callback.ts    # OAuth 回调处理
```

---

*文档版本：1.0*
*基于：OpenCode dev branch (2024)*
*用途：自定义Code Agent开发参考*
