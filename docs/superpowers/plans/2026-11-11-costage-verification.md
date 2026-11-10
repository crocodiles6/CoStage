# CoStage 运行与验收 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让质量、费用、恢复与容量都有可查的证据，并形成可部署的首版。

**Architecture:** 观测副本不决定业务状态，评测绑定不可变发布，故障与隔离测试覆盖两个执行后端。容量和云端备份指标通过独立环境实测，设备恢复按原生能力单列。

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

- O01：观测端不可用与业务结果重放，不得影响聊天或重复收费。
- O02：评测回放引发真实副作用，必须由默认模拟工具阻止。
- O03：原生 hook/MCP/后台子进程绕过限制，不能用表面配置通过受控模式。
- O04：滚动更新后不兼容 Worker 接管旧 Run，必须拒绝而非丢弃状态。
- O04：容量与灾备只写目标没有实际测量，验收报告不得标为通过。

---

## 文件与验收层次

O01 的 API 可以先由测试读取，产品面板在 P05 接入，因此不存在观测与界面循环依赖。O02 固定评测资料和发布条件；O03 检查真实隔离与跨模块故障；O04 才做负载、部署和恢复演练。

本计划中的 PASS 都是未来执行的验收要求。当前规划阶段没有运行产品测试、连接个人原生 Agent 或创建部署环境。真实第三方调用与设备操作按执行时已有授权进行，缺少环境时报告具体未验证项，不用模拟结果替代。

<a id="o01"></a>

### Task O01: 运行观测与可核对费用

**Depends on:** F05, M04, M08, N07

**Files:**
- Create: `packages/infra/src/telemetry.ts`
- Create: `packages/core/src/observability/query.ts`
- Create: `packages/core/src/usage/pricing.ts`
- Create: `packages/contracts/src/observability.ts`
- Create: `packages/infra/migrations/019-observability.sql`
- Create: `apps/api/src/routes/observability.ts`
- Create: `deploy/otel-collector.yaml`
- Modify: `apps/api/src/app.ts`
- Modify: `apps/worker/src/main.ts`
- Modify: `apps/connector-gateway/src/server.ts`
- Test: `tests/integration/observability.test.ts`

**Interfaces:**

- **Consumes:** `settleUsage(reservationId: Id, actual: UsageAmount): Promise<void>`（F05）；`getRun(actor: Actor, runId: Id): Promise<RunSpec>`（F05）。另消费 M01 ModelInvocation、M04 阶段事件、M08 replay/cache_hit 与 N07 NativeEvent，字段见总入口。
- **Produces:** `recordTrace(event: TraceRecord): Promise<void>`；`readRunTrace(actor: Actor, runId: Id): Promise<RunTrace>`；`readUsage(actor: Actor, scope: {roomId?: Id; releaseId?: Id; taskId?: Id}): Promise<UsageReport>`；`priceUsage(usage: UsageAmount, priceRevision: Revision): CostEstimate`。TraceRecord/RunTrace/UsageReport 见总入口，费用必须区分 actual/estimated/unknown。

- [ ] **Step 1: 编写失败测试。** 关闭 Langfuse/OTel 接收端后继续完成任务；原文和秘密仅进入加密受权存储，公开 trace 不含正文；同 modelCall 业务结果重放不重复计费，native 缺 usage。

```ts
it("telemetry outage does not block facts or duplicate usage", () => {
  expect(runStatus).toBe("succeeded");
  expect(auditedToolIntents).toHaveLength(1);
  expect(chargedModelInvocations).toBe(1);
  expect(replayTrace.kind).toBe("cache_hit");
});
it("native cost gaps stay explicit", () => {
  expect(nativeReport.completeness).toBe("partial");
  expect(nativeReport.unknownCount).toBe(1);
  expect(publicTraceText).not.toContain(secretText);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/observability.test.ts`。预期 FAIL，原因应为观测故障阻断任务、重放重复计费或未知费用显示零；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** telemetry.ts 导出 OTel 与 Langfuse 脱敏事件，异步委派用 trace link，人工等待结束当前 span；导出失败只积压观测副本，关键审计仍先写业务账本。query.ts 按当前权限返回输入引用和工具/审批/委派轨迹；隐藏 reasoning 不作为必需数据。pricing.ts 固定带生效时间的 PriceCatalog，记录实际/估算与未知部分，本地推理另计 GPU 时间口径。指标覆盖设计的服务、恢复、模型、工具、成本、压缩、协作和设备状态，不让观测监听器修改循环。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/observability.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/infra/src/telemetry.ts packages/core/src/observability/query.ts packages/core/src/usage/pricing.ts packages/contracts/src/observability.ts packages/infra/migrations/019-observability.sql apps/api/src/routes/observability.ts deploy/otel-collector.yaml apps/api/src/app.ts apps/worker/src/main.ts apps/connector-gateway/src/server.ts tests/integration/observability.test.ts
git commit -m "feat: o01 运行观测与可核对费用"
```

<a id="o02"></a>

### Task O02: 版本评测与发布准入

**Depends on:** F03, P04, P05

**Files:**
- Create: `packages/core/src/evaluation/runner.ts`
- Create: `packages/core/src/evaluation/release-gate.ts`
- Create: `packages/contracts/src/evaluation.ts`
- Create: `tests/fixtures/eval/roleplay.json`
- Create: `tests/fixtures/eval/mystery.json`
- Create: `tests/fixtures/eval/office.json`
- Create: `apps/api/src/routes/evaluations.ts`
- Create: `apps/web/src/agents/EvaluationReport.tsx`
- Create: `docs/evaluation.md`
- Modify: `packages/core/src/releases/service.ts`
- Modify: `apps/api/src/app.ts`
- Modify: `apps/web/src/agents/AgentEditor.tsx`
- Test: `tests/integration/release-evaluation.test.ts`

**Interfaces:**

- **Consumes:** `startTask(actor: Actor, input: TaskInput): Promise<{taskId: Id; runIds: Id[]}>`（F05）；`readUsage(actor: Actor, scope: {roomId?: Id; releaseId?: Id; taskId?: Id}): Promise<UsageReport>`（O01）。
- **Produces:** `evaluateRelease(actor: Actor, releaseId: Id, dataset: RevisionRef, policy: RevisionRef): Promise<EvaluationReport>`；`checkReleaseGate(report: EvaluationReport): {allowed: boolean; reasons: string[]}`。报告固定 release/dataset/policy/model/runtime/profile refs，逐例记录结果、确定性断言、人工/模型评价、耗时、成本和可复现程度。

- [ ] **Step 1: 编写失败测试。** 三个场景包含典型任务、长对话与故障；严格剧本泄密、重复外部写入和权限越界任一出现均阻止发布。native 资源为 observed/unknown 时降低复现声明，不能制造完整输入。

```ts
it("hard behavioral regressions block release", () => {
  expect(checkReleaseGate(secretLeakReport).allowed).toBe(false);
  expect(checkReleaseGate(duplicateWriteReport).allowed).toBe(false);
  expect(checkReleaseGate(allInvariantPassReport).allowed).toBe(true);
});
it("comparison runs use simulated side effects", () => {
  expect(realExternalWrites).toBe(0);
  expect(nativeUnknownReport.reproducibility).toBe("observed");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/release-evaluation.test.ts`。预期 FAIL，原因应为严重约束失败仍发布、评测触发真实写入或夸大 native 可复现性；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** runner.ts 默认用录制/模拟工具建立关联新 Run 比较，绝不把历史 checkpoint 回放当安全预览。release-gate.ts 将权限、版本、重复副作用等程序断言设为全通过要求；文本质量按有版本的评测策略记录人工抽查与模型评审，不用通用固定分数代替场景验收。staged→published 须有合格报告；权限、泄密、重复副作用等硬约束不得人工豁免。文本质量未达推荐值时，可由有权限者记录理由后接受该质量取舍。灰度按房间/用户分组，既有 Task 和同一剧本保持原版本。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/release-evaluation.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/evaluation/runner.ts packages/core/src/evaluation/release-gate.ts packages/contracts/src/evaluation.ts tests/fixtures/eval/roleplay.json tests/fixtures/eval/mystery.json tests/fixtures/eval/office.json apps/api/src/routes/evaluations.ts apps/web/src/agents/EvaluationReport.tsx docs/evaluation.md packages/core/src/releases/service.ts apps/api/src/app.ts apps/web/src/agents/AgentEditor.tsx tests/integration/release-evaluation.test.ts
git commit -m "feat: o02 版本评测与发布准入"
```

<a id="o03"></a>

### Task O03: 故障隔离与整链验收

**Depends on:** M08, N08, P04, P05

**Files:**
- Create: `tests/fault/cross-boundary.test.ts`
- Create: `tests/fixtures/isolation/probes.json`
- Create: `packages/core/src/security/isolation-evidence.ts`
- Create: `deploy/tool-runner-container.yaml`
- Create: `deploy/tool-runner-microvm.md`
- Create: `docs/acceptance-matrix.md`
- Modify: `deploy/tool-runner-policy.json`
- Modify: `packages/runtime-native/src/profiles.ts`
- Modify: `packages/runtime-managed/src/tools/gateway.ts`
- Test: `tests/integration/isolation-policy.test.ts`

**Interfaces:**

- **Consumes:** `guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`（F06）；`withFence<T>(lease: Lease, work: (tx: Tx) => Promise<T>): Promise<T>`（F06）。另按 N02 RuntimeProfile 与 N04–N06 ProviderPort 的实际执行边界检查隔离。
- **Produces:** `verifyIsolation(evidence: IsolationEvidence, required: IsolationRequirements): {allowed: boolean; failures: string[]}`。IsolationEvidence 固定设备/OS/runtime/profile/策略版本与逐项探测结果；requirements 覆盖文件、网络、hooks、MCP、子进程、后台任务、配置/记忆目录和执行前审批。

- [ ] **Step 1: 编写失败测试。** 在测试设备/隔离执行器内运行无害探针：读取未授权临时目录、访问拒绝的本地 HTTP 桩、通过子进程/MCP/hook 重复尝试、撤权时仍有在途动作。整链故障用 M08/N03 的崩溃点，加权限撤销和重复事件交错。

```ts
it("controlled mode requires the whole effective boundary", () => {
  expect(verifyIsolation(sessionOnlyEvidence, requirements).allowed).toBe(false);
  expect(verifyIsolation(hookBypassEvidence, requirements).allowed).toBe(false);
  expect(unverifiedCodeToolEnabled).toBe(false);
});
it("revocation blocks late public effects across both backends", () => {
  expect(publicResultsAfterRevocation).toEqual([]);
  expect(restartedUnknownWrites).toBe(0);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/isolation-policy.test.ts`。预期 FAIL，原因应为只凭新 session/配置表宣称隔离或撤权后继续公开；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** isolation-evidence.ts 只接受与当前版本/范围匹配的实际探测报告，profile 漂移使旧证据失效。管理员脚本按非 root 容器、资源/目录/出站限制验收；任意代码和浏览器执行要求 microVM 或同等级边界，未部署则能力标 unsupported 并拒绝启用。原生个人模式仍可用，但不能冒充已通过受控协作；对各提供者实际可保证的隔离组合逐项验证。acceptance-matrix.md 对照设计关键行为逐条链接自动测试、实测证据或具体受限能力。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/isolation-policy.test.ts` 和 `pnpm typecheck`，预期全部 PASS。另运行 `pnpm exec vitest run tests/fault/cross-boundary.test.ts`，预期整链故障断言全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add tests/fault/cross-boundary.test.ts tests/fixtures/isolation/probes.json packages/core/src/security/isolation-evidence.ts deploy/tool-runner-container.yaml deploy/tool-runner-microvm.md docs/acceptance-matrix.md deploy/tool-runner-policy.json packages/runtime-native/src/profiles.ts packages/runtime-managed/src/tools/gateway.ts tests/integration/isolation-policy.test.ts
git commit -m "feat: o03 故障隔离与整链验收"
```

<a id="o04"></a>

### Task O04: 部署与容量恢复演练

**Depends on:** O01, O02, O03

**Files:**
- Create: `deploy/compose.yaml`
- Create: `deploy/Dockerfile.api`
- Create: `deploy/Dockerfile.worker`
- Create: `deploy/Dockerfile.gateway`
- Create: `deploy/connector-release.md`
- Create: `tests/load/platform.ts`
- Create: `tests/load/thresholds.json`
- Create: `tests/operations/restore.test.ts`
- Create: `docs/operations.md`
- Create: `.github/workflows/ci.yml`
- Create: `.github/workflows/connector-compatibility.yml`
- Modify: `package.json`
- Modify: `docs/development.md`
- Test: `tests/operations/deployment.test.ts`

**Interfaces:**

- **Consumes:** `buildApp(): Promise<FastifyInstance>`（F01）；`claimJob(workerId: Id, supportedRuntimeRevisions: Revision[]): Promise<Lease | null>`（F06）；`readRunTrace(actor: Actor, runId: Id): Promise<RunTrace>`（O01）。
- **Produces:** 部署入口为 API、Worker、Connector Gateway、隔离执行器、PostgreSQL/pgvector、对象存储、独立 LiteLLM 与观测系统；可选 Python 服务关闭。导出 `measurePlatform(): Promise<CapacityReport>`，报告保存负载条件、硬件、版本、样本、百分位、失败率和阈值判定。

- [ ] **Step 1: 编写失败测试。** 在独立验收环境运行确定性模型桩，1000 在线连接、20 消息/秒、100 并发模型请求、平均每请求 20 秒；压测中杀死 Worker、断 SSE、撤销权限。备份演练写入带时间戳数据后故障恢复，记录实际数据缺口与恢复时间。

```ts
it("platform meets measured design targets", () => {
  expect(report.messageCommitP95Ms).toBeLessThanOrEqual(200);
  expect(report.queueP95MsWithinQuota).toBeLessThanOrEqual(1000);
  expect(report.chunkForwardP95Ms).toBeLessThanOrEqual(100);
  expect(report.workerRecoveryMaxMs).toBeLessThanOrEqual(60000);
  expect(report.stopNewDispatchMaxMs).toBeLessThanOrEqual(2000);
});
it("cloud recovery meets trial RPO and RTO", () => {
  expect(restore.dataLossMs).toBeLessThanOrEqual(300000);
  expect(restore.elapsedMs).toBeLessThanOrEqual(3600000);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/operations/deployment.test.ts`。预期 FAIL，原因应为部署缺组件、平台开销超目标或备份无法恢复；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** 固定部署镜像及 secrets 引用，API 无状态，Worker 停止领取→安全节点→兼容接管，迁移使用独立受控连接。PostgreSQL 开启备份/PITR，对象存储版本/生命周期，观测存储单列资源。Connector 首批交付当前用户使用的 macOS arm64 以及 Linux x64 的安装/doctor 验证，提供固定 Node.js 24 运行环境或明确检查兼容版本；其他平台未验证前不宣称支持。CI 默认类型/单元/集成/浏览器/协议录制测试，容量和恢复演练单独触发。运行命令写入 operations.md，失败目标保留实测结果并定位瓶颈，不改低阈值换通过。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/operations/deployment.test.ts` 和 `pnpm typecheck`，预期全部 PASS。验收环境执行 `pnpm exec tsx tests/load/platform.ts --profile design` 与 `pnpm exec vitest run tests/operations/restore.test.ts`；真实供应商、本机提供者与设备离线另出测量报告，不混入 managed 平台开销结论。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add deploy/compose.yaml deploy/Dockerfile.api deploy/Dockerfile.worker deploy/Dockerfile.gateway deploy/connector-release.md tests/load/platform.ts tests/load/thresholds.json tests/operations/restore.test.ts docs/operations.md .github/workflows/ci.yml .github/workflows/connector-compatibility.yml package.json docs/development.md tests/operations/deployment.test.ts
git commit -m "feat: o04 部署与容量恢复演练"
```

