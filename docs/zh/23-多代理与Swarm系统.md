# 多代理与 Swarm 系统

## 技术分析

Claude Code 内部实现了一套被称为 “swarm” 的多代理协作架构。它允许一个 leader 会话在同一任务中生成多个 teammate，让这些代理并行工作、相互通信，并由 leader 统一处理权限请求、状态跟踪和 UI 协调。

这套系统横跨：

- `src/utils/swarm/`
- `src/tasks/`
- `src/tools/AgentTool/`

它不只是“多开几个 agent”，而是包含了执行后端选择、团队状态持久化、文件邮箱通信、权限桥接、pane 布局和会话恢复的完整运行时。

## 1. 团队拓扑与 TeamFile

Swarm 采用 leader-worker 拓扑：

- 一个实例担任 team lead
- 多个实例担任 teammate

团队状态持久化在 `~/.claude/teams/{team-name}/config.json`，这里的 `TeamFile` 是系统级事实来源，记录：

- 团队名、描述、创建时间
- `leadAgentId`
- 隐藏 pane 列表
- team 级 allowed paths
- 成员列表

成员项里又包含：

- `agentId`
- `name`
- `agentType`
- `model`
- `tmuxPaneId`
- `cwd`
- `worktreePath`
- `sessionId`
- `backendType`
- `isActive`
- `mode`

可以看出，TeamFile 不只是静态配置，更承担运行态控制面的角色。

## 2. 执行后端选择

Swarm 支持三类后端：

- `tmux`
- `iterm2`
- `in-process`

后端检测遵循一条优先级链：

1. 如果当前就在 tmux 内，优先 tmux
2. 如果在 iTerm2 且 `it2` CLI 可用，优先 iTerm2
3. 若在 iTerm2 但 `it2` 不可用，则考虑 tmux
4. 若系统可用 tmux，则启用外部 tmux session
5. 非交互会话强制走 in-process
6. 没有终端多路复用器时退回 in-process

`isInProcessEnabled()` 还会综合：

- `--teammate-mode`
- 当前是否非交互
- 是否已触发 fallback
- 当前是否位于 tmux / iTerm2

一旦 pane backend 首次创建失败，系统会打上 in-process fallback 标记，后续所有 teammate 都直接走进程内模式，避免反复试错。

## 3. In-Process Teammate

in-process teammate 与 leader 共处同一 Node.js 进程，但通过 `AsyncLocalStorage` 做上下文隔离。每个 teammate 都拥有独立的：

- 身份
- 团队上下文
- 任务状态
- `AbortController`

`spawnInProcessTeammate()` 在启动时会：

- 生成 `agentId`
- 创建 task state
- 注册 cleanup handler
- 在开启 tracing 时接入 Perfetto
- 启动后台 agent 循环

这里有两个非常关键的资源约束设计。

第一，系统给 teammate 在 UI 中镜像的消息数量设置了上限，避免大量 agent 爆炸性生成时造成内存失控。

第二，启动 in-process teammate 时会显式去掉父对话消息，避免整段父线程上下文长期被 pin 进每个 teammate 的内存中。

这表明 swarm teammate 与“继承完整上下文的 fork 子代理”是两种完全不同的执行模型。

## 4. Pane-Based Teammate

pane-based teammate 运行在独立 OS 进程和独立 pane 中，由 `PaneBackendExecutor` 统一适配 tmux / iTerm2。

其流程包括：

- 创建 pane
- 构造 CLI 启动命令
- 传递身份、team 信息、环境变量和继承参数

`buildInheritedCliFlags()` 会传播如下关键配置：

- `--model`
- `--settings`
- `--plugin-dir`
- `--teammate-mode`
- 权限模式

其中一个安全约束是：如果 teammate 被要求运行在 plan mode，则不会继承 bypass 权限。

`buildInheritedEnvVars()` 还会显式转发：

- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- 代理与证书变量
- Claude Code Remote 标记

这是因为 tmux 打开的新 shell 不一定自动继承父环境。

## 5. Fork Subagent 与 Swarm Teammate 的区别

Swarm teammate 与 fork subagent 很容易被混为一谈，但两者的语义完全不同。

Swarm teammate 的特点是：

- 独立 prompt
- 团队身份
- 通过 mailbox 显式通信
- 面向协作分工

Fork subagent 的特点是：

- 继承父线程完整上下文
- 继承渲染后的 system prompt bytes
- 面向 prompt cache 对齐

fork 机制还会检查 `FORK_BOILERPLATE_TAG`，阻止递归 fork。

## 6. 权限同步

Swarm 中最核心的安全原则之一是：权限尽量集中在 leader 侧处理。

经典权限同步协议基于文件系统：

1. worker 遇到需要授权的工具调用
2. 写入 `permissions/pending/`
3. leader 轮询读取并展示给用户
4. 用户批准或拒绝
5. leader 把结果写入 `permissions/resolved/`
6. worker 轮询结果

目录级操作带 lockfile，防止并发竞争。

同时，系统也支持 mailbox 版本的权限路由：

- 普通工具权限请求
- 沙箱网络 host 权限请求

TeamFile 中的 `teamAllowedPaths` 则提供了另一层批量授权能力。teammate 初始化时会把这些路径转成 session 级规则，写入权限上下文，使整个团队在指定路径上共享放行策略。

## 7. Leader Permission Bridge

对于 in-process teammate，最佳体验不是写文件再轮询，而是直接复用 leader 原生的 ToolUseConfirm UI。

`leaderPermissionBridge.ts` 对外提供两类注册入口：

- leader 的确认队列 setter
- leader 的权限上下文 setter

当 in-process teammate 触发 `ask` 权限时，`createInProcessCanUseTool()` 会优先检查 bridge：

- 有 bridge：直接把请求塞进 leader 的原生确认队列
- 没有 bridge：退回 mailbox 协议

UI 层会在权限提示里显示带颜色的 worker badge，让用户知道请求来自哪个 teammate。

实现中还刻意加了 `preserveMode: true`，避免 worker 的权限模式回写时污染 leader 的共享权限状态。

对 Bash 命令，系统还会先尝试 classifier 自动批准，再决定是否真正升级为 leader 审批。

## 8. 邮箱通信与空闲通知

Swarm 中的跨代理通信通过文件邮箱系统完成。支持的消息类型包括：

- 直接消息
- 权限请求 / 响应
- 沙箱权限请求 / 响应
- 关停请求
- 空闲通知

每个 teammate 启动时还会收到一段固定 system prompt addendum，明确告知：

- 直接输出文本不会被其他 teammate 看见
- 必须使用 `SendMessage` 工具通信
- 广播只能谨慎使用

这相当于把跨代理通信从“隐式共享上下文”变成“显式工具调用”。

当 teammate 完成工作后，`Stop` hook 会：

- 把成员标记为 `inactive`
- 生成 idle notification
- 把最近 peer DM 摘要发给 leader

leader 因而可以把 worker 完成情况纳入自己的调度视野。

## 9. 任务类型

Swarm 与任务框架结合得很深，主要涉及三类任务：

### 9.1 InProcessTeammateTask

它保存：

- 身份与团队字段
- 当前 prompt / model
- `abortController`
- `pendingUserMessages`
- 截断后的 `messages`
- `onIdleCallbacks`
- `shutdownRequested`

并支持：

- `kill()`
- `requestTeammateShutdown()`
- `injectUserMessageToTeammate()`

### 9.2 LocalAgentTask

这是更传统的后台 agent，不具备 swarm 团队语义，但同样运行在本地进程内。它更关注：

- 工具调用计数
- token 计数
- 活动轨迹
- worktree 隔离

### 9.3 RemoteAgentTask

这是运行在 Claude Code Remote 基础设施上的远程任务，额外跟踪：

- 远程 `sessionId`
- `todoList`
- SDK 消息日志
- 长运行模式
- review 进度

这些类型最后都汇入统一的 `TaskState` 联合类型，被背景任务 UI 和状态系统统一管理。

## 10. 重连与清理

Swarm 明确处理两种恢复场景。

第一类是新启动会话。`computeInitialTeamContext()` 会在首屏渲染前读取 CLI 注入的动态 team context，并立刻构造 AppState 所需的 `teamContext`，避免后续再异步补齐导致竞争条件。

第二类是恢复旧 transcript。`initializeTeammateContextFromSession()` 会从会话记录中恢复 `teamName` 与 `agentName`，然后再回到 TeamFile 里找出 `agentId`。

崩溃清理方面，系统会记录当前 session 创建过哪些团队。若发生非优雅退出，`cleanupSessionTeams()` 会：

1. 杀掉遗留 pane
2. 删除 git worktree
3. 清理 team 目录
4. 清理 task 目录

对 in-process teammate，`killInProcessTeammate()` 会：

- abort 正在运行的 controller
- 解除 cleanup 注册
- 唤醒等待中的 idle callbacks
- 从 teamContext 移除成员
- 清理运行时引用，降低泄漏风险

## 11. 布局与 UI 协调

`teammateLayoutManager` 负责多代理输出的可视化组织，包括：

- 颜色分配
- pane 创建
- pane border status

tmux backend 采用较明确的布局策略：

- leader 位于左侧
- teammate 位于右侧

为避免并发创建 pane 造成布局冲突，`TmuxBackend` 还实现了串行化锁，并在每次 pane 创建后等待一小段 shell 初始化时间，再发送命令。

TeamFile 中的 `hiddenPaneIds` 用来跟踪被暂时移出可视布局的 pane。

对 in-process teammate，UI 侧对应的能力包括：

- task pills
- zoom 进入 teammate transcript
- 直接向 teammate 注入用户消息
- 权限提示上的 worker badge

模型选择方面，teammate 会回退到 provider 对应的最新 Opus 模型，也可以被单个 spawn 配置或全局设置覆盖。
