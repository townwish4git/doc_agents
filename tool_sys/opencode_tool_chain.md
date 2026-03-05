# OpenCode 工具调用链详解

> 本文档用简单的方式解释：当你在 OpenCode 中输入一句话时，系统是如何理解你的意图并执行相应操作的。

---

## 📖 阅读指南

- **给奶奶看的版本**：只看 🏠 **生活化类比** 部分
- **给教授看的版本**：重点看 🔬 **技术实现** 部分
- **完整理解**：建议按顺序阅读，从宏观到微观

---

## 第一部分：整体流程（三步走）

### 🏠 生活化类比：餐厅点餐

想象你去一家高级餐厅吃饭：

```
你（顾客）："我想吃一道辣的鸡肉菜"
        ↓
服务员（听懂需求）：→ "顾客要辣子鸡丁，口味偏辣"
        ↓
厨房（执行）：→ 厨师切鸡肉、炒辣椒、装盘
        ↓
你（顾客）：→ 吃到美味的辣子鸡丁
```

### 🔬 技术实现：三段式架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      第一阶段：理解意图                           │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │  用户输入    │ → │   LLM 分析    │ → │   生成工具调用     │   │
│  │  "查git状态" │    │  理解意图      │    │  tool: bash      │   │
│  └─────────────┘    └──────────────┘    │  params: {...}   │   │
│                                          └──────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      第二阶段：权限检查                           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  解析工具调用  │ → │  检查权限规则   │ → │   请求用户确认    │  │
│  │  参数校验      │    │  匹配规则      │    │  允许/拒绝       │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      第三阶段：实际执行                           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  调用工具函数  │ → │  执行具体操作  │ → │   返回执行结果    │  │
│  │  Tool.handle │    │  spawn bash  │    │  stdout/stderr   │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 第二部分：详细拆解每个阶段

### 第一阶段：理解意图（The Brain）

#### 🏠 类比：翻译官

想象你是一位不会中文的外国厨师，面前有一位中国顾客和一位翻译官：

```
顾客说："我想看看代码改了哪些东西"
        ↓
翻译官思考：
  - "代码" → git 相关
  - "改了哪些" → status 命令
  - 结论：需要执行 "git status"
        ↓
厨师收到指令："执行 git status 命令"
```

#### 🔬 技术细节

**1. 系统提示构建（System Prompt Construction）**

```typescript
// packages/opencode/src/agent/agent.ts
const systemPrompt = `
你是 OpenCode 助手，可以帮助用户完成编程任务。

你可以使用以下工具：

## bash
执行 shell 命令。
参数：
  - command: 要执行的命令字符串
  - timeout: 超时时间（可选）
  - description: 命令描述

## read
读取文件内容。
参数：
  - file_path: 文件路径
  - offset: 起始行号（可选）
  - limit: 读取行数（可选）

## glob
查找匹配模式的文件。
参数：
  - pattern: glob 模式，如 "**/*.ts"

## edit
编辑文件内容。
参数：
  - file_path: 文件路径
  - old_string: 要被替换的文本
  - new_string: 新文本

[... 其他工具]

## 使用指南
${await loadToolInstructions()}  // ← 这里加载 bash.txt 等提示
`;
```

**2. 工具定义 Schema（Zod 验证）**

```typescript
// packages/opencode/src/tool/bash.ts
export const BashTool = Tool.define({
  name: 'bash',
  description: 'Execute bash commands in the project directory',
  schema: z.object({
    command: z.string()
      .describe('The bash command to execute. Must be valid shell syntax.'),
    description: z.string()
      .describe('A clear, concise description (5-10 words)'),
    timeout: z.number().optional()
      .describe('Timeout in milliseconds (default: 120000)'),
  }),
  handle: async (params, context) => { /* ... */ }
});
```

**3. LLM 决策过程**

```
用户输入: "帮我检查一下 git 状态"

LLM 处理:
├─ 识别关键词: "检查", "git", "状态"
├─ 匹配工具: bash 工具适合执行 git 命令
├─ 参数推导:
│   ├─ command: "git status" (基于用户意图)
│   ├─ description: "Check git status" (简洁描述)
│   └─ timeout: 未提供 → 使用默认值
└─ 生成调用:
    {
      "tool": "bash",
      "params": {
        "command": "git status",
        "description": "Check git status"
      }
    }
```

---

### 第二阶段：权限检查（The Gatekeeper）

#### 🏠 类比：小区保安

想象你要进入一个高档小区：

```
你："我要去 5 号楼 202 室"
        ↓
保安检查：
  - 5 号楼是否在允许范围内？ → 是
  - 是否需要业主同意？ → 是（访客登记）
        ↓
保安呼叫业主："有一位访客想进您家，允许吗？"
        ↓
业主："允许" / "拒绝"
        ↓
保安：放行 / 阻拦
```

#### 🔬 技术细节

**1. 权限检查流程**

```typescript
// packages/opencode/src/permission/next.ts
async function checkPermission(toolCall, context) {
  // 1. 提取工具信息
  const { tool, params } = toolCall;

  // 2. 检查预定义规则
  const rule = findMatchingRule(tool, params);

  if (rule) {
    switch (rule.action) {
      case 'allow':
        return { allowed: true };  // 直接放行
      case 'deny':
        return { allowed: false, reason: rule.reason };  // 拒绝
      case 'ask':
        // 继续询问用户
        break;
    }
  }

  // 3. 默认行为：询问用户
  const userResponse = await promptUser({
    tool: tool.name,
    description: generateHumanReadableDescription(tool, params),
    params: params
  });

  return { allowed: userResponse === 'allow' };
}
```

**2. 权限规则示例**

```json
// opencode.json
{
  "permission": {
    "rules": [
      {
        "tool": "bash",
        "pattern": "git status",
        "action": "allow"
      },
      {
        "tool": "bash",
        "pattern": "rm -rf /*",
        "action": "deny",
        "reason": "Dangerous command"
      },
      {
        "tool": "edit",
        "pattern": "**/*.env",
        "action": "ask"
      }
    ]
  }
}
```

**3. 命令分类系统**

```typescript
// packages/opencode/src/permission/arity.ts
// 定义常见命令的参数数量，用于生成可读描述
const commandArity = {
  'git': 2,           // git <command>
  'git commit': 3,    // git commit <message>
  'npm run': 3,       // npm run <script>
  'docker compose': 3 // docker compose <command>
};

// 生成人类可读的命令描述
function describeCommand(cmd: string): string {
  // "git status" → "Run git status"
  // "npm run build" → "Run npm build script"
}
```

---

### 第三阶段：实际执行（The Executor）

#### 🏠 类比：工厂工人

想象工厂的生产线：

```
指令："制作一个红色的盒子"
        ↓
工人准备：
  - 检查材料（红色油漆、纸板）
  - 穿戴防护装备
        ↓
工人操作：
  - 裁剪纸板
  - 折叠成盒
  - 刷上红漆
        ↓
产出：红色盒子 + 工作报告（用时 5 分钟，耗材 2 单位）
```

#### 🔬 技术细节

**1. Bash 工具执行流程**

```typescript
// packages/opencode/src/tool/bash.ts
export const BashTool = Tool.define({
  // ... schema 定义 ...

  handle: async (params, context) => {
    const { command, timeout, description } = params;

    // 1. 解析命令结构（安全检查）
    const parsed = parseCommandWithTreeSitter(command);

    // 2. 检查目录限制
    if (parsed.hasCdCommand) {
      throw new Error('Use "workdir" parameter instead of "cd" command');
    }

    // 3. 确定工作目录
    const cwd = params.workdir || context.projectRoot;

    // 4. 执行命令
    const result = await new Promise((resolve, reject) => {
      const child = spawn(parsed.shell, ['-c', command], {
        cwd,
        timeout: timeout || 120000,
        env: sanitizeEnv(process.env)
      });

      let stdout = '';
      let stderr = '';

      child.stdout.on('data', (data) => {
        stdout += data.toString();
      });

      child.stderr.on('data', (data) => {
        stderr += data.toString();
      });

      child.on('close', (code) => {
        resolve({
          exitCode: code,
          stdout: truncateIfNeeded(stdout),
          stderr: truncateIfNeeded(stderr)
        });
      });
    });

    return result;
  }
});
```

**2. 完整的调用链图示**

```
用户输入
    ↓
┌─────────────────────────────────────────────────────────────┐
│                     Agent 处理层                             │
│  1. 构建系统提示（包含可用工具定义 + 使用指南）                │
│  2. 发送到 LLM                                              │
└─────────────────────────────────────────────────────────────┘
    ↓
LLM 返回工具调用请求
    ↓
┌─────────────────────────────────────────────────────────────┐
│                     权限控制层                               │
│  1. 解析工具调用（验证参数是否符合 schema）                   │
│  2. 检查权限规则                                            │
│  3. 如需确认，展示人类可读的描述，等待用户响应                 │
└─────────────────────────────────────────────────────────────┘
    ↓
用户批准
    ↓
┌─────────────────────────────────────────────────────────────┐
│                     工具执行层                               │
│  1. 调用 Tool.handle()                                      │
│  2. 执行具体操作（如 spawn bash 进程）                       │
│  3. 捕获输出                                                │
│  4. 截断处理（如果输出过大）                                 │
│  5. 返回结果                                                │
└─────────────────────────────────────────────────────────────┘
    ↓
结果返回给 LLM
    ↓
LLM 生成自然语言回复
    ↓
用户看到结果
```

---

## 第三部分：一个完整的例子

### 场景：用户想查看项目中有多少个 TypeScript 文件

#### Step 1: 用户输入

```
用户："帮我统计一下项目里有多少个 ts 文件"
```

#### Step 2: LLM 分析

```
LLM 思考过程：
- 关键词："统计"、"ts 文件"
- 需求：找到所有 .ts 文件并计数
- 可用工具选择：
  * bash: find . -name "*.ts" | wc -l（但 bash.txt 说不要用 find）
  * glob: pattern="**/*.ts"（更推荐）
- 选择：使用 glob 工具
- 参数：pattern = "**/*.ts"
```

**生成的工具调用：**
```json
{
  "tool": "glob",
  "params": {
    "pattern": "**/*.ts"
  }
}
```

#### Step 3: 权限检查

```
检查 glob 工具规则：
- glob 是只读操作，通常允许
- pattern "**/*.ts" 没有安全风险
- 结论：直接放行（无需用户确认）
```

#### Step 4: 工具执行

```typescript
// packages/opencode/src/tool/glob.ts
handle: async (params) => {
  const files = await fastGlob(params.pattern, {
    cwd: projectRoot,
    ignore: ['node_modules/**', '.git/**']
  });
  return {
    files: files,
    count: files.length
  };
}
```

**执行结果：**
```json
{
  "files": ["src/index.ts", "src/utils.ts", "src/types.ts"],
  "count": 3
}
```

#### Step 5: LLM 生成回复

```
LLM 收到结果：
- 原始数据：3 个文件
- 格式化回复："项目中共有 3 个 TypeScript 文件：
  - src/index.ts
  - src/utils.ts
  - src/types.ts"
```

#### Step 6: 用户看到

```
OpenCode: 项目中共有 3 个 TypeScript 文件：
  - src/index.ts
  - src/utils.ts
  - src/types.ts
```

---

## 第四部分：关键文件关系图

### 工具相关文件

```
packages/opencode/src/tool/
├── tool.ts          # 工具基类定义（Tool.define, Tool.Context）
├── registry.ts      # 工具注册中心（所有可用工具的列表）
├── bash.ts          # Bash 工具实现（代码）
├── bash.txt         # Bash 工具使用指南（给 LLM 的提示）
├── read.ts          # Read 工具实现
├── edit.ts          # Edit 工具实现
├── glob.ts          # Glob 工具实现
├── grep.ts          # Grep 工具实现
├── task.ts          # Task 工具实现（创建子 Agent）
└── ...
```

### 核心模块关系

```
┌─────────────────────────────────────────────────────────────┐
│                         用户界面                             │
│                     (packages/app)                          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                         Agent 层                             │
│              packages/opencode/src/agent/                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Agent.ts   │  │  Session.ts  │  │   Plan.ts    │      │
│  │  主 Agent    │  │  会话管理     │  │  计划模式     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                        工具管理层                            │
│              packages/opencode/src/tool/                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   tool.ts    │  │  registry.ts │  │  bash.ts     │      │
│  │  工具基类     │  │  工具注册     │  │  bash实现    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  bash.txt    │  │  read.ts     │  │  edit.ts     │      │
│  │  bash指南    │  │  read实现    │  │  edit实现    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                        权限控制层                            │
│           packages/opencode/src/permission/                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   next.ts    │  │   arity.ts   │  │   types.ts   │      │
│  │  权限检查主逻辑│  │  命令分类    │  │  类型定义    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                        系统交互层                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  shell.ts    │  │  child_process│  │   fs.ts     │      │
│  │  Shell 管理  │  │  Node.js 进程 │  │  文件操作   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## 第五部分：常见问题 FAQ

### Q1: LLM 怎么知道什么时候调用工具？

**简单回答**：就像你阅读说明书后知道何时使用工具一样，LLM 阅读了系统提示中的工具定义和使用指南后，就能判断何时需要调用工具。

**技术细节**：系统提示中明确告诉 LLM "当你需要执行 shell 命令时，使用 bash 工具"，LLM 根据这种描述和当前任务判断。

### Q2: 如果 LLM 选择了错误的工具怎么办？

**简单回答**：有双重保险。第一，参数 schema 验证会拦截明显错误的调用；第二，权限检查会让用户确认，用户可以拒绝执行。

**技术细节**：
1. Zod schema 验证会检查参数类型和必填项
2. 权限系统会根据规则决定是否拦截
3. 用户有最终否决权

### Q3: bash.txt 和 bash.ts 为什么不能合并？

**简单回答**：一个是"给 AI 看的说明书"，一个是"给机器执行的代码"。就像电器的用户手册和内部电路板是分开的一样。

**技术细节**：
- `bash.txt`：自然语言提示，帮助 LLM 理解最佳实践
- `bash.ts`：TypeScript 代码，实现实际的命令执行逻辑
- 分离使得可以在不修改代码的情况下调整 LLM 行为

### Q4: 为什么有些命令不需要确认，有些需要？

**简单回答**：就像小区门禁一样，业主回家直接刷卡通行的，但陌生人进小区需要登记。

**技术细节**：权限系统有预定义规则：
- 只读操作（如 `git status`、`read`）通常自动允许
- 修改操作（如 `edit`、`write`）需要确认
- 危险操作（如 `rm -rf`）默认拒绝

### Q5: Agent 和 Tool 的关系是什么？

**简单回答**：Agent 是"项目经理"，Tool 是"具体干活的工人"。经理决定做什么工作，工人负责实际执行。

**技术细节**：
- Agent：负责对话管理、任务规划、调用决策
- Tool：负责具体的原子操作（读文件、执行命令等）
- 一个 Agent 可以调用多个 Tools

---

## 附录：核心代码片段

### 工具定义模板

```typescript
// 定义一个新工具的模板
export const MyTool = Tool.define({
  // 工具名称
  name: 'my_tool',

  // 工具描述（LLM 用来看的）
  description: 'Does something useful',

  // 参数 schema（Zod 验证）
  schema: z.object({
    param1: z.string()
      .describe('Description for LLM'),
    param2: z.number().optional()
      .describe('Optional parameter')
  }),

  // 执行逻辑
  handle: async (params, context) => {
    // 1. 参数已验证，可以直接使用
    const { param1, param2 } = params;

    // 2. 执行具体操作
    const result = await doSomething(param1, param2);

    // 3. 返回结果
    return { success: true, data: result };
  }
});
```

### 权限配置模板

```json
{
  "permission": {
    "rules": [
      {
        "tool": "tool_name",
        "pattern": "regex_pattern",
        "action": "allow|deny|ask",
        "reason": "Optional explanation"
      }
    ]
  }
}
```

---

*文档生成时间：2024年*
*适用版本：OpenCode dev branch*
