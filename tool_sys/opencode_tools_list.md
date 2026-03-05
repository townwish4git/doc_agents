# OpenCode 内置工具列表

> 本文档列出了 OpenCode 中所有可用的内置工具，包括标准工具和实验性工具。

---

## 📋 工具总览

OpenCode 提供 **15+ 个内置工具**，分为以下几类：

| 类别 | 工具数量 | 说明 |
|------|---------|------|
| **文件操作** | 5 | 读、写、编辑、查找、搜索文件 |
| **系统操作** | 1 | 执行 Bash 命令 |
| **网络操作** | 2 | 网页获取、网络搜索、代码搜索 |
| **任务管理** | 2 | 子任务创建、待办事项管理 |
| **辅助工具** | 4 | 交互提问、技能加载、补丁应用、无效调用处理 |
| **实验性工具** | 3 | 批量执行、LSP 支持、计划模式 |

---

## 🔧 标准工具（始终可用）

### 1. bash - Bash 命令执行

**用途**：在项目目录中执行 shell 命令。

**典型场景**：
- 运行 Git 命令（`git status`, `git diff`, `git log`）
- 执行构建脚本（`npm run build`, `make`, `cargo build`）
- 运行测试（`pytest`, `jest`, `go test`）
- 使用包管理器（`npm install`, `pip install`）

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `command` | string | 是 | 要执行的 bash 命令 |
| `description` | string | 是 | 命令的简洁描述（5-10 字） |
| `timeout` | number | 否 | 超时时间（毫秒），默认 120000 |
| `workdir` | string | 否 | 工作目录（默认项目根目录） |

**示例**：
```json
{
  "tool": "bash",
  "params": {
    "command": "git status",
    "description": "Check git status"
  }
}
```

**注意事项**：
- 禁止使用 `find`, `grep`, `cat`, `sed`, `awk`, `echo`（应使用专用工具）
- 禁止使用 `cd`（使用 `workdir` 参数代替）
- 禁止文件重定向操作符（`>`, `>>`, `<`）

---

### 2. read - 读取文件

**用途**：读取文件内容，支持指定行范围。

**典型场景**：
- 查看源代码文件
- 读取配置文件
- 查看日志文件的特定部分
- 阅读文档

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file_path` | string | 是 | 文件的绝对路径 |
| `offset` | number | 否 | 起始行号（从 1 开始） |
| `limit` | number | 否 | 读取的最大行数 |

**示例**：
```json
{
  "tool": "read",
  "params": {
    "file_path": "/path/to/file.ts",
    "offset": 1,
    "limit": 50
  }
}
```

---

### 3. write - 创建/覆盖文件

**用途**：创建新文件或完全覆盖现有文件。

**典型场景**：
- 创建新源代码文件
- 生成配置文件
- 创建文档
- 写入日志或报告

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file_path` | string | 是 | 文件的绝对路径 |
| `content` | string | 是 | 文件内容 |

**示例**：
```json
{
  "tool": "write",
  "params": {
    "file_path": "/path/to/newfile.ts",
    "content": "export const hello = 'world';"
  }
}
```

---

### 4. edit - 编辑文件

**用途**：通过字符串替换方式修改文件内容。

**典型场景**：
- 修改函数实现
- 更新变量名
- 修复拼写错误
- 添加或删除代码行

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file_path` | string | 是 | 文件的绝对路径 |
| `old_string` | string | 是 | 要被替换的文本 |
| `new_string` | string | 是 | 新的文本 |
| `replace_all` | boolean | 否 | 是否替换所有匹配项（默认 false） |

**示例**：
```json
{
  "tool": "edit",
  "params": {
    "file_path": "/path/to/file.ts",
    "old_string": "const x = 1;",
    "new_string": "const x = 2;"
  }
}
```

---

### 5. glob - 文件模式匹配

**用途**：根据 glob 模式查找文件。

**典型场景**：
- 查找所有 TypeScript 文件（`**/*.ts`）
- 查找测试文件（`**/*.test.ts`）
- 查找配置文件（`**/config.*`）
- 统计某类文件数量

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `pattern` | string | 是 | Glob 模式（如 `**/*.ts`） |
| `path` | string | 否 | 搜索的目录路径 |

**示例**：
```json
{
  "tool": "glob",
  "params": {
    "pattern": "**/*.ts"
  }
}
```

---

### 6. grep - 内容搜索

**用途**：使用正则表达式搜索文件内容。

**典型场景**：
- 查找函数定义
- 搜索特定文本模式
- 查找 TODO 注释
- 代码重构前的引用查找

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `pattern` | string | 是 | 正则表达式模式 |
| `path` | string | 否 | 搜索的目录或文件 |
| `glob` | string | 否 | 文件类型过滤（如 `*.ts`） |
| `output_mode` | string | 否 | 输出模式：`content`/`files_with_matches` |

**示例**：
```json
{
  "tool": "grep",
  "params": {
    "pattern": "function\s+\w+",
    "glob": "*.ts",
    "output_mode": "content"
  }
}
```

---

### 7. list - 列出目录内容

**用途**：列出目录结构和文件列表。

**典型场景**：
- 查看项目结构
- 浏览目录内容
- 了解文件组织方式

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `path` | string | 否 | 目录的绝对路径（默认当前目录） |
| `ignore` | string[] | 否 | 忽略的 glob 模式列表 |

**示例**：
```json
{
  "tool": "list",
  "params": {
    "path": "/path/to/directory"
  }
}
```

**注意事项**：
- 自动忽略常见目录（`node_modules/`, `.git/`, `dist/` 等）
- 最多返回 100 个文件

---

### 8. task - 创建子任务

**用途**：创建专门的子 Agent 来处理复杂任务。

**典型场景**：
- 并行处理多个独立任务
- 委派专门领域的工作（如测试、文档、研究）
- 处理耗时任务而不阻塞主会话

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `description` | string | 是 | 任务的简短描述 |
| `prompt` | string | 是 | 给子 Agent 的详细指令 |
| `subagent_type` | string | 是 | 子 Agent 类型 |

**子 Agent 类型**：
| 类型 | 用途 |
|------|------|
| `bash` | 专门执行命令行任务 |
| `general-purpose` | 通用目的子 Agent |
| `explore` | 快速探索代码库 |
| `plan` | 架构规划 |
| `statusline` | 状态栏配置 |

**示例**：
```json
{
  "tool": "task",
  "params": {
    "description": "Find auth code",
    "prompt": "Search for authentication-related code in the codebase",
    "subagent_type": "explore"
  }
}
```

---

### 9. todowrite - 待办事项管理

**用途**：创建和管理任务列表，跟踪进度。

**典型场景**：
- 规划复杂功能的实现步骤
- 跟踪多个文件的修改进度
- 记录需要处理的事项

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `todos` | array | 是 | 待办事项列表 |
| `todos[].content` | string | 是 | 任务内容 |
| `todos[].status` | string | 是 | 状态：`pending`/`in_progress`/`completed` |
| `todos[].activeForm` | string | 是 | 进行中的描述形式 |

**示例**：
```json
{
  "tool": "todowrite",
  "params": {
    "todos": [
      {
        "content": "Update README",
        "status": "pending",
        "activeForm": "Updating README"
      }
    ]
  }
}
```

---

### 10. webfetch - 网页获取

**用途**：获取网页内容并转换为 Markdown。

**典型场景**：
- 获取技术文档
- 查看 API 参考
- 阅读博客文章
- 获取最新的库文档

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `url` | string | 是 | 要获取的 URL |
| `prompt` | string | 否 | 对内容的处理提示 |

**示例**：
```json
{
  "tool": "webfetch",
  "params": {
    "url": "https://example.com/docs",
    "prompt": "Extract the API endpoints"
  }
}
```

---

### 11. websearch - 网络搜索

**用途**：通过网络搜索获取最新信息。

**启用条件**：
- 使用 OpenCode 官方模型提供商，或
- 设置 `OPENCODE_ENABLE_EXA` 环境变量

**典型场景**：
- 查找最新的技术文档
- 搜索 API 使用示例
- 获取最新的软件版本信息
- 研究技术方案

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | 是 | 搜索查询 |

**示例**：
```json
{
  "tool": "websearch",
  "params": {
    "query": "React 19 new features 2025"
  }
}
```

---

### 12. codesearch - 代码搜索

**用途**：搜索代码和 API 文档。

**启用条件**：
- 使用 OpenCode 官方模型提供商，或
- 设置 `OPENCODE_ENABLE_EXA` 环境变量

**典型场景**：
- 查找开源库的用法示例
- 搜索特定函数的调用方式
- 学习 API 的最佳实践

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | 是 | 搜索查询 |
| `repos` | string[] | 否 | 限定搜索的仓库 |

**示例**：
```json
{
  "tool": "codesearch",
  "params": {
    "query": "useEffect cleanup function example",
    "repos": ["facebook/react"]
  }
}
```

---

### 13. skill - 技能加载

**用途**：加载专门的技能指令，扩展 Agent 能力。

**典型场景**：
- 加载特定的代码审查技能
- 加载安全审计技能
- 加载性能优化技能
- 加载特定框架的专业知识

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `skill` | string | 是 | 技能名称或路径 |

**示例**：
```json
{
  "tool": "skill",
  "params": {
    "skill": "security-audit"
  }
}
```

---

### 14. apply_patch - 应用补丁

**用途**：应用 unified diff 格式的补丁。

**特殊说明**：
- 仅在使用 GPT 模型时可用（除了 GPT-4 和 OSS 模型）
- 替代 `edit` 和 `write` 工具

**典型场景**：
- 应用大段代码修改
- 批量文件更新
- 复杂的重构操作

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `patch` | string | 是 | Unified diff 格式的补丁内容 |

**示例**：
```json
{
  "tool": "apply_patch",
  "params": {
    "patch": "--- a/file.ts\n+++ b/file.ts\n@@ -1,3 +1,3 @@\n-const x = 1;\n+const x = 2;"
  }
}
```

---

### 15. question - 交互提问

**用途**：向用户提出交互式问题，获取澄清或确认。

**启用条件**：仅在 `app`, `cli`, `desktop` 客户端或使用 `OPENCODE_ENABLE_QUESTION_TOOL` 时可用。

**典型场景**：
- 获取实现方案的选择
- 询问文件路径
- 确认重要的操作
- 收集缺失的信息

**主要参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `questions` | array | 是 | 问题列表 |
| `questions[].question` | string | 是 | 问题内容 |
| `questions[].options` | array | 是 | 选项列表 |
| `questions[].header` | string | 是 | 问题标签 |

**示例**：
```json
{
  "tool": "question",
  "params": {
    "questions": [
      {
        "question": "Which testing framework?",
        "options": ["Jest", "Vitest", "Mocha"],
        "header": "Test framework"
      }
    ]
  }
}
```

---

### 16. invalid - 无效工具处理

**用途**：处理无效或错误的工具调用。

**说明**：这是一个内部工具，用于优雅地处理工具调用错误，通常不需要直接使用。

---

## 🧪 实验性工具

以下工具处于实验阶段，需要特定条件启用。

### 17. lsp - LSP 操作

**用途**：与语言服务器协议（LSP）交互。

**启用条件**：设置 `OPENCODE_EXPERIMENTAL_LSP_TOOL` 环境变量。

**典型场景**：
- 获取代码定义
- 查找引用
- 类型信息查询
- 代码重构

---

### 18. batch - 批量执行

**用途**：并行执行多个工具调用。

**启用条件**：在配置中设置 `experimental.batch_tool: true`。

**典型场景**：
- 同时读取多个文件
- 并行执行独立的命令
- 批量搜索操作

---

### 19. plan_enter / plan_exit - 计划模式

**用途**：进入或退出计划模式。

**启用条件**：
- 设置 `OPENCODE_EXPERIMENTAL_PLAN_MODE` 环境变量
- 仅在 CLI 客户端可用

**典型场景**：
- 复杂功能的架构规划
- 需要多步骤实现的重大变更
- 需要用户确认的设计方案

---

## 📊 工具选择速查表

| 想做... | 使用工具 |
|---------|---------|
| 执行命令 | `bash` |
| 读文件 | `read` |
| 创建文件 | `write` |
| 修改文件 | `edit` 或 `apply_patch` |
| 找文件 | `glob` |
| 搜索内容 | `grep` |
| 看目录结构 | `list` |
| 创建子任务 | `task` |
| 管理待办 | `todowrite` |
| 获取网页 | `webfetch` |
| 网络搜索 | `websearch` |
| 搜索代码 | `codesearch` |
| 加载技能 | `skill` |
| 问用户问题 | `question` |

---

## ⚠️ 权限说明

大多数工具在执行前会触发权限检查：

- **自动允许**：只读操作（如 `read`, `glob`, `grep`）
- **需要确认**：修改操作（如 `write`, `edit`, `bash`）
- **可配置**：通过 `opencode.json` 设置权限规则

---

## 🔌 自定义工具

除了内置工具，OpenCode 还支持自定义工具：

1. **通过文件**：在 `~/.opencode/tool/` 或 `~/.opencode/tools/` 目录放置 `.js` 或 `.ts` 文件
2. **通过插件**：开发 npm 插件并通过配置加载

自定义工具会被自动注册并可在 Agent 中使用。

---

*文档生成时间：2024年*
*适用版本：OpenCode dev branch*
