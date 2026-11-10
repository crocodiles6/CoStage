# CoStage 产品场景 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付可使用的多人聊天、Agent 配置、办公协作和角色场景。

**Architecture:** React 使用统一 HTTP/SSE 契约，不在前端重建任务或权限状态。角色和办公场景复用 Coordinator，严格剧本通过独立私密规划器和公开发言器隔离信息。

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

- P01：文本 @、断线重连和取消中按钮，不能制造额外任务或重复执行。
- P02：提供者参数能力不同，界面不能给出实际不支持的 effort。
- P03：旧分支、并发产物修改及来源权限，不能泄露未来内容或覆盖他人结果。
- P04：私密规划自由文本，即使模型声称可公开也不能进入公开输入。
- P05：部分成功和 native 费用未知，界面必须展示真实状态。

---

## 文件与交付顺序

前端按 rooms/agents/tasks/approvals/artifacts/scenes/runs 组织，每个页面只通过 `api/client.ts` 访问后端。P03/P04 同时增加对应的领域模块、迁移和页面，避免先做只会展示假数据的界面。

P01 可以在模型桩上先验收；P02、P03 随对应后端完成；P04 和 P05 分别验收剧本与办公端到端流程。所有浏览器测试由 playwright.config.ts 的 webServer 启动实际 API/Web 和确定性外部桩，端到端流程不连接付费供应商。

<a id="p01"></a>

### Task P01: 多人房间与任务交互

**Depends on:** F04, F05, F07

**Files:**
- Create: `apps/web/package.json`
- Create: `apps/web/index.html`
- Create: `apps/web/vite.config.ts`
- Create: `apps/web/src/main.tsx`
- Create: `apps/web/src/api/client.ts`
- Create: `apps/web/src/rooms/RoomPage.tsx`
- Create: `apps/web/src/rooms/event-store.ts`
- Create: `apps/web/src/tasks/TaskCard.tsx`
- Create: `apps/web/src/approvals/ApprovalDialog.tsx`
- Create: `playwright.config.ts`

- Test: `tests/e2e/room.spec.ts`

**Interfaces:**

- **Consumes:** `submitMessage(actor: Actor, input: MessageInput): Promise<{event: RoomEvent; taskId?: Id; runIds: Id[]}>`（F05）；`readEvents(actor: Actor, branchId: Id, after: number): Promise<RoomEvent[]>`（F04）；`streamEvents(actor: Actor, branchId: Id, after: number): AsyncIterable<StreamEvent>`（F04）；`getRun(actor: Actor, runId: Id): Promise<RunSpec>`（F05）；`steer(actor: Actor, taskId: Id, delta: RevisionRef): Promise<number>`（F05）；`requestCancel(actor: Actor, target: {taskId: Id} | {runId: Id}): Promise<void>`（F05）；`decideApproval(actor: Actor, approvalId: Id, decision: "approve" | "deny"): Promise<Approval>`（F07）。这些服务通过所属任务的 HTTP/SSE 路由调用，浏览器不传入 Actor 或 Lease。
- **Produces:** `RoomPage(props: {roomId: Id; branchId: Id}): ReactElement`；`applyStreamEvent(state: RoomView, event: StreamEvent): RoomView`；`submitRoomMessage(input: MessageInput): Promise<{event: RoomEvent; taskId?: Id; runIds: Id[]}>`。RoomView 是按 eventId 去重的持久事件、按 run/attempt 的临时流及 Task 卡片索引。

- [ ] **Step 1: 编写失败测试。** 两个浏览器上下文代表同一房间不同成员；输入普通 @ 文本与通过成员选择器创建的结构化 mention，断开/恢复 SSE，慢消费者切换 tab 后再回来。使用基础平台加确定性 runtime 桩启动测试。

```ts
test("structured mention, steering and reconnect", async ({page}) => {
  await expect(page.getByTestId("task-card")).toHaveCount(0);
  // 通过选择器提交 @ 后，测试驱动后端桩产生一个任务。
  await expect(page.getByTestId("task-card")).toHaveCount(1);
  await expect(page.getByText("补充当前任务")).toBeVisible();
  await expect(page.getByTestId("final-message")).toHaveCount(1);
});
test("cancel pending and partial results are distinct", async ({page}) => {
  await expect(page.getByText("取消中")).toBeVisible();
  await expect(page.getByTestId("restart-task")).toBeDisabled();
  await expect(page.getByText("部分完成")).toBeVisible();
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec playwright test tests/e2e/room.spec.ts`。预期 FAIL，原因应为文本 @ 误触发、断线重复终态或取消中仍可重做；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** api/client.ts 统一认证、幂等键、If-Match、错误码和 SSE 游标；event-store.ts 不把临时 token 作为历史，重试清楚区分 attempt。RoomPage 提供房间成员/Agent、消息选择器、附件、回复、显式 steering 与停止操作；TaskCard 展示排队/等待/blocked/partial，不把取消请求当成功终止。ApprovalDialog 展示具体动作、对象、受众、版本与影响。仅挂载本任务路由，不预建无用途页面。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec playwright test tests/e2e/room.spec.ts` 和 `pnpm typecheck`，预期全部 PASS。由 Playwright 检查桌面与窄屏下长文本、键盘选择 @、审批对象和可访问标签；截图用于审阅实际可读性。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add apps/web/package.json apps/web/index.html apps/web/vite.config.ts apps/web/src/main.tsx apps/web/src/api/client.ts apps/web/src/rooms/RoomPage.tsx apps/web/src/rooms/event-store.ts apps/web/src/tasks/TaskCard.tsx apps/web/src/approvals/ApprovalDialog.tsx playwright.config.ts tests/e2e/room.spec.ts
git commit -m "feat: p01 多人房间与任务交互"
```

<a id="p02"></a>

### Task P02: 双入口 Agent 配置与版本界面

**Depends on:** P01, F03, M01, M06, N02

**Files:**
- Create: `apps/web/src/agents/AgentEditor.tsx`
- Create: `apps/web/src/agents/ModelProfileForm.tsx`
- Create: `apps/web/src/agents/NativeProfileForm.tsx`
- Create: `apps/web/src/agents/ReleaseDiff.tsx`
- Modify: `apps/web/src/main.tsx`
- Test: `tests/e2e/agent-configuration.spec.ts`

**Interfaces:**

- **Consumes:** `saveDraft(actor: Actor, draft: ReleaseDraft): Promise<RevisionRef>`（F03）；`publishRelease(actor: Actor, draft: RevisionRef): Promise<AgentRelease>`（F03）；`bindInstance(actor: Actor, instanceId: Id, releaseId: Id, ifMatch: Revision): Promise<Revision>`（F03）；`diffRelease(actor: Actor, base: Id, target: Id): Promise<ReleaseDiff>`（F03）；`validateModelProfile(actor: Actor, profile: ModelProfile): Promise<RevisionRef>`（M01）；`createPairing(actor: Actor): Promise<{pairingId: Id; expiresAt: ISODate; code: string}>`（N01）；`publishRuntimeProfile(actor: Actor, draft: RuntimeProfileDraft): Promise<RevisionRef>`（N02）。这些服务通过所属任务的 HTTP/SSE 路由调用，浏览器不传入 Actor 或 Lease。
- **Produces:** `AgentEditor(props: {agentId: Id}): ReactElement`；`ModelProfileForm(props: {initial?: ModelProfile; onSave(profile: ModelProfile): Promise<void>}): ReactElement`；`NativeProfileForm(props: {capabilities: CapabilitySnapshot; onSave(draft: RuntimeProfileDraft): Promise<void>}): ReactElement`。

- [ ] **Step 1: 编写失败测试。** 同一用户分别创建 managed 与 native 配置，native 不支持 effort=high 且 memory coverage 为 unknown；发布后已有 r1 任务继续运行，界面更新到 r2。

```ts
test("managed and native forms follow different capabilities", async ({page}) => {
  await expect(page.getByText("模型 API + 自研 Agent")).toBeVisible();
  await expect(page.getByText("已有 Agent")).toBeVisible();
  await expect(page.getByRole("option",{name:"high",exact:true})).toBeDisabled();
  await expect(page.getByText("原生记忆覆盖范围未知")).toBeVisible();
});
test("published edits create revisions", async ({page}) => {
  await expect(page.getByTestId("release-diff")).toContainText("r1 → r2");
  await expect(page.getByTestId("running-task-release")).toHaveText("r1");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec playwright test tests/e2e/agent-configuration.spec.ts`。预期 FAIL，原因应为把两类接入混为 endpoint、提供无效选项或修改旧任务版本；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** 表单按后端能力 schema 生成受支持选项，unsupported、unknown、denied 使用不同文案；不统一解释各提供者 effort。资源编辑覆盖 Prompt/Context/Memory/Skills/Tools/MCP 与 runtime binding，发布差异展示固定与观测资源。已有 Agent 默认个人模式，展示允许调用人、配置来源、工作目录、原生版本及分享策略。临时参数仅写 ResolvedRunConfig，不修改已发布对象；密钥输入只进凭据登记端点，不保存在浏览器持久存储。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec playwright test tests/e2e/agent-configuration.spec.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add apps/web/src/agents/AgentEditor.tsx apps/web/src/agents/ModelProfileForm.tsx apps/web/src/agents/NativeProfileForm.tsx apps/web/src/agents/ReleaseDiff.tsx apps/web/src/main.tsx tests/e2e/agent-configuration.spec.ts
git commit -m "feat: p02 双入口 Agent 配置与版本界面"
```

<a id="p03"></a>

### Task P03: 分支与可协作产物

**Depends on:** F04, F05, M03, P01

**Files:**
- Create: `packages/core/src/branches/service.ts`
- Create: `packages/core/src/artifacts/service.ts`
- Create: `packages/contracts/src/artifacts.ts`
- Create: `packages/infra/migrations/017-branches-artifacts.sql`
- Create: `apps/api/src/routes/artifacts.ts`
- Create: `apps/web/src/artifacts/ArtifactPanel.tsx`
- Create: `apps/web/src/rooms/BranchPicker.tsx`
- Create: `tests/e2e/artifact.spec.ts`
- Modify: `apps/api/src/routes/rooms.ts`
- Modify: `apps/api/src/app.ts`
- Modify: `apps/web/src/rooms/RoomPage.tsx`
- Test: `tests/integration/branches-artifacts.test.ts`

**Interfaces:**

- **Consumes:** `readEvents(actor: Actor, branchId: Id, after: number): Promise<RoomEvent[]>`（F04）；`readMemory(grant: Grant, scope: MemoryScope, atRevision: Revision): Promise<MemoryRevision[]>`（M03）；`authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）。这些服务通过所属任务的 HTTP/SSE 路由调用，浏览器不传入 Actor 或 Lease。
- **Produces:** `forkBranch(actor: Actor, input: {branchId: Id; forkSeq: number}): Promise<{branchId: Id}>`；`commitArtifact(lease: Lease, input: {operationId: Id; content: BlobRef; audience: Audience; expectedRevision?: Revision; artifactId?: Id}): Promise<RevisionRef>`；`editArtifact(actor: Actor, ref: RevisionRef, content: BlobRef): Promise<RevisionRef>`。

- [ ] **Step 1: 编写失败测试。** 从线索公布之前分叉，原分支之后有新记忆；两个 Run 同时修改同一 expectedRevision 的产物，其中一个写入成功。新成员无来源项目权限。

```ts
it("branch inherits only history and memory before fork", () => {
  expect(branchEvents).not.toContainEqual(futureClue);
  expect(branchMemories).not.toContainEqual(futureMemory);
});
it("concurrent edits preserve the winning revision", () => {
  expect(secondEdit.code).toBe("REVISION_CONFLICT");
  expect(currentArtifact.revision).toBe(firstEdit.revision);
  expect(unprivilegedRead.code).toBe("FORBIDDEN");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/branches-artifacts.test.ts`。预期 FAIL，原因应为未来信息回流旧分支、并发覆盖或产物脱离来源 ACL；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** branches/service.ts 固定 forkSeq 与记忆视图；WorldState 继承由 P04 引入该状态时接入。跨分支个人资料必须显式标记。artifacts/service.ts 用稳定 operationId、预期 revision 和来源 ACL 提交，冲突不得覆盖；结果预览/下载每次检查当前来源授权。注册 branches、artifacts 查询/编辑/下载 API，并在房间加入分支选择和版本产物面板。UI 冲突提示重新读取并裁决，不自动用最后保存覆盖。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/branches-artifacts.test.ts` 和 `pnpm typecheck`，预期全部 PASS。增加 `tests/e2e/artifact.spec.ts` 的面板冲突与分支切换验收，并执行 `pnpm exec playwright test tests/e2e/artifact.spec.ts`。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/branches/service.ts packages/core/src/artifacts/service.ts packages/contracts/src/artifacts.ts packages/infra/migrations/017-branches-artifacts.sql apps/api/src/routes/artifacts.ts apps/web/src/artifacts/ArtifactPanel.tsx apps/web/src/rooms/BranchPicker.tsx tests/e2e/artifact.spec.ts apps/api/src/routes/rooms.ts apps/api/src/app.ts apps/web/src/rooms/RoomPage.tsx tests/integration/branches-artifacts.test.ts
git commit -m "feat: p03 分支与可协作产物"
```

<a id="p04"></a>

### Task P04: 角色场景与严格剧本规则

**Depends on:** F03, F08, M08, P03

**Files:**
- Create: `packages/core/src/scenes/releases.ts`
- Create: `packages/core/src/scenes/engine.ts`
- Create: `packages/core/src/scenes/scheduler.ts`
- Create: `packages/contracts/src/scenes.ts`
- Create: `packages/infra/migrations/018-scenes.sql`
- Create: `apps/api/src/routes/scenes.ts`
- Create: `apps/web/src/scenes/ScenePanel.tsx`
- Create: `tests/fixtures/scenes/mystery.json`
- Create: `tests/fixtures/scenes/roleplay.json`
- Modify: `packages/core/src/branches/service.ts`
- Modify: `apps/api/src/app.ts`
- Modify: `apps/web/src/rooms/RoomPage.tsx`
- Test: `tests/integration/scenes.test.ts`

**Interfaces:**

- **Consumes:** `startTask(actor: Actor, input: TaskInput): Promise<{taskId: Id; runIds: Id[]}>`（F05）；`publishRelease(actor: Actor, draft: RevisionRef): Promise<AgentRelease>`（F03）。这些服务通过所属任务的 HTTP/SSE 路由调用，浏览器不传入 Actor 或 Lease。
- **Produces:** `publishScene(actor: Actor, draft: SceneDraft): Promise<RevisionRef>`；`applyWorldAction(lease: Lease, input: {scene: RevisionRef; expectedRevision: Revision; action: WorldAction}): Promise<{worldRevision: Revision; publicEventRefs: RevisionRef[]}>`；`scheduleRound(actor: Actor, scene: RevisionRef, branchId: Id): Promise<Id[]>`。SceneDraft 固定角色版本、私密/公开实例、规则、轮次参与者/上限/结束条件；WorldAction 为场景 schema 的结构化线索/移动/投票动作。

- [ ] **Step 1: 编写失败测试。** 严格剧本私密规划器知道凶手，公开发言器只能收到规则解锁字段；模型生成“我已拿到线索”但未提交合法动作。两个投票动作竞争相同世界 revision，角色扮演使用有界轮流发言。

```ts
it("strict mystery never sends hidden truth to public speaker", () => {
  expect(publicModelInput).not.toContain("甲是凶手");
  expect(publicTokenStream).not.toContain(privatePlannerText);
  expect(nativeStrictRoleBinding.code).toBe("CAPABILITY_UNSUPPORTED");
});
it("rules own world state and bounded rounds", () => {
  expect(worldAfterNarration).toEqual(worldBeforeNarration);
  expect(forkedWorld.revision).toBe(worldRevisionAtFork);
  expect(conflictingVote.code).toBe("REVISION_CONFLICT");
  expect(roundsExecuted).toBe(configuredRoundLimit);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/scenes.test.ts`。预期 FAIL，原因应为隐藏真相进入公开输入、叙述直接改权威状态或自动发言无界；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** releases.ts 分别绑定私密 planner/public speaker 的 managed AgentInstance；engine.ts 校验动作、授权、阶段和 expectedRevision 后更新 WorldState，只输出允许公开字段。scheduler.ts 支持 mention_only、轮流和主持人选择的有界模式，每轮仍创建有服务身份授权和统一预算的 Task；主持 Agent 仅提出建议。ScenePanel 展示角色、公开线索、阶段、投票和合法动作，私密信息按角色权限单独渲染。将 WorldState 的分叉点视图接入 P03 的 branch service，旧分支不能看到后续世界状态。SceneRelease 中途不变。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/scenes.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/scenes/releases.ts packages/core/src/scenes/engine.ts packages/core/src/scenes/scheduler.ts packages/contracts/src/scenes.ts packages/infra/migrations/018-scenes.sql apps/api/src/routes/scenes.ts apps/web/src/scenes/ScenePanel.tsx tests/fixtures/scenes/mystery.json tests/fixtures/scenes/roleplay.json packages/core/src/branches/service.ts apps/api/src/app.ts apps/web/src/rooms/RoomPage.tsx tests/integration/scenes.test.ts
git commit -m "feat: p04 角色场景与严格剧本规则"
```

<a id="p05"></a>

### Task P05: 办公协作与运行面板

**Depends on:** P02, P03, M08, N08, O01

**Files:**
- Create: `apps/web/src/runs/RunInspector.tsx`
- Create: `apps/web/src/runs/InputSources.tsx`
- Create: `apps/web/src/runs/NativeSessionPanel.tsx`
- Create: `apps/web/src/office/OfficeRoom.tsx`
- Create: `tests/fixtures/office/review-brief.md`
- Modify: `apps/web/src/tasks/TaskCard.tsx`
- Modify: `apps/web/src/main.tsx`
- Test: `tests/e2e/office-collaboration.spec.ts`

**Interfaces:**

- **Consumes:** `getRun(actor: Actor, runId: Id): Promise<RunSpec>`（F05）；`readRunTrace(actor: Actor, runId: Id): Promise<RunTrace>`（O01）；`readUsage(actor: Actor, scope: {roomId?: Id; releaseId?: Id; taskId?: Id}): Promise<UsageReport>`（O01）；`reconcileNative(actor: Actor, runId: Id): Promise<NativeObservation>`（N07）；`requestCancel(actor: Actor, target: {taskId: Id} | {runId: Id}): Promise<void>`（F05）。这些服务通过所属任务的 HTTP/SSE 路由调用，浏览器不传入 Actor 或 Lease。
- **Produces:** `RunInspector(props: {runId: Id}): ReactElement`；`OfficeRoom(props: {roomId: Id; branchId: Id}): ReactElement`。面板消费统一 RunView，保留 managed 的输入快照与 native ContextEnvelope/coverage 的差别。

- [ ] **Step 1: 编写失败测试。** 办公资料包含来源 revision，用户 @ 分析 Agent 委派本地 Codex 产出评审，再由人工审批发布私密结果；另一个必需 Agent 失败使 Task 为 partial。外部提交工具使用模拟仓库，不执行真实 Git 操作。

```ts
test("office work preserves sources and partial results", async ({page}) => {
  await expect(page.getByTestId("artifact-sources")).toContainText("review-brief.md");
  await expect(page.getByText("部分完成")).toBeVisible();
  await expect(page.getByTestId("completed-artifact")).toBeVisible();
});
test("runtime inspector reports actual coverage", async ({page}) => {
  await expect(page.getByTestId("native-cost")).toHaveText("unknown");
  await expect(page.getByText("平台发送的上下文")).toBeVisible();
  await expect(page.getByText("等待结果分享批准")).toBeVisible();
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec playwright test tests/e2e/office-collaboration.spec.ts`。预期 FAIL，原因应为隐藏失败、把 unknown 显示为零费用或给危险的通用重试入口；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** RunInspector 展示发布/实际模型、来源、工具、审批、委派、耗时和费用，缺失字段显示来源与原因；Prompt/原始工具结果权限独立于公开回复。NativeSessionPanel 显示设备、会话、投递、漂移、取消确认和允许的恢复动作；unknown 不给通用重试按钮。OfficeRoom 提供资料→讨论→结构化结论/需求产物的模板，不建立独立办公调度系统。需求开发中的写入动作继续走工具审批，首版不自动合并或发送外部消息。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec playwright test tests/e2e/office-collaboration.spec.ts` 和 `pnpm typecheck`，预期全部 PASS。用两个成员身份检查公开产物与私密输入面板的权限差异；检查长任务树和窄屏下可读性。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add apps/web/src/runs/RunInspector.tsx apps/web/src/runs/InputSources.tsx apps/web/src/runs/NativeSessionPanel.tsx apps/web/src/office/OfficeRoom.tsx tests/fixtures/office/review-brief.md apps/web/src/tasks/TaskCard.tsx apps/web/src/main.tsx tests/e2e/office-collaboration.spec.ts
git commit -m "feat: p05 办公协作与运行面板"
```

