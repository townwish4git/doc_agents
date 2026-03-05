# Codex Agent 工具调用机制参考文档

> 本文档总结 Codex 项目中 Agent 调用 Tools 的架构设计与实现机制。
>
> 涵盖内容：工具架构设计、调用流程、权限系统、Handler 实现、最佳实践

---

## 目录

1. [架构概览](#1-架构概览)
2. [核心组件](#2-核心组件)
3. [工具注册与发现](#3-工具注册与发现)
4. [完整调用链](#4-完整调用链)
5. [权限与沙箱系统](#5-权限与沙箱系统)
6. [工具 Handler 实现](#6-工具-handler-实现)
7. [MCP 工具集成](#7-mcp-工具集成)
8. [多 Agent 协作工具](#8-多-agent-协作工具)
9. [原生工具参考](#9-原生工具参考)
10. [实现参考代码](#10-实现参考代码)
11. [最佳实践总结](#11-最佳实践总结)

---

## 1. 架构概览

### 1.1 核心设计原则

```
┌─────────────────────────────────────────────────────────────────┐
│                    Codex Agent-Tool 核心原则                      │
├─────────────────────────────────────────────────────────────────┤
│ 1. 类型安全：Rust 类型系统确保工具调用的正确性                      │
│ 2. 沙箱隔离：所有命令执行在受控的沙箱环境中进行                      │
│ 3. 统一抽象：ToolHandler trait 统一处理不同类型的工具               │
│ 4. 可扩展性：支持 MCP 协议动态加载外部工具                          │
│ 5. 权限分级：AskForApproval 策略控制执行前审批                      │
│ 6. 错误恢复：沙箱失败后自动尝试升级沙箱重试执行                      │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 四层架构

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Agent/Session 层                                   │
│  - Session 管理对话状态                                        │
│  - TurnContext 管理单次交互上下文                               │
│  - 模型采样请求 (run_sampling_request)                         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 2: 工具路由层 (Tool Router)                            │
│  - ToolRouter 路由工具调用                                     │
│  - ToolRegistry 管理工具注册                                   │
│  - build_specs 构建工具规格                                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: 执行编排层 (Orchestrator)                           │
│  - ToolOrchestrator 管理执行流程                               │
│  - 权限审批 (AskForApproval)                                   │
│  - 沙箱选择与升级策略                                          │
│  - 错误重试机制                                                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  Layer 4: Handler 执行层                                      │
│  - ToolHandler trait 具体实现                                  │
│  - ShellHandler / McpHandler / MultiAgentHandler              │
│  - 实际系统调用执行                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 核心组件

### 2.1 Session 与 TurnContext

```rust
// codex-rs/core/src/codex.rs
pub(crate) struct Session {
    pub(crate) conversation_id: ThreadId,
    tx_event: Sender<Event>,
    agent_status: watch::Sender<AgentStatus>,
    state: Mutex<SessionState>,
    features: ManagedFeatures,
    pending_mcp_server_refresh_config: Mutex<Option<McpServerRefreshConfig>>,
    pub(crate) conversation: Arc<RealtimeConversationManager>,
    pub(crate) active_turn: Mutex<Option<ActiveTurn>>,
    pub(crate) services: SessionServices,
    js_repl: Arc<JsReplHandle>,
    next_internal_sub_id: AtomicU64,
}

// 单次交互的上下文
pub(crate) struct TurnContext {
    pub config: Arc<Config>,
    pub model: Model,
    pub headers: HeaderMap,
    pub approval_policy: AskForApproval,
    pub sandbox_policy: SandboxPolicy,
    pub full_automation: FullAutomation,
    pub notify_computer_abilities: NotifyComputerAbilities,
    pub max_tool_iterations: u32,
    pub tools: ToolsConfig,
    pub project_directory: PathBuf,
}
```

### 2.2 工具调用上下文

```rust
// codex-rs/core/src/tools/context.rs
#[derive(Clone)]
pub struct ToolInvocation {
    pub session: Arc<Session>,          // 会话引用
    pub turn: Arc<TurnContext>,         // 当前交互上下文
    pub tracker: SharedTurnDiffTracker, // 变更追踪
    pub call_id: String,                // 调用唯一ID
    pub tool_name: String,              // 工具名称
    pub payload: ToolPayload,           // 调用载荷
}

#[derive(Clone, Debug)]
pub enum ToolPayload {
    Function { arguments: String },           // 标准函数调用
    Custom { input: String },                 // 自定义工具
    LocalShell { params: ShellToolCallParams }, // 本地shell
    Mcp { server: String, tool: String, raw_arguments: String }, // MCP工具
}
```

### 2.3 ToolHandler Trait

```rust
// codex-rs/core/src/tools/registry.rs
#[async_trait]
pub trait ToolHandler: Send + Sync {
    /// 返回工具类型
    fn kind(&self) -> ToolKind;

    /// 检查是否能处理该类型的 payload
    fn matches_kind(&self, payload: &ToolPayload) -> bool {
        matches!(
            (self.kind(), payload),
            (ToolKind::Function, ToolPayload::Function { .. })
                | (ToolKind::Mcp, ToolPayload::Mcp { .. })
        )
    }

    /// 判断调用是否可能改变环境（用于权限检查）
    async fn is_mutating(&self, _invocation: &ToolInvocation) -> bool {
        false
    }

    /// 执行工具调用
    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError>;
}
```

---

## 3. 工具注册与发现

### 3.1 工具注册中心构建器

```rust
// codex-rs/core/src/tools/registry.rs
pub struct ToolRegistryBuilder {
    handlers: HashMap<String, Arc<dyn ToolHandler>>,
    specs: Vec<ConfiguredToolSpec>,
}

impl ToolRegistryBuilder {
    pub fn new() -> Self {
        Self {
            handlers: HashMap::new(),
            specs: Vec::new(),
        }
    }

    /// 注册工具规格
    pub fn push_spec(&mut self, spec: ToolSpec) {
        self.push_spec_with_parallel_support(spec, false);
    }

    /// 注册工具处理器
    pub fn register_handler(&mut self, name: impl Into<String>, handler: Arc<dyn ToolHandler>) {
        let name = name.into();
        if self.handlers.insert(name.clone(), handler.clone()).is_some() {
            warn!("overwriting handler for tool {name}");
        }
    }

    /// 构建最终注册表
    pub fn build(self) -> (Vec<ConfiguredToolSpec>, ToolRegistry) {
        let registry = ToolRegistry::new(self.handlers);
        (self.specs, registry)
    }
}
```

### 3.2 工具规格构建流程

```rust
// codex-rs/core/src/tools/spec.rs
pub(crate) fn build_specs(
    config: &ToolsConfig,
    mcp_tools: Option<HashMap<String, rmcp::model::Tool>>,
    app_tools: Option<HashMap<String, ToolInfo>>,
    dynamic_tools: &[DynamicToolSpec],
) -> ToolRegistryBuilder {
    let mut builder = ToolRegistryBuilder::new();

    // 1. 注册 Shell 处理器
    let shell_handler = Arc::new(ShellHandler);
    let unified_exec_handler = Arc::new(UnifiedExecHandler);

    // 根据配置选择 shell 工具类型
    match &config.shell_type {
        ConfigShellToolType::Default => {
            builder.push_spec_with_parallel_support(
                create_shell_tool(request_permission_enabled),
                true
            );
        }
        ConfigShellToolType::UnsafeReplacement { .. } => {
            // 注册替代 shell 工具
        }
    }
    builder.register_handler("shell", shell_handler.clone());
    builder.register_handler("container.exec", shell_handler.clone());

    // 2. 注册文件操作工具
    if config.file_tools {
        builder.push_spec(create_read_file_tool());
        builder.register_handler("read_file", Arc::new(ReadFileHandler));

        builder.push_spec(create_write_file_tool());
        builder.register_handler("write_file", Arc::new(WriteFileHandler));

        builder.push_spec(create_edit_file_tool());
        builder.register_handler("edit_file", Arc::new(EditFileHandler));
    }

    // 3. 注册 MCP 工具（如果启用）
    if let Some(mcp_tools) = mcp_tools {
        for (name, tool) in mcp_tools {
            match mcp_tool_to_openai_tool(name.clone(), tool.clone()) {
                Ok(converted_tool) => {
                    builder.push_spec(ToolSpec::Function(converted_tool));
                    builder.register_handler(name, mcp_handler.clone());
                }
                Err(e) => warn!("Failed to convert MCP tool {name}: {e}"),
            }
        }
    }

    // 4. 注册协作工具（如果启用多 Agent）
    if config.collab_tools {
        let multi_agent_handler = Arc::new(MultiAgentHandler);
        builder.push_spec(create_spawn_agent_tool(config));
        builder.register_handler("spawn_agent", multi_agent_handler.clone());
    }

    // 5. 注册实验性工具
    if config.js_repl_enabled {
        builder.push_spec(create_js_repl_tool());
        builder.register_handler("js_repl", js_repl_handler);
    }

    builder
}
```

### 3.3 工具路由配置

```rust
// codex-rs/core/src/tools/router.rs
pub struct ToolRouter {
    registry: ToolRegistry,
    specs: Vec<ConfiguredToolSpec>,
}

impl ToolRouter {
    pub fn from_config(
        config: &ToolsConfig,
        mcp_tools: Option<HashMap<String, Tool>>,
        app_tools: Option<HashMap<String, ToolInfo>>,
        dynamic_tools: &[DynamicToolSpec],
    ) -> Self {
        let builder = build_specs(config, mcp_tools, app_tools, dynamic_tools);
        let (specs, registry) = builder.build();
        Self { registry, specs }
    }

    /// 获取发送给模型的工具规格列表
    pub fn tool_specs(&self) -> Vec<ConfiguredToolSpec> {
        self.specs.clone()
    }
}
```

---

## 4. 完整调用链

### 4.1 调用流程图

```
用户输入: "帮我读取 config.ts 的内容"
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Agent 构建 Sampling Request                          │
│                                                             │
│ 1. 获取可用工具规格 (ToolRouter.tool_specs())                 │
│ 2. 构建系统提示词（包含工具定义）                              │
│ 3. 发送请求到模型                                             │
│                                                             │
│ Tools Spec: [                                                │
│   { name: "read_file", params: {...} },                    │
│   { name: "shell", params: {...} },                        │
│   ...                                                       │
│ ]                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: 模型决策与工具调用                                     │
│                                                             │
│ 模型输出 FunctionCall:                                        │
│ {                                                           │
│   "name": "read_file",                                     │
│   "arguments": "{\"file_path\": \"config.ts\"}",            │
│   "call_id": "call_abc123"                                 │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: 构建 ToolCall                                        │
│                                                             │
│ ToolRouter::build_tool_call():                               │
│ - 解析模型输出的 FunctionCall                                │
│ - 识别 MCP 工具（通过 parse_mcp_tool_name）                   │
│ - 封装为 ToolCall 结构                                        │
│                                                             │
│ ToolCall {                                                  │
│   tool_name: "read_file",                                  │
│   call_id: "call_abc123",                                  │
│   payload: Function { arguments: "..." }                   │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: 工具路由与分发                                         │
│                                                             │
│ ToolRouter::dispatch_tool_call():                            │
│ 1. 创建 ToolInvocation 上下文                                │
│ 2. 查询 Registry 获取对应 Handler                            │
│ 3. 调用 registry.dispatch(invocation)                        │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: 权限审批（Orchestrator）                              │
│                                                             │
│ ToolOrchestrator::run():                                     │
│ 1. 检查 exec_approval_requirement                            │
│ 2. 根据 AskForApproval 策略决定：                             │
│    - Skip: 跳过审批                                          │
│    - NeedsApproval: 请求用户确认                              │
│    - Forbidden: 拒绝执行                                     │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 6: 沙箱选择与执行                                         │
│                                                             │
│ 1. 选择初始沙箱（select_initial）                              │
│ 2. 在沙箱中执行工具调用                                        │
│ 3. 如果沙箱拒绝，升级沙箱重试                                  │
│ 4. 返回执行结果                                               │
└─────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 7: 结果返回给模型                                         │
│                                                             │
│ 将 ToolOutput 转换为 ResponseInputItem                      │
│ 追加到对话历史，供模型生成回复                                 │
│                                                             │
│ ResponseInputItem::ToolOutput {                             │
│   call_id: "call_abc123",                                  │
│   output: "export const config = {...}"                    │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
           ↓
用户看到模型生成的回复
```

### 4.2 关键数据结构

```rust
// 工具调用请求
pub struct ToolCall {
    pub tool_name: String,
    pub call_id: String,
    pub payload: ToolPayload,
}

// 工具输出
pub enum ToolOutput {
    Function { output: String },
    Mcp { result: String },
    Custom { output: String },
}

// 执行审批要求
pub enum ExecApprovalRequirement {
    Skip { reason: String },
    Forbidden { reason: String },
    NeedsApproval {
        title: String,
        reason: String,
    },
}
```

---

## 5. 权限与沙箱系统

### 5.1 审批策略

```rust
// codex-rs/core/src/approval.rs
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub enum AskForApproval {
    #[default]
    Auto,      // 自动决定（基于启发式规则）
    Always,    // 总是询问
    Never,     // 从不询问（危险）
}

impl AskForApproval {
    /// 结合沙箱策略确定是否需要审批
    pub fn exec_approval_requirement(
        self,
        sandbox_policy: SandboxPolicy,
        reason: &str,
    ) -> ExecApprovalRequirement {
        match self {
            AskForApproval::Never => ExecApprovalRequirement::Skip {
                reason: "AskForApproval::Never".into(),
            },
            AskForApproval::Always => ExecApprovalRequirement::NeedsApproval {
                title: "Tool execution".into(),
                reason: reason.into(),
            },
            AskForApproval::Auto => {
                // 根据沙箱策略和命令风险自动判断
                if sandbox_policy.is_restricted() {
                    ExecApprovalRequirement::NeedsApproval { ... }
                } else {
                    ExecApprovalRequirement::Skip { ... }
                }
            }
        }
    }
}
```

### 5.2 沙箱策略

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum SandboxPolicy {
    // 允许所有操作（仅本地 CLI 使用）
    DangerousNoSandboxButMightUseNetwork,
    // 使用容器沙箱
    NetworkAndContainer,
    // 仅网络隔离
    NetworkOnly,
}

impl SandboxPolicy {
    /// 沙箱是否提供有意义的隔离
    pub fn is_restricted(&self) -> bool {
        matches!(self, SandboxPolicy::NetworkAndContainer)
    }
}
```

### 5.3 工具编排器

```rust
// codex-rs/core/src/tools/orchestrator.rs
pub(crate) struct ToolOrchestrator {
    sandbox: SandboxManager,
}

impl ToolOrchestrator {
    pub async fn run<Rq, Out, T>(
        &mut self,
        tool: &mut T,
        req: &Rq,
        tool_ctx: &ToolCtx,
        turn_ctx: &TurnContext,
        approval_policy: AskForApproval,
    ) -> Result<OrchestratorRunResult<Out>, ToolError>
    where
        T: ToolRuntime<Rq, Out>,
    {
        // 1) 确定审批要求
        let requirement = tool.exec_approval_requirement(req)
            .unwrap_or_else(|| default_exec_approval_requirement(approval_policy, &turn_ctx.sandbox_policy));

        // 2) 处理审批
        match requirement {
            ExecApprovalRequirement::Skip { .. } => {}
            ExecApprovalRequirement::Forbidden { reason } => {
                return Err(ToolError::Rejected(reason));
            }
            ExecApprovalRequirement::NeedsApproval { reason, .. } => {
                let decision = tool.start_approval_async(req, approval_ctx).await;
                match decision {
                    ApprovalDecision::Allow => {}
                    ApprovalDecision::Deny => return Err(ToolError::Rejected("User denied".into())),
                }
            }
        }

        // 3) 选择初始沙箱
        let initial_sandbox = self.sandbox.select_initial(...);

        // 4) 第一次尝试执行
        let (first_result, _) = Self::run_attempt(
            tool, req, tool_ctx, &initial_attempt, has_managed_network_requirements
        ).await;

        // 5) 如果沙箱拒绝，尝试升级沙箱重试
        match first_result {
            Ok(out) => Ok(OrchestratorRunResult { output: out, ... }),
            Err(ToolError::Codex(CodexErr::Sandbox(SandboxErr::Denied { msg, recovery }))) => {
                if recovery == SandboxRecovery::RetryWithEscalatedSandbox {
                    let escalated = self.sandbox.escalate(&initial_attempt);
                    let (retry_result, _) = Self::run_attempt(...).await;
                    match retry_result {
                        Ok(out) => Ok(OrchestratorRunResult { output: out, ... }),
                        Err(err) => Err(err),
                    }
                } else {
                    Err(first_result.unwrap_err())
                }
            }
            Err(err) => Err(err),
        }
    }
}
```

---

## 6. 工具 Handler 实现

### 6.1 Shell Handler

```rust
// codex-rs/core/src/tools/handlers/shell.rs
pub struct ShellHandler;

#[async_trait]
impl ToolHandler for ShellHandler {
    fn kind(&self) -> ToolKind {
        ToolKind::Function
    }

    fn matches_kind(&self, payload: &ToolPayload) -> bool {
        matches!(payload,
            ToolPayload::Function { .. } | ToolPayload::LocalShell { .. }
        )
    }

    /// 判断是否可能改变环境
    async fn is_mutating(&self, invocation: &ToolInvocation) -> bool {
        match &invocation.payload {
            ToolPayload::Function { arguments } => {
                serde_json::from_str::<ShellToolCallParams>(arguments)
                    .map(|params| !is_known_safe_command(&params.command))
                    .unwrap_or(true)
            }
            ToolPayload::LocalShell { params } => !is_known_safe_command(&params.command),
            _ => true,
        }
    }

    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
        let ToolInvocation { session, turn, call_id, payload, .. } = invocation;

        // 解析参数
        let params = match payload {
            ToolPayload::Function { arguments } => {
                serde_json::from_str::<ShellToolCallParams>(&arguments)
                    .map_err(|e| FunctionCallError::Fatal(e.to_string()))?
            }
            ToolPayload::LocalShell { params } => params,
            _ => return Err(FunctionCallError::Fatal("Invalid payload".into())),
        };

        // 创建执行参数
        let exec_params = Self::to_exec_params(&params, turn.as_ref(), session.conversation_id);

        // 执行命令
        Self::run_exec_like(RunExecLikeArgs {
            exec_params,
            session,
            turn,
            call_id,
        }).await
    }
}
```

### 6.2 MCP Handler

```rust
// codex-rs/core/src/tools/handlers/mcp.rs
pub struct McpHandler;

#[async_trait]
impl ToolHandler for McpHandler {
    fn kind(&self) -> ToolKind {
        ToolKind::Mcp
    }

    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
        let ToolInvocation { session, turn, call_id, payload, .. } = invocation;

        // 解析 MCP 参数
        let (server, tool, raw_arguments) = match payload {
            ToolPayload::Mcp { server, tool, raw_arguments } => (server, tool, raw_arguments),
            _ => return Err(FunctionCallError::Fatal("Invalid payload for MCP".into())),
        };

        // 调用 MCP 工具
        let response = handle_mcp_tool_call(
            Arc::clone(&session),
            turn.as_ref(),
            call_id.clone(),
            server,
            tool,
            raw_arguments,
        ).await;

        // 转换结果为 ToolOutput
        match response {
            ResponseInputItem::McpToolCallOutput { result, .. } => {
                Ok(ToolOutput::Mcp { result })
            }
            _ => Err(FunctionCallError::Fatal("Unexpected MCP response".into())),
        }
    }
}
```

### 6.3 多 Agent Handler

```rust
// codex-rs/core/src/tools/handlers/multi_agent.rs
pub struct MultiAgentHandler;

#[async_trait]
impl ToolHandler for MultiAgentHandler {
    fn kind(&self) -> ToolKind {
        ToolKind::Function
    }

    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
        // 解析 spawn_agent 参数
        let params: SpawnAgentParams = ...;

        // 创建子 Agent 会话
        let sub_session = session.create_sub_session(&params).await?;

        // 运行子 Agent 任务
        let result = sub_session.run_task(&params.prompt).await?;

        // 返回结果
        Ok(ToolOutput::Function {
            output: serde_json::to_string(&result)?,
        })
    }
}
```

---

## 7. MCP 工具集成

### 7.1 MCP 工具发现

```rust
// MCP 工具通过 servers 配置动态加载
pub async fn refresh_mcp_servers(
    &self,
    config: McpServerRefreshConfig,
) -> Result<Vec<McpServerRefreshResult>, McpError> {
    let mut results = Vec::new();

    for (server_name, server_config) in &config.servers {
        match self.connect_to_mcp_server(server_name, server_config).await {
            Ok(connection) => {
                // 获取该服务器提供的工具列表
                let tools = connection.list_tools().await?;

                // 注册工具到会话
                for tool in tools {
                    let full_name = format!("{}_{}", server_name, tool.name);
                    self.register_mcp_tool(full_name, tool).await;
                }

                results.push(McpServerRefreshResult::Success { server_name });
            }
            Err(e) => {
                results.push(McpServerRefreshResult::Failure {
                    server_name,
                    error: e.to_string(),
                });
            }
        }
    }

    Ok(results)
}
```

### 7.2 MCP 工具名称解析

```rust
// 解析形如 "serverName_toolName" 的工具名称
pub async fn parse_mcp_tool_name(&self, name: &str) -> Option<(String, String)> {
    // 遍历已知的 MCP 服务器前缀
    for server_name in self.mcp_servers.keys() {
        let prefix = format!("{}_", server_name);
        if name.starts_with(&prefix) {
            let tool_name = name[prefix.len()..].to_string();
            return Some((server_name.clone(), tool_name));
        }
    }
    None
}
```

---

## 8. 多 Agent 协作工具

### 8.1 Agent 创建与配置

```rust
// spawn_agent 工具规格
create_spawn_agent_tool(config: &ToolsConfig) -> ToolSpec {
    ToolSpec::Function(FunctionTool {
        name: "spawn_agent".into(),
        description: "创建一个专门处理特定任务的子 Agent".into(),
        parameters: json!({
            "type": "object",
            "properties": {
                "description": { "type": "string", "description": "子任务的简短描述" },
                "prompt": { "type": "string", "description": "给子 Agent 的完整指令" },
                "subagent_type": {
                    "type": "string",
                    "enum": ["Explore", "Plan", "TestRunner", ...],
                    "description": "子 Agent 类型"
                },
            },
            "required": ["description", "prompt"],
        }),
    })
}
```

### 8.2 Agent 间通信

```rust
// 父 Agent 可以通过以下方式与子 Agent 通信：
// 1. 在 prompt 中传递上下文
// 2. 通过 session 共享状态
// 3. 子 Agent 完成后返回结构化结果

pub struct SubAgentResult {
    pub success: bool,
    pub output: String,
    pub artifacts: Vec<Artifact>,
}

// 子 Agent 可以调用所有可用工具
// 子 Agent 完成后，结果返回给父 Agent
```

---

## 9. 原生工具参考

### 9.1 工具分类概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      Codex 原生工具分类                           │
├─────────────────────────────────────────────────────────────────┤
│ 1. Shell 执行类    │ shell, shell_command, exec_command          │
│ 2. 文件操作类      │ read_file, list_dir                         │
│ 3. 代码编辑类      │ apply_patch                                 │
│ 4. 搜索类          │ grep_files, search_tool_bm25                │
│ 5. 多 Agent 协作   │ spawn_agent, send_input, wait, close_agent  │
│ 6. 批处理任务      │ spawn_agents_on_csv                         │
│ 7. JavaScript 执行 │ js_repl, js_repl_reset                      │
│ 8. MCP 集成        │ list_mcp_resources, read_mcp_resource       │
│ 9. 计划管理        │ update_plan                                 │
│ 10. 交互输入       │ request_user_input                          │
│ 11. 外部服务       │ web_search, image_generation                │
│ 12. 可视化         │ view_image                                  │
│ 13. 产物生成       │ presentation_artifact, spreadsheet_artifact │
│ 14. 测试同步       │ test_sync_tool                              │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Shell 执行类工具

#### `shell`
- **用途**: 执行 shell 命令（跨平台，Windows 使用 PowerShell）
- **类型**: Function Tool
- **核心参数**:
  - `command`: string[] - 命令参数数组（如 `["bash", "-lc", "git status"]`）
  - `workdir`: string - 工作目录（可选）
  - `timeout_ms`: number - 超时时间（可选）
- **启用条件**: 默认启用（可通过配置禁用）
- **并行支持**: 是
- **特殊说明**: 自动根据平台选择执行方式（bash/PowerShell）

#### `shell_command`
- **用途**: 直接执行 shell 脚本字符串
- **类型**: Function Tool
- **核心参数**:
  - `command`: string - 完整的 shell 脚本
  - `workdir`: string - 工作目录
  - `timeout_ms`: number - 超时时间
  - `login`: boolean - 是否使用 login shell（可选）
- **启用条件**: 需要 `ShellZshFork` 特性
- **并行支持**: 是

#### `exec_command` / `write_stdin`
- **用途**: UnifiedExec 模式下的命令执行和交互输入
- **类型**: Function Tool
- **启用条件**: 需要 `UnifiedExec` 特性
- **并行支持**: 是

### 9.3 文件操作类工具

#### `read_file`
- **用途**: 读取文件内容，支持行号和缩进感知模式
- **类型**: Function Tool（实验性，默认启用）
- **核心参数**:
  - `file_path`: string - 文件的绝对路径（必需）
  - `offset`: number - 起始行号（1-indexed，可选）
  - `limit`: number - 最大返回行数（可选）
  - `mode`: "slice" | "indentation" - 读取模式（可选）
  - `indentation`: object - 缩进模式配置（可选）
    - `anchor_line`: number - 锚定行号
    - `max_levels`: number - 最大缩进层级
    - `include_siblings`: boolean - 是否包含同级代码块
    - `include_header`: boolean - 是否包含文档注释
    - `max_lines`: number - 最大行数限制
- **启用条件**: 需要 `experimental_supported_tools` 包含 `"read_file"`
- **并行支持**: 是

#### `list_dir`
- **用途**: 列出目录内容
- **类型**: Function Tool（实验性）
- **核心参数**:
  - `dir_path`: string - 目录的绝对路径（必需）
  - `offset`: number - 起始条目序号（1-indexed，可选）
  - `limit`: number - 最大返回条目数（可选）
  - `depth`: number - 最大遍历深度（可选）
- **启用条件**: 需要 `experimental_supported_tools` 包含 `"list_dir"`
- **并行支持**: 是

### 9.4 代码编辑类工具

#### `apply_patch`
- **用途**: 应用代码补丁，支持添加/删除/更新文件
- **类型**: Freeform Tool（默认）或 Function Tool
- **模式**:
  - **Freeform 模式**: 直接发送补丁文本，使用 Lark 语法验证
  - **JSON 模式**: 通过 `input` 参数传递补丁内容
- **补丁语法**:
```
*** Begin Patch
*** Add File: <path>
+<content>
*** Update File: <path>
*** Move to: <new_path>
@@ <context>
-<old_line>
+<new_line>
*** Delete File: <path>
*** End Patch
```
- **启用条件**: 需要 `ApplyPatchFreeform` 特性
- **Handler**: `ApplyPatchHandler`

### 9.5 搜索类工具

#### `grep_files`
- **用途**: 使用正则表达式搜索文件内容
- **类型**: Function Tool（实验性）
- **核心参数**:
  - `pattern`: string - 正则表达式模式（必需）
  - `include`: string - 文件匹配模式（如 `"*.rs"`，可选）
  - `path`: string - 搜索路径（默认为工作目录，可选）
  - `limit`: number - 最大返回文件数（默认 100，可选）
- **启用条件**: 需要 `experimental_supported_tools` 包含 `"grep_files"`
- **并行支持**: 是

#### `search_tool_bm25`
- **用途**: 使用 BM25 算法搜索 App 工具
- **类型**: Function Tool
- **核心参数**:
  - `query`: string - 搜索查询
  - `limit`: number - 最大返回工具数（默认 10）
- **启用条件**: 需要 `search_tool` 配置为 true 且有可用的 app_tools
- **并行支持**: 是

### 9.6 多 Agent 协作工具

#### `spawn_agent`
- **用途**: 创建子 Agent 处理特定任务
- **类型**: Function Tool
- **核心参数**:
  - `message`: string - 初始任务描述（与 items 二选一）
  - `items`: array - 结构化输入项（与 message 二选一）
  - `agent_type`: string - Agent 角色类型
  - `fork_context`: boolean - 是否复制当前对话上下文
- **返回**: Agent ID 和昵称
- **启用条件**: 需要 `Collab` 特性
- **Handler**: `MultiAgentHandler`

#### `send_input`
- **用途**: 向运行中的 Agent 发送输入
- **启用条件**: 需要 `Collab` 特性
- **Handler**: `MultiAgentHandler`

#### `resume_agent`
- **用途**: 恢复暂停的 Agent
- **启用条件**: 需要 `Collab` 特性
- **Handler**: `MultiAgentHandler`

#### `wait`
- **用途**: 等待 Agent 完成
- **启用条件**: 需要 `Collab` 特性
- **Handler**: `MultiAgentHandler`

#### `close_agent`
- **用途**: 关闭 Agent 连接
- **启用条件**: 需要 `Collab` 特性
- **Handler**: `MultiAgentHandler`

### 9.7 批处理任务工具

#### `spawn_agents_on_csv`
- **用途**: 基于 CSV 文件批量创建 Agent 任务
- **类型**: Function Tool
- **核心参数**:
  - `csv_path`: string - CSV 文件路径
  - `instruction`: string - 指令模板（支持 `{column_name}` 占位符）
  - `id_column`: string - 用作 ID 的列名（可选）
  - `output_csv_path`: string - 输出 CSV 路径（可选）
  - `max_concurrency`: number - 最大并发数（默认 16）
- **启用条件**: 需要 `agent_jobs_tools` 和 `Sqlite` 特性
- **Handler**: `BatchJobHandler`

#### `report_agent_job_result`
- **用途**: 报告 Agent 批处理任务结果
- **启用条件**: 仅 worker Agent 可用

### 9.8 JavaScript 执行工具

#### `js_repl`
- **用途**: 在持久的 Node 环境中执行 JavaScript，支持 top-level await
- **类型**: Freeform Tool
- **语法**: 原生 JavaScript（非 JSON）
- **Pragma 支持**: 支持首行注释设置参数，如 `// codex-js-repl: timeout_ms=15000`
- **启用条件**: 需要 `JsRepl` 特性
- **Handler**: `JsReplHandler`

#### `js_repl_reset`
- **用途**: 重启 js_repl 内核并清除所有绑定
- **类型**: Function Tool（无参数）
- **启用条件**: 需要 `JsRepl` 特性
- **Handler**: `JsReplResetHandler`

### 9.9 MCP 集成工具

#### `list_mcp_resources`
- **用途**: 列出 MCP 服务器提供的资源
- **类型**: Function Tool
- **核心参数**:
  - `server`: string - MCP 服务器名称（可选，省略则列出所有）
  - `cursor`: string - 分页游标（可选）
- **启用条件**: 配置了 MCP 服务器
- **并行支持**: 是
- **Handler**: `McpResourceHandler`

#### `list_mcp_resource_templates`
- **用途**: 列出 MCP 资源模板
- **启用条件**: 配置了 MCP 服务器
- **并行支持**: 是

#### `read_mcp_resource`
- **用途**: 读取 MCP 资源内容
- **启用条件**: 配置了 MCP 服务器
- **并行支持**: 是

### 9.10 计划管理工具

#### `update_plan`
- **用途**: 更新任务计划
- **类型**: Function Tool
- **核心参数**:
  - `explanation`: string - 计划说明（可选）
  - `plan`: array - 计划项列表（必需）
    - `step`: string - 步骤描述
    - `status`: "pending" | "in_progress" | "completed" - 状态
- **约束**: 最多一个步骤可以处于 `in_progress` 状态
- **启用条件**: 始终启用
- **Handler**: `PlanHandler`

### 9.11 交互输入工具

#### `request_user_input`
- **用途**: 请求用户输入
- **类型**: Function Tool
- **核心参数**:
  - `prompt`: string - 提示信息
  - `type`: string - 输入类型（可选）
- **启用条件**: 始终启用
- **Handler**: `RequestUserInputHandler`

### 9.12 外部服务工具

#### `web_search`
- **用途**: 网络搜索
- **类型**: 内置工具（非 Function Tool）
- **模式**:
  - `Cached`: 使用缓存结果（不访问外部网络）
  - `Live`: 实时搜索（访问外部网络）
- **启用条件**: 需要配置 `web_search_mode`

#### `image_generation`
- **用途**: 生成图像
- **类型**: 内置工具
- **启用条件**: 需要 `ImageGeneration` 特性且模型支持图像生成

### 9.13 可视化工具

#### `view_image`
- **用途**: 查看本地图像文件
- **类型**: Function Tool
- **核心参数**:
  - `path`: string - 本地图像文件路径
- **启用条件**: 始终启用
- **并行支持**: 是
- **Handler**: `ViewImageHandler`

### 9.14 产物生成工具

#### `presentation_artifact`
- **用途**: 创建演示文稿产物
- **类型**: Function Tool
- **启用条件**: 需要 `Artifact` 特性
- **Handler**: `PresentationArtifactHandler`

#### `spreadsheet_artifact`
- **用途**: 创建电子表格产物
- **类型**: Function Tool
- **启用条件**: 需要 `Artifact` 特性
- **Handler**: `SpreadsheetArtifactHandler`

### 9.15 测试同步工具

#### `test_sync_tool`
- **用途**: 测试同步功能（开发/测试用）
- **类型**: Function Tool
- **启用条件**: 需要 `experimental_supported_tools` 包含 `"test_sync_tool"`
- **并行支持**: 是
- **Handler**: `TestSyncHandler`

### 9.16 工具启用配置对照表

| 工具 | Feature Flag | Config 字段 | 默认状态 |
|------|-------------|-------------|----------|
| shell | `ShellTool` | `shell_type` | 启用 |
| shell_command | `ShellZshFork` | `shell_type` | 可选 |
| exec_command | `UnifiedExec` | `shell_type` | 可选 |
| read_file | - | `experimental_supported_tools` | 实验性 |
| list_dir | - | `experimental_supported_tools` | 实验性 |
| grep_files | - | `experimental_supported_tools` | 实验性 |
| apply_patch | `ApplyPatchFreeform` | `apply_patch_tool_type` | 可选 |
| js_repl | `JsRepl` | `js_repl_enabled` | 可选 |
| spawn_agent | `Collab` | `collab_tools` | 可选 |
| spawn_agents_on_csv | `Collab` + `Sqlite` | `agent_jobs_tools` | 可选 |
| search_tool_bm25 | `Apps` | `search_tool` | 可选 |
| web_search | - | `web_search_mode` | 可选 |
| image_generation | `ImageGeneration` | `image_gen_tool` | 可选（模型依赖） |
| presentation_artifact | `Artifact` | `artifact_tools` | 可选 |
| spreadsheet_artifact | `Artifact` | `artifact_tools` | 可选 |
| update_plan | - | - | 始终启用 |
| request_user_input | - | - | 始终启用 |
| view_image | - | - | 始终启用 |
| list_mcp_resources | - | 配置 MCP 服务器 | 自动 |

### 9.17 工具 Handler 映射表

| 工具名称 | Handler 类型 | 注册别名 |
|---------|-------------|---------|
| shell | `ShellHandler` | `shell`, `container.exec`, `local_shell` |
| shell_command | `ShellCommandHandler` | `shell_command` |
| exec_command / write_stdin | `UnifiedExecHandler` | `exec_command`, `write_stdin` |
| read_file | `ReadFileHandler` | `read_file` |
| list_dir | `ListDirHandler` | `list_dir` |
| grep_files | `GrepFilesHandler` | `grep_files` |
| apply_patch | `ApplyPatchHandler` | `apply_patch` |
| js_repl | `JsReplHandler` | `js_repl` |
| js_repl_reset | `JsReplResetHandler` | `js_repl_reset` |
| update_plan | `PlanHandler` | `update_plan` |
| request_user_input | `RequestUserInputHandler` | `request_user_input` |
| spawn_agent / send_input / resume_agent / wait / close_agent | `MultiAgentHandler` | `spawn_agent`, `send_input`, `resume_agent`, `wait`, `close_agent` |
| spawn_agents_on_csv / report_agent_job_result | `BatchJobHandler` | `spawn_agents_on_csv`, `report_agent_job_result` |
| search_tool_bm25 | `SearchToolBm25Handler` | `search_tool_bm25` |
| view_image | `ViewImageHandler` | `view_image` |
| presentation_artifact | `PresentationArtifactHandler` | `presentation_artifact` |
| spreadsheet_artifact | `SpreadsheetArtifactHandler` | `spreadsheet_artifact` |
| list_mcp_resources / list_mcp_resource_templates / read_mcp_resource | `McpResourceHandler` | `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource` |
| MCP 动态工具 | `McpHandler` | `<server_name>_<tool_name>` |

---

## 10. 实现参考代码

### 10.1 自定义 Tool Handler 模板

```rust
use crate::tools::{ToolHandler, ToolInvocation, ToolOutput, ToolKind};
use crate::error::FunctionCallError;
use async_trait::async_trait;

pub struct MyCustomHandler;

#[async_trait]
impl ToolHandler for MyCustomHandler {
    fn kind(&self) -> ToolKind {
        ToolKind::Function
    }

    /// 可选：判断调用是否会改变环境
    async fn is_mutating(&self, invocation: &ToolInvocation) -> bool {
        // 解析参数判断是否为修改操作
        true
    }

    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
        let ToolInvocation { session, turn, call_id, payload, .. } = invocation;

        // 1. 解析参数
        let args = match payload {
            ToolPayload::Function { arguments } => {
                serde_json::from_str::<MyParams>(&arguments)
                    .map_err(|e| FunctionCallError::Fatal(e.to_string()))?
            }
            _ => return Err(FunctionCallError::Fatal("Invalid payload".into())),
        };

        // 2. 执行操作
        let result = do_something(args).await
            .map_err(|e| FunctionCallError::Fatal(e.to_string()))?;

        // 3. 返回结构化结果
        Ok(ToolOutput::Function {
            output: serde_json::to_string(&result)
                .map_err(|e| FunctionCallError::Fatal(e.to_string()))?,
        })
    }
}
```

### 10.2 工具注册

```rust
// 在 build_specs 中添加自定义工具
pub(crate) fn build_specs(...) -> ToolRegistryBuilder {
    let mut builder = ToolRegistryBuilder::new();

    // ... 其他工具注册 ...

    // 注册自定义工具
    builder.push_spec(create_my_custom_tool());
    builder.register_handler("my_custom_tool", Arc::new(MyCustomHandler));

    builder
}

fn create_my_custom_tool() -> ToolSpec {
    ToolSpec::Function(FunctionTool {
        name: "my_custom_tool".into(),
        description: "描述该工具的用途".into(),
        parameters: json!({
            "type": "object",
            "properties": {
                "param1": { "type": "string" },
                "param2": { "type": "number" },
            },
            "required": ["param1"],
        }),
    })
}
```

---

## 11. 最佳实践总结

### 11.1 工具设计原则

1. **单一职责**：每个 Handler 只处理一种工具类型
   - ✅ `ShellHandler`: 处理 shell 命令
   - ✅ `McpHandler`: 处理 MCP 工具调用
   - ❌ 避免一个 Handler 处理多种不相关的工具

2. **类型安全**：使用强类型参数
   ```rust
   #[derive(Deserialize)]
   struct ShellToolCallParams {
       command: String,
       description: String,
       timeout_ms: Option<u64>,
   }
   ```

3. **清晰的错误分类**：区分 Fatal 错误和可恢复错误
   ```rust
   pub enum FunctionCallError {
       Fatal(String),           // 不可恢复，终止执行
       RespondToMessage(String), // 返回错误信息给模型
   }
   ```

### 11.2 安全最佳实践

1. **始终检查 is_mutating**：帮助编排器做出正确的权限决策
2. **沙箱逃逸处理**：正确处理沙箱拒绝错误，支持重试升级
3. **参数验证**：在 handle 中验证所有参数
4. **超时控制**：为长时间运行的操作设置超时

### 11.3 性能优化

1. **并行执行**：标记支持并行的工具
   ```rust
   builder.push_spec_with_parallel_support(spec, true);
   ```

2. **惰性初始化**：MCP 工具按需连接

3. **结果缓存**：对于昂贵的操作考虑缓存结果

### 11.4 调试与可观测性

1. **使用 tracing**：记录工具调用生命周期
2. **结构化日志**：使用 otel 记录工具指标
3. **调用追踪**：通过 call_id 追踪完整调用链

---

## 附录：核心文件结构

```
codex-rs/core/src/
├── codex.rs                    # Session 和 TurnContext 定义
│
├── tools/
│   ├── mod.rs                  # 工具模块导出
│   ├── registry.rs             # ToolRegistry 和 ToolHandler trait
│   ├── router.rs               # ToolRouter 路由逻辑
│   ├── spec.rs                 # 工具规格构建 (build_specs)
│   ├── context.rs              # ToolInvocation 上下文
│   ├── orchestrator.rs         # ToolOrchestrator 执行编排
│   │
│   └── handlers/
│       ├── mod.rs              # Handler 导出
│       ├── agent_jobs.rs       # BatchJobHandler (批处理)
│       ├── apply_patch.rs      # ApplyPatchHandler (代码补丁)
│       ├── dynamic.rs          # DynamicToolHandler (动态工具)
│       ├── grep_files.rs       # GrepFilesHandler (文件搜索)
│       ├── js_repl.rs          # JsReplHandler/JsReplResetHandler
│       ├── list_dir.rs         # ListDirHandler (目录列表)
│       ├── mcp.rs              # McpHandler (MCP 工具)
│       ├── mcp_resource.rs     # McpResourceHandler (MCP 资源)
│       ├── multi_agents.rs     # MultiAgentHandler (多 Agent)
│       ├── plan.rs             # PlanHandler (计划管理)
│       ├── presentation_artifact.rs  # PresentationArtifactHandler
│       ├── read_file.rs        # ReadFileHandler (文件读取)
│       ├── request_user_input.rs     # RequestUserInputHandler
│       ├── search_tool_bm25.rs       # SearchToolBm25Handler
│       ├── shell.rs            # ShellHandler/ShellCommandHandler
│       ├── spreadsheet_artifact.rs   # SpreadsheetArtifactHandler
│       ├── test_sync.rs        # TestSyncHandler
│       ├── unified_exec.rs     # UnifiedExecHandler
│       └── view_image.rs       # ViewImageHandler
│
├── approval.rs                 # 审批策略 (AskForApproval)
├── sandbox.rs                  # 沙箱策略 (SandboxPolicy)
│
└── mcp/
    ├── mod.rs                  # MCP 客户端管理
    └── server.rs               # MCP 服务器连接
```

---

*文档版本：1.1*
*基于：Codex main branch (2025)*
*用途：Codex Agent-Tool 架构理解参考*
*更新内容：新增原生工具参考章节（第9节），详细列出所有内置工具及其配置*
