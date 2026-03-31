# IDE 与 LSP 集成系统

## 概览

Claude Code 在开发环境集成上采用双轨结构：

1. LSP 子系统
2. IDE 扩展桥接

前者让 Claude Code 作为 LSP client，直接启动和管理语言服务器，获取类型信息、诊断、跳转与符号能力；后者则让它发现正在运行的 IDE 实例，通过 MCP 与扩展通信，实现文件同步、diff 展示、编辑器导航等能力。

这两套系统互相独立，但组合起来形成了一个“代码修改 -> 语言反馈 -> 编辑器联动”的闭环。

## 1. LSP Client 设计

`LSPClient` 没有使用 class，而是采用工厂函数 + closure 模式，把状态放在局部变量中封装。它内部持有：

- `process`
- `connection`
- `capabilities`
- `isInitialized`
- `isStopping`

底层基于 `vscode-jsonrpc`，通过 stdio 与语言服务器建立 JSON-RPC 通道。

一个值得注意的设计点是“延迟 handler 注册”。即便连接还没建立，上层也可以先注册 notification / request handler，这些 handler 会先进入 pending 队列，等 `start()` 真正完成后再统一挂载。

`stop()` 则严格遵守 LSP 协议顺序：

1. 发送 `shutdown`
2. 发送 `exit`
3. 清理 connection 与 child process

清理逻辑写在 `finally` 中，说明实现目标是“无论停机过程中哪一步失败，都不能遗留资源”。

## 2. Server 生命周期

LSP 生命周期管理分三层：

1. `LSPClient`：原始 JSON-RPC 连接
2. `LSPServerInstance`：单个 server 的状态机、重试与健康状态
3. `LSPServerManager`：多 server 编排、扩展名路由、文档同步

最外层的 `manager.ts` 则负责单例化与重初始化。

### 2.1 状态机

`LSPServerInstance` 的状态转换非常清晰：

- `stopped -> starting -> running`
- `running -> stopping -> stopped`
- 任意状态 -> `error`
- `error -> starting`

这种明确状态划分使得“启动失败”“运行中崩溃”“主动关闭”不会混在一起。

### 2.2 Lazy Load

`LSPServerInstance` 通过 `require()` 动态加载 `LSPClient`，而不是启动时静态 import 整个 JSON-RPC 依赖链。这样做是为了避免在用户根本没有配置 LSP 时拖慢 Claude Code 启动。

### 2.3 初始化参数

server 启动时会发送比较完整的 `InitializeParams`，包括：

- `workspaceFolders`
- `rootPath` / `rootUri`
- diagnostics、hover、definition、references、documentSymbol、call hierarchy 等 client capabilities
- `UTF-16` position encoding
- 插件提供的 `initializationOptions`

这让 Claude Code 能同时兼容较新和较旧实现的语言服务器。

### 2.4 瞬态错误重试

请求遇到 `-32801`（content modified）时会做指数退避重试。这类错误常见于诸如 `rust-analyzer` 这类在索引阶段频繁变化的 server。

### 2.5 崩溃恢复上限

server 崩溃后可以自动拉起，但存在 `maxRestarts` 上限。这个限制直接防止了“一个永远崩溃的 server 不断拉起新进程”的资源失控问题。

### 2.6 全局单例与 generation 保护

`manager.ts` 使用 generation counter 防止旧的异步初始化结果覆盖新的初始化状态。这种模式是因为 `reinitializeLspServerManager()` 可能在前一次初始化还未结束时再次触发。

bare mode 下则完全跳过 LSP 初始化，因为脚本式调用并不需要编辑器增强。

## 3. Diagnostic Registry

`LSPDiagnosticRegistry` 负责把语言服务器发来的 diagnostics 变成下一轮会话可用的 attachment。

处理链路是：

1. server 发 `publishDiagnostics`
2. 注册 pending diagnostic
3. 去重、限流
4. 转换成 attachment
5. 在下一轮会话中交给 agent

### 3.1 两层去重

系统做了两层去重：

- 批次内去重
- 跨轮次去重

跨轮次去重基于 `LRUCache`，最多保留 500 个文件的投递记录。每个 diagnostic 的 key 由消息、严重度、range、source、code 等字段序列化得到。

当文件发生编辑时，`clearDeliveredDiagnosticsForFile()` 会清空该文件已投递记录，使新的同类诊断仍能再次进入对话。

### 3.2 限流

为避免 diagnostics 洪水淹没上下文，系统设置了硬上限：

- 每个文件最多 10 条
- 总量最多 30 条

裁剪前先按严重程度排序，确保 Error 优先保留。

### 3.3 Passive Feedback

`passiveFeedback.ts` 负责给所有 server 注册 `publishDiagnostics` 监听器，并把 LSP severity 数字映射成 Claude Code 内部格式。某个 server 连续失败时会告警，但不会拖垮整个子系统。

## 4. IDE 检测

Claude Code 支持两大家族 IDE：

- VS Code family：VS Code、Cursor、Windsurf
- JetBrains family：IntelliJ、PyCharm、WebStorm、GoLand、Rider 等

每个 IDE 都维护平台相关的进程关键字。某些 JetBrains 产品在特定平台上故意不做自动检测，因为进程名太通用，误判成本高。

### 4.1 Lockfile 发现

主要发现机制并不是窗口枚举，而是读取 `~/.claude/ide/` 下的 lockfile。每个 lockfile 里包含：

- `workspaceFolders`
- `pid`
- `ideName`
- `transport`
- `runningInWindows`
- `authToken`

系统会并行读取这些 lockfile，按修改时间排序，再根据当前工作目录筛选最相关的 IDE 实例。路径匹配还会做 Unicode NFC 归一化，以兼容 macOS 的 NFD 表示。

### 4.2 进程祖先校验

若 Claude Code 跑在受支持 IDE 的内置终端中，系统会沿进程树向上检查祖先 PID，用来区分多个同类 IDE 实例同时打开相同工作区的情况。

这个检查放在 workspace 匹配之后，避免每个 lockfile 都先做进程树追踪带来的性能开销。

### 4.3 集成终端识别

`isSupportedTerminal()` 通过 `TERM_PROGRAM` 或 JetBrains 动态环境识别当前是否位于 IDE 集成终端。tmux / screen 会覆盖 `TERM_PROGRAM`，但 `CLAUDE_CODE_SSE_PORT` 一类变量仍可能被继承，因此自动连接逻辑仍有兜底。

### 4.4 过期 Lockfile 清理

连接前会先清理 stale lockfile：进程已死或端口不响应的项会被删除。端口检测用的是 500ms 超时的原始 TCP 连接。

### 4.5 WSL

WSL 支持是单独处理的复杂分支，涉及：

- Windows `USERPROFILE`
- Windows 路径到 WSL 路径转换
- distro 匹配
- 主机 IP 解析

这说明 IDE 集成从一开始就不是“只面向本机 Unix”的实现。

## 5. IDE 扩展通信

### 5.1 MCP 传输

IDE 扩展与 Claude Code 的通信走 MCP。`useIDEIntegration.tsx` 会把检测到的 IDE 注入为一个动态 MCP server 配置。

支持两类传输：

- SSE
- WebSocket

具体走哪种传输，由 lockfile 的 `transport` 字段决定。

### 5.2 自动连接

自动连接在以下任一条件满足时触发：

- 开启 `autoConnectIde`
- 传入 `--ide`
- 当前位于受支持 IDE 的集成终端
- 存在 `CLAUDE_CODE_SSE_PORT`
- 用户显式请求安装 IDE 扩展
- `CLAUDE_CODE_AUTO_CONNECT_IDE` 为真

如果 `CLAUDE_CODE_AUTO_CONNECT_IDE` 被显式设为 falsy，则会压过自动连接。

### 5.3 IDE RPC

连上后，Claude Code 通过 `callIdeRpc()` 调用 IDE 侧工具，本质仍然是 MCP `callTool` 的封装。典型能力包括：

- 打开文件
- 显示 diff
- 编辑器导航
- UI 联动

调用前会确认 IDE MCP server 当前处于 `connected` 状态。

### 5.4 扩展安装

VS Code 家族支持通过 CLI 自动安装扩展，例如：

- `code --install-extension`
- `cursor --install-extension`

扩展 ID 是 `anthropic.claude-code`。系统还会比较 bundled version 与本机已安装版本，必要时升级。

JetBrains 家族则不走 CLI 自动安装，而是引导用户去 marketplace 获取。

`/ide` 命令提供手动连接 UI，用于处理多个 IDE 实例并存时的工作区选择问题。

## 6. 文件同步

### 6.1 文档生命周期

`LSPServerManager` 实现了完整文档同步流程：

- `openFile()`
- `changeFile()`
- `saveFile()`
- `closeFile()`

内部的 `openedFiles` map 记录某个文件当前挂在哪个 server 上，保证 `didOpen` 必须先于 `didChange` 这一 LSP 协议约束不被破坏。

### 6.2 与文件工具联动

`FileWriteTool` / `FileEditTool` 改文件后，会做三件事：

1. 清空该文件已投递 diagnostics
2. 发 `didChange`
3. 发 `didSave`

LSP 调用采用 fire-and-forget 方式，错误只记录日志，不阻塞主工具响应。这体现出 LSP 在整体架构中的定位是增强层，而不是核心依赖。

### 6.3 扩展名到 Server 的路由

`LSPServerManager` 通过 `extensionMap` 把文件扩展名映射到对应 server。来源是插件定义的 `extensionToLanguage` 配置。若多个 server 同时声明支持某扩展名，目前采用“先注册者优先”。

### 6.4 IDE 侧文件通知

除了同步给语言服务器，文件变化还会通知 IDE 扩展，使：

- 文件树刷新
- diff 视图同步
- 编辑器内容状态更新

因此 Claude Code 实际维护了两条并行同步链路：

- 面向语言语义的 LSP
- 面向编辑器界面的 IDE bridge

## 7. 上下文增强

### 7.1 LSPTool

`LSPTool` 直接把语言服务器能力暴露给 agent，支持：

- `goToDefinition`
- `findReferences`
- `hover`
- `documentSymbol`
- `workspaceSymbol`
- `goToImplementation`
- `prepareCallHierarchy`
- `incomingCalls`
- `outgoingCalls`

工具是否启用由 `isLspConnected()` 决定，只要 manager 中至少有一个 server 不是错误态，就视为 LSP 可用。

### 7.2 诊断驱动反馈闭环

LSP 集成最有价值的部分并不只是查询能力，而是被动反馈闭环：

1. agent 修改文件
2. Claude Code 把改动同步给 LSP
3. 语言服务器重新分析
4. diagnostics 被注册
5. 下一轮会话自动作为 attachment 进入上下文
6. agent 看到错误和警告后继续修正

整个过程不要求 agent 显式调用“检查类型错误”命令。

### 7.3 插件驱动配置

LSP server 配置完全来自插件，而不是用户 / 项目 settings。插件可以通过 `.lsp.json` 或 manifest 提供：

- `command`
- `args`
- `extensionToLanguage`
- `initializationOptions`
- `startupTimeout`
- `maxRestarts`
- `workspaceFolder`

系统会用 Zod 校验这些配置，并限制 server 二进制路径必须留在插件目录内，防止路径穿越。

### 7.4 插件刷新后的重初始化

曾经存在一个边界问题：插件加载过早 memoize，导致 marketplace plugin 还未同步时就缓存了空配置，而 LSP manager 又不像 commands / hooks / MCP 一样在插件刷新后自动重建。

后续修复让 `reinitializeLspServerManager()` 在插件刷新时明确销毁旧 manager 并重新初始化。
