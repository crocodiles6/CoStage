# 原生 Agent 接入说明

设计版本：v5。更新日期：2026-11-10；原生接口核对日期：2026-11-07，北京时间。本文补充 [系统设计](system-design.md) 中的原生 Agent 接入契约，描述目标行为与接口依据；当前仓库处于设计阶段。

## 两个入口，保留各自的执行方式

| 入口 | 用户提供什么 | 谁执行 Agent 循环 | 适合的使用方式 |
| --- | --- | --- | --- |
| 模型 API + 自研 Agent，managed | 云端或本地推理 API、模型和参数、平台发布的 Prompt/Skills/Tools/MCP | CoStage Harness；LangGraph 保存阶段与恢复等待 | 精细优化上下文和记忆、完整埋点、受控办公流程与保密剧情 |
| 已有 Agent，native | 已安装的 Codex、Claude Code、dsh，设备和获准配置范围 | 原生 Agent；由其协议负责会话和恢复 | 延用个人工作环境、原生能力和现有资源，通过群聊发起任务 |

本地模型 API 与本地 Agent 是两个对象。Ollama/vLLM 提供推理服务时仍走 managed；Codex/dsh 即便使用本地模型，仍是 native。平台不把原生 Agent 的输出再塞进一套外层 Agent 循环来替它决定工具调用。

## 实现语言与集成边界

Local Connector、Connector Gateway 与原生 Agent 适配器采用 TypeScript，运行在固定兼容版本的 Node.js。Codex 通过 App Server 协议接入，Claude Code 和 dsh 优先使用兼容的 TypeScript SDK；原生程序保留自身实现语言、配置、工具与持久存储。

平台的 managed Harness、Coordinator 与 LangGraph JS 同样采用 TypeScript。跨浏览器、云端、设备和 Python 服务的消息使用版本化 JSON Schema / OpenAPI 契约，并进行运行时校验。原生 SDK 的消息先在适配器内转换，不将供应商内部类型直接作为平台公共协议。

Python 按需用于独立解析、检索、评测或推理服务，不复制原生适配层或 managed 循环，不持有 Task/Run、审批和平台记忆的权威状态。服务结果由 TS 业务层验权后接纳。Connector 分发明确携带运行环境或声明 Node.js 安装要求，不假设用户的原生 Agent 安装已提供兼容环境。完整选型理由与模块分配见 [系统设计的主语言与配套技术](system-design.md#主语言与配套技术)。

## “我本地有 Codex 和 dsh”的使用过程

1. 设备所有者安装并配对 Local Connector，连接到自己的团队空间。配对只登记设备，不自动读取个人主目录。
2. 在“已有 Agent”中分别选择本机 Codex 和 dsh 的可执行文件、获准工作目录、配置来源与原生记忆使用方式。程序由设备侧探测，不能由群聊成员临时传入任意执行路径。
3. 从该运行时真实可用的模型和推理强度中选择参数，保存 RuntimeProfileRevision。原生协议支持时使用运行级覆盖；否则生成独立 profile，不修改用户平时的全局默认值。
4. 创建“我的 Codex”和“我的 dsh”两个房间 AgentInstance。默认只有所有者可以调用，结果、工具细节与临时 token 流先对本人可见。
5. 用户 @ 我的 Codex 后，平台生成 Task、native Run 和 ContextEnvelope，Connector 驱动 CoStage 专用原生会话。原有终端会话继续由用户管理。
6. 运行完成后，用户批准具体结果和受众，再发布到群聊；需要自动委派和自动分享时，选择经过隔离验证的受控协作 profile。

这里的“复用已有能力”指原生 Agent 在设备上加载获准的记忆、Skills、Tools 和 MCP。它们不需要先复制到 CoStage，但平台也不会自动获得其全部内容、版本和权限控制能力。

## 提供者适配依据

| 对象 | 推荐接入点 | 应使用的能力 | 必须核对的差异 |
| --- | --- | --- | --- |
| Codex | App Server，设备内 stdio | thread/start、thread/resume，turn/start、turn/interrupt，结构化事件、审批、model/list | 当前模型支持的 effort；Skills/MCP 配置；会话与实际执行状态；原生记忆管理暴露范围 |
| Claude Code | Claude Agent SDK，显式绑定兼容的已安装 CLI | session/resume、权限回调、hooks、模型与运行配置 | SDK/CLI 版本；配置来源；工具策略是否真正生效；允许使用的认证方式 |
| dsh | 官方 SDK Client/Server 或 stdio JSON-RPC | initialize、session/prompt、session/resume、session/wait、审批与事件 | 指定本地 dshBin；持久目录；根及后代完成语义；取消能力；通知的 session 过滤 |

Codex 官方将 App Server 用于需要认证、历史、审批和流式事件的深度集成；本设计选择设备内 stdio，避免直接暴露实验性 WebSocket 端口。[官方接口](https://learn.chatgpt.com/docs/app-server)

Claude Agent SDK 使用 Claude Code 的原生循环和工具，可指定已安装的 CLI。配置来源须显式选择，不能假设省略 settingSources 就不加载用户配置；该字段也不覆盖所有文件读取来源。第三方产品不能直接承诺复用 claude.ai 订阅登录与额度，默认采用官方允许的 API 凭据或已获准方式。[SDK 概览](https://code.claude.com/docs/en/agent-sdk/overview)、[配置与类型](https://code.claude.com/docs/en/agent-sdk/typescript)

dsh 的 session/prompt 回执表示持久入队，不表示任务已完成。session/resume 依赖原持久会话及兼容状态；session/wait 的完成条件涉及根会话和相应后代。通知是 runtime-wide，必须在设备侧过滤到获准会话后再上传。客户端超时只说明等待失败，不自动回滚原生执行。[协议](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sdk/protocol/README.md)、[客户端](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sdk/client/README.md)、[服务端](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/sdk/server/README.md)

上述资料证明接口存在；CoStage 的兼容结论仍需绑定具体程序版本、协议版本、配置和账号能力。

## 能力报告决定界面与运行约束

RuntimeCapabilities 以设备探测结果为准，带探测时间、版本、配置 digest 和有效期。每项至少区分 supported、unsupported、unknown；有能力但无权限另记 denied。配置、程序或协议变化后失效并重新校验。

| 能力组 | 需要记录 | 对产品行为的影响 |
| --- | --- | --- |
| 模型与参数 | 可用模型、参数 schema、effort 枚举、可否中途改变 | 不显示不可用值，不静默忽略或猜测跨提供者等价参数 |
| 会话 | 新建、恢复、分叉、历史读取、恢复未完 turn | 只有已验证的恢复动作可出现在界面 |
| 执行控制 | 中断、进程退出、后台子进程/子 Agent 状态、持久等待 | 决定取消确认与离线后的处理方式 |
| 原生资源 | 配置来源、Skills/MCP 目录、记忆可见和隔离范围 | 资源分 pinned、observed、unknown；未知不冒充版本快照 |
| 审批与隔离 | 执行前阻断、动作参数、工具限制、文件与网络约束 | 决定是否可启用受控协作模式 |
| 事件与成本 | 可补读事件、工具回执、逐调用 usage、费用来源 | 展示 coverage；unknown 不记零，估算单独标识 |
| 协作桥 | 可加载获准 MCP/工具、异步结果读取 | 决定能否委派群聊 Agent；不解析文本中的 @ 执行 |

“读取历史”“恢复会话”“恢复未完成动作”是不同能力。支持其中一项，不自动获得另外两项。

## 状态归属与恢复证据

| 状态 | 权威保存方 | CoStage 保存什么 |
| --- | --- | --- |
| 房间、成员、Task/Run、委派、输出受众 | 平台业务库 | 完整业务事实、授权、发布和审计 |
| managed 执行阶段 | LangGraph JS + TS 业务层操作账本 | RuntimeBinding、checkpoint 引用与稳定业务操作 ID |
| native 上下文、压缩、原生记忆和会话 | 原生运行时及其本地持久存储 | session/turn 关联、能力与资源快照；可得事件引用 |
| 命令是否已派发、原生接收回执 | 设备侧 Connector 日志与原生证据 | command_id、digest、投递状态和回执镜像 |
| 原生外部写入效果 | 原生回执或目标系统事实 | 可取得的结果与未决状态；不伪造平台 ToolInvocation 的完整账本 |

Connector 收到命令后先持久保存 accepted；发送前先保存 dispatch_intent，再向原生进程写入。dispatch_intent 后没有可核实回执时，一律进入 delivery_unknown，包括“可能尚未发出”的崩溃间隙。重投相同 command_id 只查询原记录；有可验证幂等或关联查询时才安全补齐。不能通过重新生成 prompt、新 ID 或新会话绕过未知操作。

同一 NativeSessionBinding 只有一个活动 Run。SessionScopeDigest 包含 tenant、设备所有者、请求人、凭据主体、房间、分支、实例、profile、资料授权集合和输出受众。后续任务仅在范围不变且实时授权有效时串行沿用；范围改变或分支时默认新建会话，并使用可隔离的原生记忆/profile 范围，否则拒绝复用。设备离线时保持该会话及工作目录占用，不能让云端 Worker 租约到期触发另一份原生执行。

取消先记录请求，再等待原生中断和所需后代的静止证据。不能确认时为 blocked/cancellation_unknown，继续接收审计结果但不自动发布或启动替代任务。存在未确认停止的 native 根 Run 或后代时，聚合 Task 同样保持 blocked，显示“取消中”，不能先标记 cancelled。持久等待可以释放原生进程资源，但始终保留会话的逻辑占用。

断线时 Connector 阻止新平台命令与审批，并按能力停止本地运行；已经发出的外部请求仍要单独核对。

## 数据和工具权限

个人原生模式默认只由本人调用。继承已有记忆和身份意味着来源可能超出当前房间；原生结果先保持私密，所有者审核具体内容后才可分享。受控协作模式需要独立且经过验证的 profile、工作目录、记忆与工具范围，不能只换 session ID 就声称完成隔离。

工作目录登记须解析真实路径和符号链接，校验附件输入与产物导出路径。运行时仍需 OS 权限或沙箱限制实际文件、网络与进程访问；路径白名单只约束 Connector 的操作，不等于限制了原生工具。无法限制的能力明确显示，受控协作模式不得启用。

原生审批保存 provider request ID、原生会话、Run、动作参数 digest、授权主体、lease epoch、有效期。批准时再次校验，过期请求不能复用。分享批准单独绑定结果 revision 和受众，不能代替执行前工具审批。

平台 Tool Gateway 只治理经过它的工具。Codex/Claude Code/dsh 的内置工具、hooks、后台任务和直接 MCP 受原生系统控制。只有确实具备阻断和审计能力时才标记平台可治理；未知动作范围不得自动批准。

## 同一群聊中的跨运行时协作

managed 与 native 共用 Task、委派 ID、预算归属和结果受众。native 可通过获准的 CoStage 协作 MCP 调用 delegate_task、task_status、task_result；原生内部 subagent 保留为原生 trace 子项，不自动成为可 @ 的房间成员。

managed 父 Run 可持久等待 native 子 Run，之后由 LangGraph 恢复。native 父 Run 继续由原生协议等待或异步查询，平台不向它发送 LangGraph 恢复命令。原生父 turn 已结束但必需子 Run 尚未完成时，父 Run 保持 waiting_children。结果齐备后，Coordinator 按持久 wait_generation 和结果 revision 生成稳定的新 command_id，向原会话提交后续 turn，继续同一父 Run。该模式要求支持可关联的串行后续输入；能力不足时拒绝需要异步续接的必需委派，只允许已验证同步等待或无需父续接的可选委派。

跨任务委派还检查原生会话和工作目录的占用依赖，拒绝导致等待环路的请求。

同一个 Run 不在两个后端之间迁移。将任务交给另一类 Agent 是显式创建子 Run；在故障后重新执行则创建关联的新 Run，并重新确认原执行状态和副作用。

