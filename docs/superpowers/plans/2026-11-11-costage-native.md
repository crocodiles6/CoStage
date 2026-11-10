# CoStage 原生 Agent 接入 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让本地 Codex、Claude Code 和 dsh 在明确授权下成为可被 @ 的 Agent。

**Architecture:** 云端共用 Task、授权与审批；设备 Connector 保存命令日志并驱动已安装程序。原生运行时拥有执行恢复，平台只根据可验证证据发布结果或继续等待。

**Tech Stack:** TypeScript、Node.js 24 LTS、React、Fastify、LangGraph JS、PostgreSQL/pgvector、对象存储、Vitest、Playwright。Python 只作为按需服务。

**Spec:** [系统设计 v5](../../system-design.md)、[原生接入 v5](../../native-agent-integration.md)。执行前同时阅读[计划总入口](2026-11-11-costage.md)，其中定义公共类型、测试约定和依赖图。

## Global Constraints

- TypeScript 主栈；Python 按需独立服务，TS 业务层拥有任务、授权和平台记忆事实。
- 邀请制、多团队独立空间；所有业务表带 `tenant_id`，授权身份来自可信登录。
- 默认每房间 20 名参与者、8 个 Agent、4 个同时执行片段。
- 默认最大委派深度 3、每次扇出 3、每个 Task 最多 12 个 Run 与 24 次模型调用，所有重试共用预算。
- 普通消息不触发执行；结构化 @ 创建 Task，Agent 委派只能通过 `delegate_task`。
- managed 使用自研循环与策略 + LangGraph JS；native 由原生运行时恢复，同一 Run 不跨后端恢复。
- 个人原生模式默认仅本人调用，结果私密后批准分享；严格剧本使用 managed。
- 仅文本与文件；不加入音视频、公开技能市场或跨空间 Agent 网络。

## Review Focus

- N03：dispatch_intent 已存但原生回执丢失，未知投递不得自动重发。
- N02/N07：同房间换调用人或凭据主体，不能沿用含前人记忆的会话。
- N06：dsh runtime-wide 通知包含其他会话，必须在设备侧过滤。
- N07：断网和取消未确认，不能在其他设备复制执行或显示已停止。
- N08：原生父 turn 已完、子任务晚到，只能幂等续接同一 Run。

---

## 文件与执行顺序

`apps/connector-gateway` 维护云端通道；`apps/connector` 维护设备身份、SQLite 日志、事件补传及本地占用；`runtime-native/providers` 每个文件只适配一个协议。云端不持有原生密钥，也不通过 LangGraph 重放原生 prompt。

N01→N02→N03 后，N04、N05、N06 可以基于相同 ProviderPort 分别完成录制协议测试；三者汇合到 N07。N08 同时依赖 managed 的等待恢复，必须在两条路径独立通过后再联调。新增提供者的完成标准包含版本矩阵与一次获准本机冒烟，不能仅凭启动命令成功声明完整兼容。

<a id="n01"></a>

### Task N01: 设备配对与受限双向通道

**Depends on:** F02, F06

**Files:**
- Create: `apps/connector-gateway/package.json`
- Create: `apps/connector-gateway/src/server.ts`
- Create: `apps/connector/package.json`
- Create: `apps/connector/src/cli.ts`
- Create: `apps/connector/src/transport.ts`
- Create: `packages/runtime-native/package.json`
- Create: `packages/runtime-native/src/devices.ts`
- Create: `packages/contracts/src/connector.ts`
- Create: `packages/infra/migrations/014-devices.sql`
- Create: `apps/api/src/routes/connectors.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/connector-transport.test.ts`

**Interfaces:**

- **Consumes:** `authenticate(jwt: string): Promise<Actor>`（F02）；`authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）；`guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`（F06）。
- **Produces:** `createPairing(actor: Actor): Promise<{pairingId: Id; expiresAt: ISODate; code: string}>`；`completePairing(code: string, devicePublicKey: string): Promise<{deviceId: Id; credential: string}>`；`revokeDevice(actor: Actor, deviceId: Id): Promise<void>`；`sendCommand(lease: Lease, command: NativeCommand): Promise<{state: "queued" | "delivered"}>`；`acceptDeviceEvent(deviceId: Id, event: NativeEvent): Promise<{ackSeq: number}>`。NativeCommand/NativeEvent 见总入口。

- [ ] **Step 1: 编写失败测试。** 同配对码重复使用、不同 tenant 设备凭据、过期授权、撤销后旧 WSS、任意 shell/env/path/RPC 字段以及断线重连重复事件分别建例。

```ts
it("revoked or foreign device cannot dispatch", () => {
  expect(revokedDispatch.code).toBe("DEVICE_REVOKED");
  expect(arbitraryRpc.code).toBe("INVALID_COMMAND");
  expect(crossTenantEvents).toEqual([]);
});
it("ordered reconnect deduplicates events", () => {
  expect(committedEventIds).toEqual(["e1","e2","e3"]);
  expect(ackSeq).toBe(3);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/connector-transport.test.ts`。预期 FAIL，原因应为旧设备继续执行、通道可调用任意 RPC 或补传产生重复业务事件；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** gateway/server.ts 仅接受设备主动认证 WSS，设备凭据绑定 owner/tenant，凭据只下发一次并在设备凭据存储中保存；配对默认 10 分钟、单次使用，属于实现默认值。connector/transport.ts 心跳和 seq/ack 补传，协议允许列表只有注册的运行/查询/取消/审批操作。CLI 提供 pair/register/start/doctor，注册可执行文件须本地所有者确认，远端不接受自由路径。发送成功只表示传输状态，不表示原生执行完成。根 app 路由接入设计中的 pairings/revoke/capabilities。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/connector-transport.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add apps/connector-gateway/package.json apps/connector-gateway/src/server.ts apps/connector/package.json apps/connector/src/cli.ts apps/connector/src/transport.ts packages/runtime-native/package.json packages/runtime-native/src/devices.ts packages/contracts/src/connector.ts packages/infra/migrations/014-devices.sql apps/api/src/routes/connectors.ts apps/api/src/app.ts tests/integration/connector-transport.test.ts
git commit -m "feat: n01 设备配对与受限双向通道"
```

<a id="n02"></a>

### Task N02: 能力配置与会话安全范围

**Depends on:** F03, N01

**Files:**
- Create: `packages/runtime-native/src/profiles.ts`
- Create: `packages/runtime-native/src/session-scope.ts`
- Create: `packages/runtime-native/src/provider-port.ts`
- Create: `packages/contracts/src/native.ts`
- Create: `packages/infra/migrations/015-native-profiles.sql`
- Create: `apps/api/src/routes/runtime-profiles.ts`
- Modify: `apps/api/src/app.ts`
- Modify: `packages/core/src/releases/manifest.ts`
- Test: `tests/integration/native-profile.test.ts`

**Interfaces:**

- **Consumes:** `authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）；`saveResource(actor: Actor, input: {kind: ResourceKind; content: BlobRef; schemaVersion: string; replaces?: RevisionRef}): Promise<RevisionRef>`（F03）。
- **Produces:** `publishRuntimeProfile(actor: Actor, draft: RuntimeProfileDraft): Promise<RevisionRef>`；`computeSessionScope(input: SessionScope): Digest`；`validateRuntimeConfig(profile: RuntimeProfileRevision, capabilities: CapabilitySnapshot): void`；`ProviderPort` 的 probe/start/query/interrupt/decide 方法签名见总入口，由 N04–N06 分别实现。

- [ ] **Step 1: 编写失败测试。** 同一实例先后由不同 requester、credentialSubject、branch、资料集合和受众调用；能力包含 supported、unsupported、unknown，授权另有 granted/denied。模拟 profile 资源变化及只换 session 却共用全局记忆。

```ts
it("identity or data scope changes require isolated session scope", () => {
  expect(scopeOwnerA).not.toBe(scopeOwnerB);
  expect(scopeRoomAudience).not.toBe(scopePrivateAudience);
  expect(unisolatedControlledProfile.code).toBe("ISOLATION_UNVERIFIED");
});
it("provider parameters are validated without global mutation", () => {
  expect(unsupportedEffort.code).toBe("CAPABILITY_UNSUPPORTED");
  expect(staleCapability.code).toBe("CAPABILITY_STALE");
  expect(globalConfigAfter).toEqual(globalConfigBefore);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/native-profile.test.ts`。预期 FAIL，原因应为会话复用串身份、unknown 被当支持或只换 session 就启用受控协作；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** profiles.ts 保存独立 revision 与 schema 化的模型/effort/configSources/memory/tool/MCP 参数；默认个人模式、owner-only、结果私密。受控协作模式必须绑定实际隔离证据，未通过 O03 的约束组合不允许自动发布。session-scope.ts 的摘要包含 tenant、deviceOwner、requester、credentialSubject、room、branch、instance、profileScope、dataGrants、audience；授权变更仍实时检查。只探测选定配置范围，原生资源标 pinned/observed/unknown。明确 executable/runtime/profile 版本，漂移要求重新验证，不改全局配置。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/native-profile.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-native/src/profiles.ts packages/runtime-native/src/session-scope.ts packages/runtime-native/src/provider-port.ts packages/contracts/src/native.ts packages/infra/migrations/015-native-profiles.sql apps/api/src/routes/runtime-profiles.ts apps/api/src/app.ts packages/core/src/releases/manifest.ts tests/integration/native-profile.test.ts
git commit -m "feat: n02 能力配置与会话安全范围"
```

<a id="n03"></a>

### Task N03: 设备命令日志与单次投递

**Depends on:** N02

**Files:**
- Create: `apps/connector/src/journal.ts`
- Create: `apps/connector/src/dispatcher.ts`
- Create: `apps/connector/src/event-buffer.ts`
- Create: `apps/connector/src/local-locks.ts`
- Modify: `apps/connector/src/cli.ts`
- Modify: `apps/connector/src/transport.ts`
- Test: `tests/integration/native-delivery.test.ts`

**Interfaces:**

- **Consumes:** `ProviderPort`（N02）；N01 NativeCommand、NativeEvent 和设备通道。
- **Produces:** `acceptCommand(command: NativeCommand): Promise<CommandRecord>`；`dispatchCommand(commandId: Id, provider: ProviderPort): Promise<CommandRecord>`；`appendLocalEvent(event: NativeEvent): Promise<number>`；`readUnacked(afterSeq: number): Promise<NativeEvent[]>`；`lockNativeResources(scopeDigest: Digest, sessionId: Id, workdirId: Id): Promise<void>`。CommandRecord 含 command、deliveryState、nativeReceipt、lastEventSeq；状态严格区分 accepted/dispatch_intent/receipt/delivery_unknown。

- [ ] **Step 1: 编写失败测试。** 在 accepted 后、dispatch_intent 后但 IO 前、原生收到后但本地回执前、回执持久后分别杀死 Connector；同 commandId 换 digest 和原样重投分别测试。

```ts
it("dispatch intent without receipt never blindly resends", () => {
  expect(postIntentRestart.deliveryState).toBe("delivery_unknown");
  expect(nativePromptWrites).toBe(1);
  expect(sameIdChangedInput.code).toBe("COMMAND_CONFLICT");
});
it("accepted only may dispatch after renewed authorization", () => {
  expect(acceptedOnlyResume.nativePromptWrites).toBe(1);
  expect(acknowledgedEventsAfterRestart).toEqual(originalEventIds);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/native-delivery.test.ts`。预期 FAIL，原因应为崩溃后重发可能已执行的 prompt、换 digest 覆盖记录或丢失事件；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** journal.ts 使用设备本地 SQLite WAL 与 FULL 同步持久化命令、事件和资源占用；日志目录仅设备所有者可访问。dispatcher.ts 先 accepted、再事务提交 dispatch_intent、最后向原生 IO；重投返回已有记录，未知投递只有 ProviderPort.query 找到可关联证据才补回执。local-locks.ts 在设备进程重启后保留待核对 session/workdir 占用，不能只用内存 mutex。event-buffer.ts 先落地再上传，收到 ack 才清理已确认缓冲。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/native-delivery.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add apps/connector/src/journal.ts apps/connector/src/dispatcher.ts apps/connector/src/event-buffer.ts apps/connector/src/local-locks.ts apps/connector/src/cli.ts apps/connector/src/transport.ts tests/integration/native-delivery.test.ts
git commit -m "feat: n03 设备命令日志与单次投递"
```

<a id="n04"></a>

### Task N04: Codex App Server 适配

**Depends on:** N03

**Files:**
- Create: `packages/runtime-native/src/providers/codex.ts`
- Create: `tests/fixtures/native/codex-transcript.json`
- Create: `docs/providers/codex.md`

- Test: `tests/integration/codex-adapter.test.ts`

**Interfaces:**

- **Consumes:** `ProviderPort`、`NativeStart`、`NativeLocator`、`NativeObservation`（N02）；N03 dispatcher 负责设备投递。
- **Produces:** `createCodexProvider(registration: LocalRegistration): ProviderPort`。适配器只映射 App Server 的结构化协议，不引入另一套循环。

- [ ] **Step 1: 编写失败测试。** 录制固定兼容版本的 initialize、model/list、thread/turn、工具审批、turn 结束与 interrupt 事件；模型列表对 effort 给出不同选项。stdout 混入诊断行须显式协议错误或按官方通道处理，不能当 assistant 文本。

```ts
it("codex capabilities drive configuration", () => {
  expect(selectableEfforts).toEqual(modelListEfforts);
  expect(receivedModel).toBe(selectedModel);
  expect(selectedProfileResourceRefs).toEqual(approvedResourceRefs);
});
it("receipt and interrupt are not completion evidence", () => {
  expect(afterTurnReceipt.executionState).toBe("running");
  expect(afterConfirmedInterrupt.executionState).toBe("quiescent");
  expect(globalConfigAfter).toEqual(globalConfigBefore);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/codex-adapter.test.ts`。预期 FAIL，原因应为固定 effort 被硬编码、receipt 被判完成或全局设置被修改；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** codex.ts 通过本机 stdio App Server 创建 CoStage 专用 thread/turn，使用 model/list 与审批事件确认实际能力。profile overlay 只写本任务专用目录，保留已批准的 Skills/MCP/记忆配置来源；无法细分的资源标 observed/unknown。映射 native turn ID、工具审批 ID、usage 和完成证据，读取历史不等于可续接。文档记载固定的 CLI/App Server 版本、资源覆盖和最小本机验证步骤；测试夹具脱敏且不含凭据。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/codex-adapter.test.ts` 和 `pnpm typecheck`，预期全部 PASS。离线协议测试为必需项；真实本机验证须由执行时已有的设备授权和兼容 Codex 完成，结果单列，不能用录制桩替代。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-native/src/providers/codex.ts tests/fixtures/native/codex-transcript.json docs/providers/codex.md tests/integration/codex-adapter.test.ts
git commit -m "feat: n04 Codex App Server 适配"
```

<a id="n05"></a>

### Task N05: Claude Code SDK 适配

**Depends on:** N03

**Files:**
- Create: `packages/runtime-native/src/providers/claude.ts`
- Create: `tests/fixtures/native/claude-transcript.json`
- Create: `docs/providers/claude.md`

- Test: `tests/integration/claude-adapter.test.ts`

**Interfaces:**

- **Consumes:** `ProviderPort`（N02）；N03 持久投递与本地日志；F07 冻结动作审批模型。
- **Produces:** `createClaudeProvider(registration: LocalRegistration): ProviderPort`。LocalRegistration 必须显式绑定兼容的已安装 CLI 和获准配置来源。

- [ ] **Step 1: 编写失败测试。** 设置本地已安装 CLI、明确 settingSources 和独立配置范围，模拟 canUseTool/hooks 审批、session resume 与取消；用户家目录另有未授权 MCP 配置，不能被默认加载。

```ts
it("explicit configuration sources control loaded resources", () => {
  expect(loadedSettingSources).toEqual(approvedSettingSources);
  expect(loadedMcpNames).not.toContain("unapproved-home-mcp");
  expect(globalConfigAfter).toEqual(globalConfigBefore);
});
it("approval cannot be auto granted without action scope", () => {
  expect(unscopedWriteDecision).toBe("deny");
  expect(afterCancelRequest.executionState).not.toBe("quiescent");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/claude-adapter.test.ts`。预期 FAIL，原因应为默认加载未批准配置、无范围审批放行或取消请求被视为已停止；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** claude.ts 使用官方 TypeScript Agent SDK，并显式指定兼容 CLI、工作目录、settingSources 和支持的模型参数；将官方审批/hooks 映射为带对象及摘要的事件。支持什么隔离须逐项实测，settingSources 不等于完整沙箱。身份采用官方允许的 API 凭据或获准认证方式，不把 claude.ai 消费者登录/额度作为产品默认入口；密钥留在设备。SDK/CLI 版本不兼容时拒绝注册。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/claude-adapter.test.ts` 和 `pnpm typecheck`，预期全部 PASS。真实认证及原生恢复需用执行时获准的环境验证，测试结果写入 provider 文档。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-native/src/providers/claude.ts tests/fixtures/native/claude-transcript.json docs/providers/claude.md tests/integration/claude-adapter.test.ts
git commit -m "feat: n05 Claude Code SDK 适配"
```

<a id="n06"></a>

### Task N06: dsh 会话与通知隔离

**Depends on:** N03

**Files:**
- Create: `packages/runtime-native/src/providers/dsh.ts`
- Create: `tests/fixtures/native/dsh-transcript.json`
- Create: `docs/providers/dsh.md`

- Test: `tests/integration/dsh-adapter.test.ts`

**Interfaces:**

- **Consumes:** `ProviderPort`（N02）；N03 设备事件缓冲；F08 required/optional 依赖语义。
- **Produces:** `createDshProvider(registration: LocalRegistration): ProviderPort`；`filterDshEvent(rootSessionId: Id, allowedDescendants: Id[], event: NativeEvent): NativeEvent | null`。

- [ ] **Step 1: 编写失败测试。** 两个不相关根 session 共用 runtime，目标根有必需子 session；输入返回 messageId，之后依次收到 assistant 文本、idle、根结束、必需子结束。夹具含不同 provider/model/reasoningEffort 组合。

```ts
it("runtime-wide notifications are filtered on device", () => {
  expect(uploadedSessionIds).toEqual([rootId,requiredChildId]);
  expect(filterDshEvent(rootId,[requiredChildId],foreignEvent)).toBeNull();
});
it("message receipt and early idle do not finish run", () => {
  expect(afterMessageId.executionState).toBe("running");
  expect(afterRootOnly.executionState).toBe("running");
  expect(afterRequiredDescendants.executionState).toBe("completed");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/dsh-adapter.test.ts`。预期 FAIL，原因应为上传外部会话事件或把 messageId/idle 当最终结果；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** dsh.ts 使用官方 SDK 或 stdio JSON-RPC，显式指定 dshBin、profile、provider/model 和支持的 reasoningEffort；能力探测与实际版本绑定。过滤必须发生在设备上传之前；内部子 Agent 只作为本 Run 子轨迹。只有协议约定的根会话与所需后代结束证据才返回 completed。未知通知保留本地诊断并标注 coverage，不擅自解释为成功。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/dsh-adapter.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-native/src/providers/dsh.ts tests/fixtures/native/dsh-transcript.json docs/providers/dsh.md tests/integration/dsh-adapter.test.ts
git commit -m "feat: n06 dsh 会话与通知隔离"
```

<a id="n07"></a>

### Task N07: 原生恢复与私密结果发布

**Depends on:** F07, F08, N04, N05, N06

**Files:**
- Create: `packages/runtime-native/src/orchestrator.ts`
- Create: `packages/runtime-native/src/reconcile.ts`
- Create: `packages/runtime-native/src/sharing.ts`
- Create: `packages/runtime-native/src/cleanup.ts`
- Create: `packages/infra/migrations/016-native-runs.sql`
- Create: `apps/api/src/routes/native-runs.ts`
- Modify: `apps/worker/src/main.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/native-lifecycle.test.ts`

**Interfaces:**

- **Consumes:** `sendCommand(lease: Lease, command: NativeCommand): Promise<{state: "queued" | "delivered"}>`（N01）；`acceptDeviceEvent(deviceId: Id, event: NativeEvent): Promise<{ackSeq: number}>`（N01）；`computeSessionScope(input: SessionScope): Digest`（N02）；`prepareApproval(lease: Lease, input: ApprovalInput): Promise<Approval>`（F07）；`requireApproval(lease: Lease, approvalId: Id, expected: ApprovalInput): Promise<void>`（F07）；`reserveResources(lease: Lease, resources: string[], waitsFor: Id[]): Promise<void>`（F08）；`aggregateTask(taskId: Id): Promise<TaskStatus>`（F05）；`recordChildResult(childRunId: Id, outcome: RunOutcome): Promise<void>`（F08）；`commitRunOutcome(lease: Lease, outcome: RunOutcome): Promise<void>`（F06）。
- **Produces:** `startNative(lease: Lease, run: RunSpec): Promise<{commandId: Id; sessionScope: Digest}>`；`reconcileNative(actor: Actor, runId: Id): Promise<NativeObservation>`；`cancelNative(lease: Lease, runId: Id): Promise<NativeObservation>`；`publishNativeResult(lease: Lease, result: RevisionRef, approvalId?: Id): Promise<RunOutcome>`；`requestNativeCleanup(actor: Actor, profileRef: RevisionRef, sources: RevisionRef[]): Promise<{cleanupId: Id; status: "pending" | "confirmed" | "unsupported"}>`。

- [ ] **Step 1: 编写失败测试。** 设备断线但进程仍执行、云 Worker 接管、profile 漂移、取消后必需原生后代仍运行、结果先完成再等待分享、跨成员调用同一实例，以及原生记忆删除能力 unknown。

```ts
it("disconnect and cancellation cannot trigger replacement execution", () => {
  expect(disconnectedRun.connectionState).toBe("offline");
  expect(unconfirmedCancel.runStatus).toBe("blocked");
  expect(replacementPrompts).toBe(0);
});
it("personal results stay private until exact sharing consent", () => {
  expect(beforeShare.runStatus).toBe("waiting_approval");
  expect(beforeShare.nativeExecutionState).toBe("completed");
  expect(publicTokensBeforeConsent).toEqual([]);
  expect(unconfirmedMemoryCleanup.status).toBe("pending");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/native-lifecycle.test.ts`。预期 FAIL，原因应为离线导致重复执行、未停止却显示 cancelled 或私密流直接广播；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** orchestrator.ts 创建经授权裁剪的 ContextEnvelope，固定 RuntimeBinding 并保留 session/workdir 占用，执行与投递/连接状态分开。reconcile.ts 只向原设备查询原命令/会话，不做云端故障转移；无恢复证据则 blocked，版本漂移拒绝续接。取消或超预算只有原生及需停止的后代静止后才落终态；停止新派发与中断已发请求分开记录。sharing.ts 个人模式所有原始流和产物先 owner-private，按结果 revision+受众批准后发布，拒绝保留私密产物和未分享原因。cleanup.ts 跟踪原生清理确认并联动冻结相关后续输入。路由包括 native-session 与 reconcile。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/native-lifecycle.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-native/src/orchestrator.ts packages/runtime-native/src/reconcile.ts packages/runtime-native/src/sharing.ts packages/runtime-native/src/cleanup.ts packages/infra/migrations/016-native-runs.sql apps/api/src/routes/native-runs.ts apps/worker/src/main.ts apps/api/src/app.ts tests/integration/native-lifecycle.test.ts
git commit -m "feat: n07 原生恢复与私密结果发布"
```

<a id="n08"></a>

### Task N08: 跨执行后端的协作桥

**Depends on:** F08, M08, N07, M06

**Files:**
- Create: `packages/runtime-native/src/collaboration-bridge.ts`
- Create: `packages/runtime-native/src/continuation.ts`
- Create: `tests/fixtures/native/bridge-protocol.json`
- Modify: `packages/runtime-native/src/orchestrator.ts`
- Test: `tests/integration/cross-runtime.test.ts`

**Interfaces:**

- **Consumes:** `delegate(lease: Lease, input: DelegateInput): Promise<{childRunId: Id; waitId: Id}>`（F08）；`resumeManaged(lease: Lease, envelope: ResumeEnvelope): Promise<RunOutcome | WaitEvidence>`（M08）；`startNative(lease: Lease, run: RunSpec): Promise<{commandId: Id; sessionScope: Digest}>`（N07）；`reconcileNative(actor: Actor, runId: Id): Promise<NativeObservation>`（N07）；`takeResume(lease: Lease, waitId: Id): Promise<ResumeEnvelope | null>`（F08）；`ackResume(lease: Lease, envelopeId: Id): Promise<void>`（F08）。
- **Produces:** `handleBridge(lease: Lease, method: "delegate_task" | "task_status" | "task_result", args: BridgeArgs): Promise<BridgeResult>`；`continueNative(lease: Lease, envelope: ResumeEnvelope): Promise<{commandId: Id}>`。BridgeArgs 为 DelegateInput 或 {childRunId: Id}；BridgeResult 为委派引用、授权可见状态或结果 revision。

- [ ] **Step 1: 编写失败测试。** managed 父等待 native 子；native 父 turn 先结束但必需 managed 子未结束；同一结果重投，以及不支持可关联后续输入的原生提供者。测试跨 Task 互占会话环仍走 F08。

```ts
it("native parent continues same run once with a new command", () => {
  expect(parentAfterTurn.runStatus).toBe("waiting_children");
  expect(continuedRunId).toBe(parentRunId);
  expect(continuationCommandId).not.toBe(firstCommandId);
  expect(continuationCountForGeneration).toBe(1);
});
it("unsupported required continuation and text mentions cannot dispatch", () => {
  expect(unsupportedContinuation.code).toBe("CAPABILITY_UNSUPPORTED");
  expect(childRunsFromTextMention).toBe(0);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/cross-runtime.test.ts`。预期 FAIL，原因应为父 Run 提前成功、重复续接或把文本 @ 当协作命令；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** collaboration-bridge.ts 只在设备所有者批准的配置中启用，以 run-scoped 短期凭据调用 Coordinator，绑定实际客户端兼容的 MCP 协议；task_result 按受众校验，不借桥访问其他 Task。continuation.ts 用 wait_generation+结果 revision 形成稳定的新 commandId，以后续 turn 继续原父 Run；结果不伪装成旧工具回执。协议不支持串行可关联续接时拒绝异步必需委派，只允许已验证同步等待或无需父续接的可选委派。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/cross-runtime.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-native/src/collaboration-bridge.ts packages/runtime-native/src/continuation.ts tests/fixtures/native/bridge-protocol.json packages/runtime-native/src/orchestrator.ts tests/integration/cross-runtime.test.ts
git commit -m "feat: n08 跨执行后端的协作桥"
```

