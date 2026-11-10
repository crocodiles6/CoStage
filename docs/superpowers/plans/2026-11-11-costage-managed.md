# CoStage managed 运行时 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现可控制上下文、记忆和工具的自研 Agent，并以 LangGraph JS 持久恢复。

**Architecture:** 业务账本是动作与权限的权威来源，LangGraph 保存持久阶段的引用与中断点。LoopController 决定下一步，Model/Tool Gateway 执行受限动作，Python 仅能返回能力结果。

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

- M01：部分流已发出后重试，客户端必须区分 attempt，保密本地模型不得自动转云端。
- M03/M04：资料删除与摘要重用，派生缓存失效且旧快照仍按当前授权读取。
- M05：写入成功但响应丢失，必须保持 unknown，禁止新 ID 重做。
- M07/M08：旧租约及 sync 模式写入尚未完成，不能错误提交 checkpoint 或提前恢复等待。
- M09：Python 超时、撤权和晚到结果，TS 仍持有业务提交权。

---

## 文件与执行顺序

`runtime-managed/models` 统一模型适配，`context` 负责输入，`loop` 负责决策，`runtime` 负责 LangGraph 对接。平台长期记忆和资料放在 `core/memory`、`core/documents`，不放进 LangGraph Store。工具执行器是隔离进程，不能直接读业务数据库。

M01→M02→M03→M04 和 M05→M06 汇合到 M07，再完成 M08。M09 是可测试的协议预留，可在基础平台完成后单独交付；真实 Python 服务按需求启用，不计入首版必需部署。

MCP 2026-07-28 的无状态请求和逐请求能力协商来自[官方规范](https://modelcontextprotocol.io/specification/2026-07-28)。LangGraph 恢复依赖持久化，具体 JS 包行为须用 M07/M08 的真实 checkpointer 测试固定，[官方说明](https://docs.langchain.com/oss/javascript/langgraph/persistence)用于解释边界，不替代版本兼容测试。

<a id="m01"></a>

### Task M01: 模型网关与唯一重试责任

**Depends on:** F03, F05, F06

**Files:**
- Create: `packages/runtime-managed/package.json`
- Create: `packages/runtime-managed/src/models/gateway.ts`
- Create: `packages/runtime-managed/src/models/profiles.ts`
- Create: `packages/runtime-managed/src/models/adapters.ts`
- Create: `packages/contracts/src/models.ts`
- Create: `packages/contracts/src/model-input.ts`
- Create: `packages/core/src/model-inputs/store.ts`
- Create: `packages/core/src/credentials/service.ts`
- Create: `packages/infra/src/secrets.ts`
- Create: `packages/infra/migrations/008-models.sql`
- Create: `apps/api/src/routes/model-connections.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/model-gateway.test.ts`

**Interfaces:**

- **Consumes:** `guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`（F06）；`reserveUsage(scope: RunRef | SystemJobRef, operationId: Id, amount: UsageReservation): Promise<{reservationId: Id}>`（F05）；`settleUsage(reservationId: Id, actual: UsageAmount): Promise<void>`（F05）。
- **Produces:** `bindCredential(actor: Actor, input: {owner: "personal" | "workspace"; connectionId: Id; secret: string}): Promise<{credentialRef: Id}>`；`storeModelInput(lease: Lease, snapshot: SnapshotDraft): Promise<ModelInputSnapshot>`；`embedText(lease: ActionLease, operationId: Id, profile: RevisionRef, texts: string[]): Promise<number[][]>`；`validateModelProfile(actor: Actor, profile: ModelProfile): Promise<RevisionRef>`；`invokeModel(lease: Lease, call: ModelCall): Promise<ModelResponse>`；`streamModel(lease: Lease, call: ModelCall): AsyncIterable<ModelStreamEvent>`。ModelCall 固定 modelCallId、snapshotRef、profileRevision、requestBodyRef；ModelResponse 含 contentRef、toolCalls、provider、actualModel、fingerprint、usage、attempts；缺失 usage 标 estimated 或 unknown。

- [ ] **Step 1: 编写失败测试。** 模型桩按顺序返回 429、超时、流中断、成功；测试 local-only profile 没有备用云端路由，另一个 profile 只允许兼容能力且同驻留区域的备用模型。登记连接时包含云元数据地址、未登记私网、重定向与 DNS 变化。

```ts
it("one gateway owns retries and never joins attempts", () => {
  expect(httpRequests).toBe(3);
  expect(sdkRetries).toBe(0);
  expect(chunks.map(c=>c.attempt)).toEqual([1,1,2,2]);
  expect(localOnlyCloudRequests).toBe(0);
});
it("unapproved endpoints and missing capabilities are rejected", () => {
  expect(metadataEndpoint.code).toBe("ENDPOINT_DENIED");
  expect(toolLessModel.code).toBe("CAPABILITY_UNSUPPORTED");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/model-gateway.test.ts`。预期 FAIL，原因应为出现叠加重试、local-only 请求被转到云端或危险 endpoint 获准；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** gateway.ts 为唯一网络重试负责人，关闭 SDK 同层自动重试，沿用 modelCallId 并逐 attempt 计预算。profiles.ts 校验 model/endpoint/credential_ref、能力及供应商参数 schema；出站策略在实际连接与重定向时生效。adapters.ts 实现 LiteLLM HTTP 和明确所需的供应商原生适配，Ollama/vLLM 按实际工具与流式语义做契约测试。credentials/service.ts 将个人/团队凭据绑定授权主体；infra/secrets.ts 对密钥加密存储，解密只授予实际模型/工具执行路径，浏览器查询、快照和 trace 均只留引用。测试确认不同主体不能借用同一连接的凭据。模型响应先落账本和对象存储再返回；model-inputs/store.ts 持有快照存储接口，M04 只负责组装，避免模型网关与上下文组装器循环依赖。fallback 改变实际模型时通过此存储接口保存新快照并记录原因；候选模型输入窗口不容纳原输入时拒绝该备用路由，不静默删改材料。embedding 也通过本网关记账。路由提供模型连接登记/探测及 profile revision，密钥不返回浏览器。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/model-gateway.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-managed/package.json packages/runtime-managed/src/models/gateway.ts packages/runtime-managed/src/models/profiles.ts packages/runtime-managed/src/models/adapters.ts packages/contracts/src/models.ts packages/contracts/src/model-input.ts packages/core/src/model-inputs/store.ts packages/core/src/credentials/service.ts packages/infra/src/secrets.ts packages/infra/migrations/008-models.sql apps/api/src/routes/model-connections.ts apps/api/src/app.ts tests/integration/model-gateway.test.ts
git commit -m "feat: m01 模型网关与唯一重试责任"
```

<a id="m02"></a>

### Task M02: 文件修订与权限内检索

**Depends on:** M01

**Files:**
- Create: `packages/core/src/documents/ingest.ts`
- Create: `packages/core/src/documents/retrieval.ts`
- Create: `packages/contracts/src/documents.ts`
- Create: `packages/infra/migrations/009-documents.sql`
- Create: `apps/api/src/routes/files.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/document-retrieval.test.ts`

**Interfaces:**

- **Consumes:** `putBlob(bytes: Uint8Array, mediaType: string): Promise<BlobRef>`（F01）；`authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）；`embedText(lease: ActionLease, operationId: Id, profile: RevisionRef, texts: string[]): Promise<number[][]>`（M01）；`claimSystemJob(workerId: Id, kind: "document_index" | "memory_cleanup"): Promise<SystemLease | null>`（F06）；`withSystemFence<T>(lease: SystemLease, work: (tx: Tx) => Promise<T>): Promise<T>`（F06）。
- **Produces:** `registerFile(actor: Actor, input: {blob: BlobRef; audience: Audience; replaces?: RevisionRef}): Promise<RevisionRef>`；`indexRevision(lease: SystemLease, file: RevisionRef): Promise<void>`；`retrieve(grant: Grant, input: {query: string; revisionRefs: RevisionRef[]; limit: number}): Promise<EvidenceChunk[]>`。EvidenceChunk 为 {source: RevisionRef; chunkId: Id; text: string; audience: Audience; score: number}。

- [ ] **Step 1: 编写失败测试。** 同一文档有 ready 的 r1 和正在解析的 r2，另一个 tenant 的高度相似私密文本；检索计数器记录进入排序的候选集合。上传相同内容但 ACL 不同的文件，正文存储可按摘要复用，授权不能复用。

```ts
it("retrieval filters candidates before similarity ranking", () => {
  expect(candidateTenantIds).toEqual([tenantId]);
  expect(results.every(x=>x.source.revision==="r1")).toBe(true);
  expect(results.some(x=>x.text.includes("foreign secret"))).toBe(false);
});
it("partial index never becomes visible", () => {
  expect(queryableRevision).toBe("r1");
  expect(failedIndex.status).toBe("failed");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/document-retrieval.test.ts`。预期 FAIL，原因应为索引半成品、新旧段落或未授权文本进入候选集；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** ingest.ts 固定上传 revision 后异步解析、切块、embedding，完整写入后才切换 ready 索引引用；上传事务写入 SystemJob，后台索引用 SystemLease 校验授权并扣独立可见的系统预算，不要求用户先 @ Agent；首批支持 UTF-8 文本、Markdown 和有明确解析器的 PDF，其他格式返回 UNSUPPORTED_MEDIA。retrieval.ts 在 SQL 候选集合执行 tenant、资料和受众权限过滤再排序，不先全库近邻后隐藏。文档内容只作为材料，不执行文件内指令。API 提供上传授权、完成登记、revision 状态和按权限下载。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/document-retrieval.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/documents/ingest.ts packages/core/src/documents/retrieval.ts packages/contracts/src/documents.ts packages/infra/migrations/009-documents.sql apps/api/src/routes/files.ts apps/api/src/app.ts tests/integration/document-retrieval.test.ts
git commit -m "feat: m02 文件修订与权限内检索"
```

<a id="m03"></a>

### Task M03: 可纠正与可删除的长期记忆

**Depends on:** F05, M02

**Files:**
- Create: `packages/core/src/memory/service.ts`
- Create: `packages/core/src/memory/invalidation.ts`
- Create: `packages/contracts/src/memory.ts`
- Create: `packages/infra/migrations/010-memory.sql`
- Create: `apps/api/src/routes/memory.ts`
- Modify: `apps/api/src/routes/files.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/memory.test.ts`

**Interfaces:**

- **Consumes:** `authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）；`withFence<T>(lease: Lease, work: (tx: Tx) => Promise<T>): Promise<T>`（F06）；`retrieve(grant: Grant, input: {query: string; revisionRefs: RevisionRef[]; limit: number}): Promise<EvidenceChunk[]>`（M02）。
- **Produces:** `registerDerivedSources(tx: Tx, derived: RevisionRef, sources: RevisionRef[]): Promise<void>`；`invalidateSource(actor: Actor, source: RevisionRef): Promise<{deletionId: Id; status: "pending" | "complete"}>`；`commitMemory(lease: Lease, operationId: Id, candidate: MemoryCandidate): Promise<RevisionRef>`；`readMemory(grant: Grant, scope: MemoryScope, atRevision: Revision): Promise<MemoryRevision[]>`；`correctMemory(actor: Actor, source: RevisionRef, text: string): Promise<RevisionRef>`；`deleteMemory(actor: Actor, source: RevisionRef): Promise<{deletionId: Id; status: "pending" | "complete"}>`。MemoryScope 为 room/branch/instance 或显式授权的 personal/project/team；候选包含文本、来源、可信度、有效期、extractorRevision 与 audience。

- [ ] **Step 1: 编写失败测试。** 同一 release 绑定两个房间，模型推断与用户纠正互相冲突；同 operationId 提交两次；删除内容同时存在向量、摘要、缓存、快照衍生引用。

```ts
it("confirmed corrections win without overwriting history", () => {
  expect(effectiveMemory.text).toBe(userCorrection);
  expect(effectiveMemory.replaces).toBe(previousRevision);
  expect(otherRoomMemories).toEqual([]);
});
it("replay and deletion respect provenance", () => {
  expect(firstCommit.revision).toBe(replayedCommit.revision);
  expect(searchAfterDeletion).toEqual([]);
  expect(derivedCaches.every(x=>x.invalidated)).toBe(true);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/memory.test.ts`。预期 FAIL，原因应为跨房间串记忆、重放产生新修订或删除后仍可检索；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** service.ts 按 scope、ACL 和固定起始视图管理记忆 revision，候选去重与冲突保留来源；自动写入只接受低风险偏好和任务结论，项目规则/剧情真相须用户或规则引擎确认。invalidation.ts 持有通用来源依赖图，摘要/索引/缓存只登记 revision 引用，M04 再接入真实摘要写入。invalidateSource 同时用于文件删除与记忆删除，files 路由接入 DELETE /files/:id/revisions/:revision；记录来源依赖并立即撤销读取资格，再清理向量、摘要、缓存及保留策略下的正文；审计只留必要元数据。原生记忆清理通过 N07 的能力请求跟踪，未收到原生确认不得显示完成。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/memory.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/memory/service.ts packages/core/src/memory/invalidation.ts packages/contracts/src/memory.ts packages/infra/migrations/010-memory.sql apps/api/src/routes/memory.ts apps/api/src/routes/files.ts apps/api/src/app.ts tests/integration/memory.test.ts
git commit -m "feat: m03 可纠正与可删除的长期记忆"
```

<a id="m04"></a>

### Task M04: 上下文压缩与逐次输入快照

**Depends on:** M01, M02, M03

**Files:**
- Create: `packages/runtime-managed/src/context/assembler.ts`
- Create: `packages/runtime-managed/src/context/compaction.ts`
- Create: `packages/runtime-managed/src/context/snapshots.ts`
- Create: `packages/contracts/src/context.ts`
- Create: `packages/infra/migrations/011-context.sql`

- Test: `tests/integration/context.test.ts`

**Interfaces:**

- **Consumes:** `registerDerivedSources(tx: Tx, derived: RevisionRef, sources: RevisionRef[]): Promise<void>`（M03）；`getRun(actor: Actor, runId: Id): Promise<RunSpec>`（F05）；`retrieve(grant: Grant, input: {query: string; revisionRefs: RevisionRef[]; limit: number}): Promise<EvidenceChunk[]>`（M02）；`readMemory(grant: Grant, scope: MemoryScope, atRevision: Revision): Promise<MemoryRevision[]>`（M03）；`invokeModel(lease: Lease, call: ModelCall): Promise<ModelResponse>`（M01）；`storeModelInput(lease: Lease, snapshot: SnapshotDraft): Promise<ModelInputSnapshot>`（M01）。
- **Produces:** `prepareInput(lease: Lease, stepId: Id): Promise<ModelInputSnapshot>`；`compactHistory(lease: Lease, operationId: Id, sources: RevisionRef[], maxTokens: number): Promise<SummaryRevision>`；`acceptDeltas(lease: Lease, throughSeq: number): Promise<RevisionRef[]>`。快照含 modelCallId、grantRevision、消息/记忆/摘要/技能/工具 refs、resolvedConfig、encryptedBodyRef、requestDigest；SummaryRevision 含覆盖区间、来源 ACL、策略与模型 revision。

- [ ] **Step 1: 编写失败测试。** 设置 contextWindow=32768、outputReserve=4096、protocolOverhead=1024、safetyMargin=512，输入上限为 27136；旧历史中有公开/私密不同范围，Task 之后有普通新消息及显式 steering。压缩提交后在 checkpoint 前崩溃。

```ts
it("budget and visibility govern every model input", () => {
  expect(inputTokenLimit).toBe(27136);
  expect(publicSnapshot.text).not.toContain(privateFact);
  expect(publicSnapshot.messageIds).not.toContain(ordinaryNewMessageId);
  expect(nextSnapshot.messageIds).toContain(acceptedSteeringId);
});
it("restart reuses committed compression and snapshot", () => {
  expect(summaryModelCalls).toBe(1);
  expect(recoveredSnapshot.id).toBe(originalSnapshot.id);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/context.test.ts`。预期 FAIL，原因应为超过预算、泄露私密摘要、新消息篡改旧输入或重复压缩；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** assembler.ts 按固定 Task cutoff/release/grant 与已接纳增量组装，先保留系统规则、当前请求、未完成事项及关键事实；超过预算时压缩旧历史，大工具输出只带片段和 Artifact 引用。compaction.ts 的稳定操作记录先于摘要模型调用，按来源权限分别压缩；摘要费用计 Task。snapshots.ts 保存加密实际请求正文和完整 revision 引用，纯网络重试复用同一快照，新工具结果、steering 或实际模型变更创建新快照。摘要与快照在提交时登记来源依赖供撤权/删除失效。各子阶段返回带版本、过滤原因、耗时和 token 变化的观测事件供 O01 使用。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/context.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-managed/src/context/assembler.ts packages/runtime-managed/src/context/compaction.ts packages/runtime-managed/src/context/snapshots.ts packages/contracts/src/context.ts packages/infra/migrations/011-context.sql tests/integration/context.test.ts
git commit -m "feat: m04 上下文压缩与逐次输入快照"
```

<a id="m05"></a>

### Task M05: 工具执行与副作用对账

**Depends on:** F06, F07, M01

**Files:**
- Create: `packages/runtime-managed/src/tools/gateway.ts`
- Create: `packages/runtime-managed/src/tools/reconcile.ts`
- Create: `packages/contracts/src/tools.ts`
- Create: `packages/infra/migrations/012-tools.sql`
- Create: `apps/tool-runner/package.json`
- Create: `apps/tool-runner/src/server.ts`
- Create: `deploy/tool-runner-policy.json`

- Test: `tests/integration/tool-effects.test.ts`

**Interfaces:**

- **Consumes:** `guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`（F06）；`prepareApproval(lease: Lease, input: ApprovalInput): Promise<Approval>`（F07）；`requireApproval(lease: Lease, approvalId: Id, expected: ApprovalInput): Promise<void>`（F07）。
- **Produces:** `prepareTool(lease: Lease, input: ToolRequest): Promise<ToolIntent>`；`executeTool(lease: Lease, intentId: Id): Promise<ToolOutcome>`；`reconcileTool(actor: Actor, intentId: Id): Promise<ToolOutcome>`。ToolRequest 含 operationId、modelToolCallId、toolRevision、args、audience；ToolIntent 增加 argsDigest、approvalId、status；ToolOutcome 为 succeeded(ref)、failed(code)、not_started 或 unknown(reason)。

- [ ] **Step 1: 编写失败测试。** 外部桩成功写入后丢响应，无幂等/查询能力的写工具必须 unknown；有幂等能力的桩按相同 operationId 返回原回执。三个并行工具分别成功、unknown、未派发。

```ts
it("lost write receipt blocks dependent work without another write", () => {
  expect(externalWriteCount).toBe(1);
  expect(outcome.status).toBe("unknown");
  expect(newOperationIdsAfterRecovery).toEqual([]);
});
it("parallel results and authorization remain individual", () => {
  expect(toolStates).toEqual(["succeeded","unknown","not_started"]);
  expect(revokedApprovalDispatches).toBe(0);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/tool-effects.test.ts`。预期 FAIL，原因应为未知写入被重做、并行成功结果丢失或旧审批继续派发；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** gateway.ts 分 prepare、wait、execute 三段，以稳定 operationId 持久保存 intent 后调用执行器；权限、参数 schema、版本和预算在派发前再检查。reconcile.ts 查询原回执或要求人工核对，不重新调用模型生成新 ID 绕过 unknown。tool-runner 首先交付只读 HTTP 和任务产物写入执行器，以及管理员脚本的非 root 容器策略；模型生成代码、浏览器等高风险工具只有经过 O03 强隔离验收才可启用。参数决定对象和影响，副作用类别来自可信工具注册信息。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/tool-effects.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-managed/src/tools/gateway.ts packages/runtime-managed/src/tools/reconcile.ts packages/contracts/src/tools.ts packages/infra/migrations/012-tools.sql apps/tool-runner/package.json apps/tool-runner/src/server.ts deploy/tool-runner-policy.json tests/integration/tool-effects.test.ts
git commit -m "feat: m05 工具执行与副作用对账"
```

<a id="m06"></a>

### Task M06: 固定 Skill 包与 MCP 目录

**Depends on:** F03, M05

**Files:**
- Create: `packages/runtime-managed/src/skills/loader.ts`
- Create: `packages/runtime-managed/src/mcp/catalog.ts`
- Create: `packages/runtime-managed/src/mcp/client.ts`
- Create: `packages/contracts/src/mcp.ts`
- Create: `tests/fixtures/mcp/legacy.json`
- Create: `tests/fixtures/mcp/stateless.json`
- Modify: `packages/core/src/releases/manifest.ts`
- Test: `tests/integration/resources.test.ts`

**Interfaces:**

- **Consumes:** `prepareTool(lease: Lease, input: ToolRequest): Promise<ToolIntent>`（M05）；`executeTool(lease: Lease, intentId: Id): Promise<ToolOutcome>`（M05）；`saveResource(actor: Actor, input: {kind: ResourceKind; content: BlobRef; schemaVersion: string; replaces?: RevisionRef}): Promise<RevisionRef>`（F03）。
- **Produces:** `loadSkill(grant: Grant, revision: RevisionRef, level: "metadata" | "full"): Promise<SkillContent>`；`validateMcpBinding(binding: McpBindingRevision): Promise<{compatible: boolean; differences: string[]}>`；`callMcp(lease: Lease, binding: McpBindingRevision, intent: ToolIntent): Promise<ToolOutcome>`。SkillContent 含正文及允许的 references/scripts/assets；McpBindingRevision 分开固定 protocolVersion、sdkVersion、implementationRef、catalogRevision/schemaDigest 和 credentialRef。

- [ ] **Step 1: 编写失败测试。** 固定 MCP 新旧协议录制桩；工具目录新增写权限、旧 schema 参数类型变化、恶意 Skill 压缩包路径穿越、同 endpoint 不同用户凭据分别建例。

```ts
it("catalog drift blocks old release and keeps operation id", () => {
  expect(drift.compatible).toBe(false);
  expect(retryRequestIds[0]).not.toBe(retryRequestIds[1]);
  expect(retryOperationIds[0]).toBe(retryOperationIds[1]);
});
it("skill packaging cannot escape or grant permissions", () => {
  expect(pathTraversal.code).toBe("INVALID_PACKAGE");
  expect(grantAfterReadingSkill).toEqual(originalGrant);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/resources.test.ts`。预期 FAIL，原因应为漂移 schema 被静默接受、工具自动扩权或包路径越界；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** loader.ts 校验完整包 digest、依赖与路径，元数据按需加载；文本权限声明不改变 ExecutionGrant，脚本只能经 M05 执行。catalog.ts 对批准目录逐项比对，新增工具不自动加入旧 release。client.ts 分别支持批准的旧协议与 2026-07-28 无状态协议，版本在 binding 中显式选择，不能共用握手假设；业务 operationId 稳定，传输请求 ID 独立。原生自有 MCP 不经过本模块，协作桥在 N08 实现相容服务端。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/resources.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-managed/src/skills/loader.ts packages/runtime-managed/src/mcp/catalog.ts packages/runtime-managed/src/mcp/client.ts packages/contracts/src/mcp.ts tests/fixtures/mcp/legacy.json tests/fixtures/mcp/stateless.json packages/core/src/releases/manifest.ts tests/integration/resources.test.ts
git commit -m "feat: m06 固定 Skill 包与 MCP 目录"
```

<a id="m07"></a>

### Task M07: 自研循环与 LangGraph 持久阶段

**Depends on:** F06, F08, M04, M05, M06

**Files:**
- Create: `packages/runtime-managed/src/loop/controller.ts`
- Create: `packages/runtime-managed/src/loop/actions.ts`
- Create: `packages/runtime-managed/src/runtime/entrypoint.ts`
- Create: `packages/runtime-managed/src/runtime/adapter.ts`
- Create: `packages/runtime-managed/src/runtime/checkpointer.ts`
- Create: `packages/infra/migrations/013-managed-runtime.sql`
- Create: `docs/runtime-compatibility.md`
- Modify: `apps/worker/src/main.ts`
- Test: `tests/integration/managed-runtime.test.ts`

**Interfaces:**

- **Consumes:** `prepareInput(lease: Lease, stepId: Id): Promise<ModelInputSnapshot>`（M04）；`invokeModel(lease: Lease, call: ModelCall): Promise<ModelResponse>`（M01）；`prepareTool(lease: Lease, input: ToolRequest): Promise<ToolIntent>`（M05）；`executeTool(lease: Lease, intentId: Id): Promise<ToolOutcome>`（M05）；`delegate(lease: Lease, input: DelegateInput): Promise<{childRunId: Id; waitId: Id}>`（F08）；`withFence<T>(lease: Lease, work: (tx: Tx) => Promise<T>): Promise<T>`（F06）；`commitRunOutcome(lease: Lease, outcome: RunOutcome): Promise<void>`（F06）。
- **Produces:** `chooseAction(state: LoopState, response: ModelResponse): LoopAction`；`runManaged(lease: Lease, run: RunSpec): Promise<RunOutcome | WaitEvidence>`；`createFencedCheckpointer(lease: Lease): BaseCheckpointSaver`；`inspectManaged(binding: ManagedBinding): Promise<ManagedPosition>`。LoopAction 为 model/tool/delegate/wait/finish；LoopState 仅含已保存步骤引用与轮次；ManagedPosition 为 not_started、interrupted(waitId/checkpointId/interruptId)、running、completed(outcomeRef)。BaseCheckpointSaver 使用锁定 LangGraph 包的导出类型。

- [ ] **Step 1: 编写失败测试。** 用真实 PostgreSQL checkpointer、确定性模型/工具桩运行两轮；在业务结果提交后但 checkpoint 未完成时强制退出进程。旧 lease 的 put/putWrites 在另一个事务接管后尝试提交；另外测试阶段顺序版本变化。

```ts
it("business commit survives lost checkpoint", () => {
  expect(modelHttpCallsForOperation).toBe(1);
  expect(memoryRevisionsForOperation).toHaveLength(1);
  expect(recoveredRunId).toBe(originalRunId);
});
it("checkpoint fence is checked inside its write transaction", async () => {
  await expect(staleCheckpointWrite).rejects.toMatchObject({code:"STALE_LEASE"});
  expect(incompatibleWorker.code).toBe("RUNTIME_INCOMPATIBLE");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/managed-runtime.test.ts`。预期 FAIL，原因应为重复业务执行、过期 checkpoint 被提交或不兼容循环接管；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** controller.ts 自研选择动作，不使用预制 Agent 决策器；entrypoint.ts 按固定顺序调用持久阶段，每段只返回业务结果引用。adapter.ts 固定 thread_id、entrypoint_revision 和框架版本并路由 Worker。checkpointer.ts 复用框架序列化/恢复格式，put 与 putWrites 必须与租约校验处于同一 PostgreSQL 事务，不能仅在调用官方 saver 之前检查；保存接口不可插入同一事务时，在适配层实现相应 SQL 存储适配并单独测试，禁止先检查后写。关闭框架叠加的模型/工具自动重试。兼容表记录确切 Node、LangGraph、checkpointer 与运行镜像版本，旧 Run 只由兼容镜像承接。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/managed-runtime.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-managed/src/loop/controller.ts packages/runtime-managed/src/loop/actions.ts packages/runtime-managed/src/runtime/entrypoint.ts packages/runtime-managed/src/runtime/adapter.ts packages/runtime-managed/src/runtime/checkpointer.ts packages/infra/migrations/013-managed-runtime.sql docs/runtime-compatibility.md apps/worker/src/main.ts tests/integration/managed-runtime.test.ts
git commit -m "feat: m07 自研循环与 LangGraph 持久阶段"
```

<a id="m08"></a>

### Task M08: managed 等待与崩溃恢复

**Depends on:** M07

**Files:**
- Create: `packages/runtime-managed/src/runtime/wait-bridge.ts`
- Create: `packages/runtime-managed/src/runtime/reconciler.ts`
- Create: `tests/fixtures/fault-cases.json`
- Modify: `apps/worker/src/main.ts`
- Test: `tests/integration/managed-recovery.test.ts`

**Interfaces:**

- **Consumes:** `inspectManaged(binding: ManagedBinding): Promise<ManagedPosition>`（M07）；`markWaitReady(lease: Lease, waitId: Id, evidence: WaitEvidence): Promise<void>`（F08）；`takeResume(lease: Lease, waitId: Id): Promise<ResumeEnvelope | null>`（F08）；`ackResume(lease: Lease, envelopeId: Id): Promise<void>`（F08）；`claimJob(workerId: Id, supportedRuntimeRevisions: Revision[]): Promise<Lease | null>`（F06）；`releaseSlot(lease: Lease): Promise<void>`（F06）；`requestCancel(actor: Actor, target: {taskId: Id} | {runId: Id}): Promise<void>`（F05）。
- **Produces:** `reconcileManaged(runId: Id): Promise<{action: "resume" | "continue" | "finalize" | "block" | "noop"}>`；`resumeManaged(lease: Lease, envelope: ResumeEnvelope): Promise<RunOutcome | WaitEvidence>`；`persistWaitEvidence(lease: Lease, waitId: Id): Promise<WaitEvidence>`。

- [ ] **Step 1: 编写失败测试。** 故障表覆盖无首 checkpoint、业务提交/框架落后、interrupt 已存但未 ready、子结果先到、消费确认丢失、框架已完但业务未完、业务已完框架未收尾，以及 checkpoint 引用业务事实缺失。使用进程退出而非只抛函数异常。

```ts
it("sync option alone does not make wait ready", () => {
  expect(waitReadyBeforeActualPutAck).toBe(false);
  expect(resumeJobsAfterResultAndDurableInterrupt).toBe(1);
});
it("position-aware recovery never resumes a consumed interrupt", () => {
  expect(duplicateCommandResumeCount).toBe(0);
  expect(cancelledParentWakeups).toBe(0);
  expect(missingBusinessFact.action).toBe("block");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/managed-recovery.test.ts`。预期 FAIL，原因应为等待提前 ready、丢唤醒、重复 Command 或取消后又恢复；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** wait-bridge.ts 只有观察到实际 checkpoint/interrupt 持久回执才提交 ready 并释放片段槽，不能凭 durability=sync 推断。reconciler.ts 扫描业务等待与框架位置间隙；匹配 interrupt 才投原 envelope，已越过则继续故障恢复，已完成只补终态，缺失事实或不兼容版本则 blocked。每次新动作继续走实时 action guard，过去缓存的授权检查不放行。审批等待同样经此桥，unknown 工具进入持久 blocked。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/managed-recovery.test.ts` 和 `pnpm typecheck`，预期全部 PASS。把各崩溃点结果写入兼容表；不能将桩测试通过写成真实供应商已验证。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/runtime-managed/src/runtime/wait-bridge.ts packages/runtime-managed/src/runtime/reconciler.ts tests/fixtures/fault-cases.json apps/worker/src/main.ts tests/integration/managed-recovery.test.ts
git commit -m "feat: m08 managed 等待与崩溃恢复"
```

<a id="m09"></a>

### Task M09: 按需 Python 能力服务契约

**Depends on:** F05, F06

**Files:**
- Create: `packages/contracts/src/capability-service.ts`
- Create: `packages/core/src/capabilities/client.ts`
- Create: `tests/fixtures/capability-service/v1.json`
- Create: `docs/capability-service-contract.md`

- Test: `tests/integration/capability-service.test.ts`

**Interfaces:**

- **Consumes:** `guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`（F06）；`reserveUsage(scope: RunRef | SystemJobRef, operationId: Id, amount: UsageReservation): Promise<{reservationId: Id}>`（F05）。
- **Produces:** `submitCapability(lease: Lease, request: CapabilityRequest): Promise<CapabilityJob>`；`queryCapability(lease: Lease, operationId: Id): Promise<CapabilityJob>`；`cancelCapability(lease: Lease, jobId: Id): Promise<CapabilityJob>`。CapabilityRequest 为 schemaVersion、runRef、operationId、inputRefs、deadline、traceparent；CapabilityJob 含 jobId、operationId、state、outputRefs，state 为 accepted/running/succeeded/failed/cancel_requested/cancelled/unknown。

- [ ] **Step 1: 编写失败测试。** 仅启用 HTTP 录制桩，模拟响应丢失后查询已有 job、新旧 schema 不兼容、取消未确认和 Run 已撤权后晚到结果。此任务不建立 Python 生产服务。

```ts
it("timeout queries the original operation and keeps cancellation pending", () => {
  expect(createdJobs).toBe(1);
  expect(cancelResponse.state).toBe("cancel_requested");
  expect(publishedLateResultsAfterRevocation).toBe(0);
});
it("incompatible schema is rejected explicitly", () => {
  expect(incompatibleResponse.code).toBe("SCHEMA_UNSUPPORTED");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/capability-service.test.ts`。预期 FAIL，原因应为超时创建重复 job、把请求取消当已停止或接受不兼容消息；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** client.ts 固定 HTTP/JSON 的 POST /v1/jobs、GET /v1/jobs/by-operation/:id、POST /v1/jobs/:id/cancel；服务调用凭据绑定 tenant/run/scope，输出只返回不可变引用。同步能力使用同一 envelope 返回 completed 结果。契约从 TS schema 导出 JSON Schema，禁止服务直接提交 Task/Run/Approval/MemoryRevision。将来出现具体解析或推理需求时，才创建 services/<capability> 的 Python 服务并用同一录制测试验收。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/capability-service.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/contracts/src/capability-service.ts packages/core/src/capabilities/client.ts tests/fixtures/capability-service/v1.json docs/capability-service-contract.md tests/integration/capability-service.test.ts
git commit -m "feat: m09 按需 Python 能力服务契约"
```

