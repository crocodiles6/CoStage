# CoStage 基础平台 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让授权用户可靠提交消息和任务，并获得可恢复、可审计的业务状态。

**Architecture:** 领域服务使用同一 TypeScript 代码库和 PostgreSQL 事务。API、Worker 按进程部署，业务事实通过 Outbox 发布；执行后端只调用这些受授权服务。

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

- F02：用户同名、tenant 不同以及成员刚被撤销，不能借旧 grant 读数据。
- F04：慢客户端丢失临时 token 后重连，最终事件必须完整。
- F05：并发根 Run 与重试争用最后预算，累计不得超过 24 次模型调用。
- F06：失去租约的 Worker 晚提交，业务写入必须被拒绝。
- F08：子结果早到与跨 Task 的原生资源环，分别只唤醒一次和明确拒绝。

---

## 文件与执行约定

`apps/api` 只做认证、参数校验、路由和推送；`packages/core/src/<领域>` 持有业务规则；`packages/infra` 持有数据库与对象存储。每个领域迁移与接口在同一任务交付，精确文件列表见各任务。所有这些代码路径目前均不存在，Create 表示未来新建；Modify 只指本计划前序任务已经创建的文件。

本计划可用受控模型桩独立验收，不要求先接真实模型或原生 Agent。F01 先把测试环境跑通，再进入失败测试；后续任务复用同一环境。

<a id="f01"></a>

### Task F01: 可验证的服务与存储入口

**Depends on:** 无；从当前仅有文档的仓库开始。

**Files:**
- Create: `package.json`
- Create: `pnpm-workspace.yaml`
- Create: `pnpm-lock.yaml`
- Create: `tsconfig.base.json`
- Create: `vitest.config.ts`
- Create: `apps/api/package.json`
- Create: `apps/api/src/app.ts`
- Create: `packages/contracts/package.json`
- Create: `packages/contracts/src/common.ts`
- Create: `packages/infra/package.json`
- Create: `packages/infra/src/db.ts`
- Create: `packages/infra/src/migrate.ts`
- Create: `packages/infra/src/blob-store.ts`
- Create: `packages/testkit/package.json`
- Create: `packages/testkit/src/database.ts`
- Create: `compose.test.yaml`
- Create: `docs/development.md`

- Test: `tests/integration/foundation.test.ts`

**Interfaces:**

- **Consumes:** Node.js、Fastify 与 PostgreSQL 的公开 API；当前没有可复用应用代码。
- **Produces:** `migrate(databaseUrl: string): Promise<void>`；`buildApp(): Promise<FastifyInstance>`；`transaction<T>(tenantId: Id, work: (tx: Tx) => Promise<T>): Promise<T>`；`putBlob(bytes: Uint8Array, mediaType: string): Promise<BlobRef>`；`getBlob(ref: BlobRef): Promise<Uint8Array>`；公共类型见总入口。测试工具 `createTestDatabase(): Promise<{ url: string; dispose(): Promise<void> }>` 每次创建独立数据库。

- [ ] **Step 1: 编写失败测试。** 先创建本任务测试配置和空导出，使测试能启动。用 compose.test.yaml 启动 pgvector PostgreSQL 与 S3 兼容对象存储；健康接口必须区分存活与数据库就绪。测试服务不使用生产凭据。

```ts
it("readiness reflects database availability", async () => {
  expect((await app.inject({url:"/health/live"})).statusCode).toBe(200);
  expect((await app.inject({url:"/health/ready"})).statusCode).toBe(503);
});
it("blob round trip preserves bytes", async () => {
  const bytes = new TextEncoder().encode("CoStage");
  expect(await getBlob(await putBlob(bytes,"text/plain"))).toEqual(bytes);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/foundation.test.ts`。预期 FAIL，原因应为ready 错报成功或对象存储无法按引用读回；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** 在 app.ts 提供健康端点；db.ts 用 pg 和显式 SQL 事务，在事务内设置 tenant；blob-store.ts 提供内容摘要与存储引用。启用 strict TS，根目录 typecheck 脚本检查当时已存在的工作区与测试。锁定 Node.js 24 的受支持补丁版本及 pnpm 精确版本，提交 lockfile，不追随 latest。Vitest 的 integration 项目从 TEST_DATABASE_URL 和对象存储测试变量读取配置；测试迁移使用独立管理连接，应用连接用非 superuser、无 BYPASSRLS 的角色。migrate.ts 在管理连接上按文件名执行尚未应用的 SQL 并记录迁移版本；createTestDatabase 在隔离测试库应用当时已有的全部迁移。此任务不预建尚未使用的应用。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/foundation.test.ts` 和 `pnpm typecheck`，预期全部 PASS。在 docs/development.md 写明 `docker compose -f compose.test.yaml up -d --wait`、`pnpm install --frozen-lockfile` 以及 `pnpm exec tsx packages/infra/src/migrate.ts --database-env TEST_DATABASE_URL`；测试清理只删除本次随机命名的测试数据库。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add package.json pnpm-workspace.yaml pnpm-lock.yaml tsconfig.base.json vitest.config.ts apps/api/package.json apps/api/src/app.ts packages/contracts/package.json packages/contracts/src/common.ts packages/infra/package.json packages/infra/src/db.ts packages/infra/src/migrate.ts packages/infra/src/blob-store.ts packages/testkit/package.json packages/testkit/src/database.ts compose.test.yaml docs/development.md tests/integration/foundation.test.ts
git commit -m "feat: f01 可验证的服务与存储入口"
```

<a id="f02"></a>

### Task F02: 空间身份与授权交集

**Depends on:** F01

**Files:**
- Create: `packages/core/package.json`
- Create: `packages/core/src/auth/access.ts`
- Create: `packages/core/src/auth/invitations.ts`
- Create: `packages/contracts/src/auth.ts`
- Create: `packages/infra/migrations/001-tenancy.sql`
- Create: `apps/api/src/routes/workspaces.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/authorization.test.ts`

**Interfaces:**

- **Consumes:** `transaction<T>(tenantId: Id, work: (tx: Tx) => Promise<T>): Promise<T>`（F01）。
- **Produces:** `authenticate(jwt: string): Promise<Actor>`；`authorize(actor: Actor, request: AccessRequest): Promise<Grant>`；`intersectGrant(parent: Grant, child: AccessRequest): Grant`；`invite(actor: Actor, input: {email: string; role: WorkspaceRole; expiresAt: ISODate}): Promise<{inviteId: Id}>`；`acceptInvite(actor: Actor, token: string): Promise<void>`。AccessRequest / Grant 字段见总入口。

- [ ] **Step 1: 编写失败测试。** 种子为两个 tenant、同名用户及两个房间；成员甲有一个项目权限，成员乙无此权限。构造篡改 user_id 的请求、非应用管理员数据库连接及已撤销成员的旧 grant。

```ts
it("tenant and current membership limit reads", async () => {
  await expect(authorize(actorA, foreignResource)).rejects.toMatchObject({code:"FORBIDDEN"});
  expect(await tenantBConnection.query("select * from workspace_memberships")).toMatchObject({rows:[]});
  await expect(authorize(revokedActor, roomRead)).rejects.toMatchObject({code:"FORBIDDEN"});
});
it("delegation cannot widen credentials or audience", () => {
  expect(intersectGrant(parentGrant, childRequest).credentialSubject).toBe(parentGrant.credentialSubject);
  expect(intersectGrant(parentGrant, childRequest).audience).toEqual(narrowAudience);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/authorization.test.ts`。预期 FAIL，原因应为跨租户可读、撤权后旧 grant 仍获准或委派扩大权限；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** auth/access.ts 校验 OIDC 的 issuer、audience、签名和有效期，从 token 主体解析用户，不接收客户端指定执行身份。迁移建立 Workspace、Membership、Invitation、资源 ACL 及包含 tenant 的复合键/RLS；空间管理员管理资源但不自动读取私密正文或密钥。invitations.ts 实现有期限、只能接受一次的邀请，路由为 POST /workspaces、POST /workspaces/:id/invitations、POST /invitations/:id/accept；授权拒绝返回 FORBIDDEN。grant 使用允许集合及当前权限求交，不通过文本标签表达授权。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/authorization.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/package.json packages/core/src/auth/access.ts packages/core/src/auth/invitations.ts packages/contracts/src/auth.ts packages/infra/migrations/001-tenancy.sql apps/api/src/routes/workspaces.ts apps/api/src/app.ts tests/integration/authorization.test.ts
git commit -m "feat: f02 空间身份与授权交集"
```

<a id="f03"></a>

### Task F03: 不可变资源与 Agent 发布

**Depends on:** F02

**Files:**
- Create: `packages/core/src/releases/service.ts`
- Create: `packages/core/src/releases/manifest.ts`
- Create: `packages/contracts/src/releases.ts`
- Create: `packages/infra/migrations/002-releases.sql`
- Create: `apps/api/src/routes/releases.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/releases.test.ts`

**Interfaces:**

- **Consumes:** `authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）；`putBlob(bytes: Uint8Array, mediaType: string): Promise<BlobRef>`（F01）。
- **Produces:** `saveResource(actor: Actor, input: {kind: ResourceKind; content: BlobRef; schemaVersion: string; replaces?: RevisionRef}): Promise<RevisionRef>`；`saveDraft(actor: Actor, draft: ReleaseDraft): Promise<RevisionRef>`；`publishRelease(actor: Actor, draft: RevisionRef): Promise<AgentRelease>`；`bindInstance(actor: Actor, instanceId: Id, releaseId: Id, ifMatch: Revision): Promise<Revision>`；`diffRelease(actor: Actor, base: Id, target: Id): Promise<ReleaseDiff>`。ReleaseDraft 定义 prompt/context/memory/skill/tool/MCP/model/loop/runtime 的固定 RevisionRef；AgentRelease 增加 releaseId、manifestDigest、status；ReleaseDiff 为变更路径与前后摘要列表。

- [ ] **Step 1: 编写失败测试。** 发布草稿引用两版 Prompt、完整 Skill 包和同名但不同 digest 的 MCP schema；发布 r1 后改绑实例到 r2，保留读取 r1 的任务夹具。测试同时包含 draft→validated→staged→published→retired 状态和凭据引用。

```ts
it("release is immutable and has no floating dependency", async () => {
  await expect(publishRelease(actor, draftWithLatest)).rejects.toMatchObject({code:"INVALID_RELEASE"});
  expect((await publishRelease(actor, fixedDraft)).manifestDigest).toBe(expectedDigest);
});
it("binding uses optimistic concurrency", async () => {
  await expect(bindInstance(actor, instanceId, r2, staleRevision)).rejects.toMatchObject({code:"REVISION_CONFLICT"});
  expect(oldTask.releaseMap[instanceId]).toBe(r1);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/releases.test.ts`。预期 FAIL，原因应为浮动依赖被接受或过期 If-Match 覆盖新绑定；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** service.ts 提供资源修订、草稿 CRUD、校验、发布、退役及差异；POST /resources/:kind/revisions 按 ResourceKind 校验内容 schema 和创建权限，工具/MCP/可执行 Skill 仅管理员可登记；manifest.ts 规范化并固定所有依赖和 schema_version。密钥仅保留 credential_ref。RuntimeReleaseBinding 定义为 managed（loopRevision、modelProfileRevision）或 native（deviceId、profileRevision）互斥联合；初期以测试能力目录校验，M01/M05/N02 接入真实目录。注册 POST /agents、PUT /agents/:id/draft、POST /agents/:id/releases、GET /releases/:id/diff 与实例 If-Match 路由。仅通过受权限控制的 API 修改绑定，回滚不修改既有 Run、记忆或外部效果。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/releases.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/releases/service.ts packages/core/src/releases/manifest.ts packages/contracts/src/releases.ts packages/infra/migrations/002-releases.sql apps/api/src/routes/releases.ts apps/api/src/app.ts tests/integration/releases.test.ts
git commit -m "feat: f03 不可变资源与 Agent 发布"
```

<a id="f04"></a>

### Task F04: 房间消息与可恢复事件流

**Depends on:** F02, F03

**Files:**
- Create: `packages/core/src/rooms/messages.ts`
- Create: `packages/core/src/rooms/stream.ts`
- Create: `packages/contracts/src/rooms.ts`
- Create: `packages/infra/migrations/003-rooms.sql`
- Create: `apps/api/src/routes/rooms.ts`
- Create: `apps/api/src/outbox.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/room-stream.test.ts`

**Interfaces:**

- **Consumes:** `authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）；`transaction<T>(tenantId: Id, work: (tx: Tx) => Promise<T>): Promise<T>`（F01）。
- **Produces:** `createRoom(actor: Actor, input: {name: string; sceneReleaseId?: Id}): Promise<{roomId: Id; branchId: Id}>`；`appendMessage(tx: Tx, actor: Actor, input: MessageInput): Promise<RoomEvent>`；`readEvents(actor: Actor, branchId: Id, after: number): Promise<RoomEvent[]>`；`streamEvents(actor: Actor, branchId: Id, after: number): AsyncIterable<StreamEvent>`。RoomEvent 与临时 delta 的互斥字段见总入口。

- [ ] **Step 1: 编写失败测试。** 同一分支并发提交相同幂等键及不同 payload；另有角色私密事件、慢 SSE 客户端和新加入的无项目授权成员。结构化 mentions 留给 F05，纯文本中的 @ 始终只是正文。

```ts
it("same key returns original event and changed payload conflicts", async () => {
  expect(second.eventId).toBe(first.eventId);
  expect(conflict.statusCode).toBe(409);
});
it("reconnect keeps final events and hides private history", async () => {
  expect((await readEvents(newMember, branchId, cursor)).map(e=>e.eventId)).toEqual(publicEventIds);
  expect(slowClient.finalEventIds).toEqual(publicEventIds);
  expect(new Set(concurrentSeqs).size).toBe(concurrentSeqs.length);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/room-stream.test.ts`。预期 FAIL，原因应为重复消息、分支序号冲突、漏最终事件或泄露私密历史；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** messages.ts 在短事务分配 branch seq，保存正文 revision、ACL、因果关系和 Outbox。stream.ts 实现持久游标补拉、提交后广播和逐事件当前权限过滤；临时 chunk 不写入房间历史，按 run/attempt/chunkSeq 区分，慢客户端可丢 delta。提供 POST /rooms、成员管理、GET /rooms/:id/events、GET /rooms/:id/stream，后者携带 branchId 和 Last-Event-ID。消息 POST 在 F05 接入原子任务创建。默认参与者/Agent/并发限制分别为 20/8/4，纯人聊天可独立验收。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/room-stream.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/rooms/messages.ts packages/core/src/rooms/stream.ts packages/contracts/src/rooms.ts packages/infra/migrations/003-rooms.sql apps/api/src/routes/rooms.ts apps/api/src/outbox.ts apps/api/src/app.ts tests/integration/room-stream.test.ts
git commit -m "feat: f04 房间消息与可恢复事件流"
```

<a id="f05"></a>

### Task F05: 请求到 Task 与统一预算

**Depends on:** F03, F04

**Files:**
- Create: `packages/core/src/tasks/coordinator.ts`
- Create: `packages/core/src/tasks/budget.ts`
- Create: `packages/core/src/tasks/aggregate.ts`
- Create: `packages/contracts/src/tasks.ts`
- Create: `packages/infra/migrations/004-tasks.sql`
- Create: `apps/api/src/routes/tasks.ts`
- Modify: `apps/api/src/routes/rooms.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/task-creation.test.ts`

**Interfaces:**

- **Consumes:** `appendMessage(tx: Tx, actor: Actor, input: MessageInput): Promise<RoomEvent>`（F04）；`authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）。
- **Produces:** `startTask(actor: Actor, input: TaskInput): Promise<{taskId: Id; runIds: Id[]}>`；`submitMessage(actor: Actor, input: MessageInput): Promise<{event: RoomEvent; taskId?: Id; runIds: Id[]}>`；`getRun(actor: Actor, runId: Id): Promise<RunSpec>`；`reserveUsage(scope: RunRef | SystemJobRef, operationId: Id, amount: UsageReservation): Promise<{reservationId: Id}>`；`settleUsage(reservationId: Id, actual: UsageAmount): Promise<void>`；`aggregateTask(taskId: Id): Promise<TaskStatus>`；`steer(actor: Actor, taskId: Id, delta: RevisionRef): Promise<number>`；`requestCancel(actor: Actor, target: {taskId: Id} | {runId: Id}): Promise<void>`。

- [ ] **Step 1: 编写失败测试。** 两个合法 mentions 只创建一个 Task 和两个根 Run；请求重放、引用 @、文本 @、伪造实例 ID、并发预算预留、required 子失败及 optional 子失败分别建例。截止时间包含队列和等待。

```ts
it("only structured mentions create roots atomically", async () => {
  expect(plain.runIds).toEqual([]);
  expect(mentioned.runIds).toHaveLength(2);
  expect(repeated.taskId).toBe(mentioned.taskId);
  expect(savedTask.baseSeq).toBe(mentioned.event.seq);
});
it("attempts and children share limits", () => {
  expect(acceptedModelReservations).toBe(24);
  expect(overLimit.code).toBe("BUDGET_EXHAUSTED");
  expect(requiredFailureStatus).toBe("partial");
  expect(nativeUnconfirmedCancellation).toBe("blocked");
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/task-creation.test.ts`。预期 FAIL，原因应为文本触发任务、重放创建新 Run、预算超过 24 次或过早汇总成功；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** coordinator.ts 在消息、Task、根 Run、RunJob、Outbox 的同一事务内保存 release_map、SceneRelease、base_seq、ExecutionGrant、deadline 和预算。RunSpec 固定输入与运行绑定。budget.ts 以 operationId 幂等预留，网络新 attempt 消耗模型次数，读取已存结果不消耗；含摘要/embedding/记忆抽取。aggregate.ts 以必需目标完成事实聚合，不信任模型宣称成功；optional 失败只警告，native 静止未确认时保留 blocked。steering 保存有序增量；取消先冻结新动作，由各后端确认终止。路由覆盖设计的 messages、tasks/runs 查询、steering 与 cancel。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/task-creation.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/tasks/coordinator.ts packages/core/src/tasks/budget.ts packages/core/src/tasks/aggregate.ts packages/contracts/src/tasks.ts packages/infra/migrations/004-tasks.sql apps/api/src/routes/tasks.ts apps/api/src/routes/rooms.ts apps/api/src/app.ts tests/integration/task-creation.test.ts
git commit -m "feat: f05 请求到 Task 与统一预算"
```

<a id="f06"></a>

### Task F06: 公平队列与租约隔离

**Depends on:** F05

**Files:**
- Create: `apps/worker/package.json`
- Create: `apps/worker/src/main.ts`
- Create: `packages/core/src/jobs/leases.ts`
- Create: `packages/core/src/jobs/scheduler.ts`
- Create: `packages/core/src/jobs/action-guard.ts`
- Create: `packages/infra/migrations/005-jobs.sql`

- Test: `tests/integration/leases.test.ts`

**Interfaces:**

- **Consumes:** `getRun(actor: Actor, runId: Id): Promise<RunSpec>`（F05）；`requestCancel(actor: Actor, target: {taskId: Id} | {runId: Id}): Promise<void>`（F05）；`authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）。
- **Produces:** `claimSystemJob(workerId: Id, kind: "document_index" | "memory_cleanup"): Promise<SystemLease | null>`；`withSystemFence<T>(lease: SystemLease, work: (tx: Tx) => Promise<T>): Promise<T>`；`claimJob(workerId: Id, supportedRuntimeRevisions: Revision[]): Promise<Lease | null>`；`renewLease(lease: Lease): Promise<Lease>`；`withFence<T>(lease: Lease, work: (tx: Tx) => Promise<T>): Promise<T>`；`guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`；`releaseSlot(lease: Lease): Promise<void>`；`commitRunOutcome(lease: Lease, outcome: RunOutcome): Promise<void>`。

- [ ] **Step 1: 编写失败测试。** 两个 Worker 竞争同一实例；旧 Worker 暂停后租约被接管，再提交状态。三个 tenant 中一个持续灌入恢复作业；用可控时钟检验取消和租约到期。

```ts
it("stale epoch cannot commit after takeover", async () => {
  await expect(withFence(oldLease, staleWrite)).rejects.toMatchObject({code:"STALE_LEASE"});
  expect(activeInstanceSegments).toBe(1);
  expect(maxRoomSegments).toBe(4);
});
it("resumed work does not starve other tenants", () => {
  expect(firstSchedulingRoundTenants).toEqual(["a","b","c"]);
  expect(newActionsAfterCancelAtPlus2s).toBe(0);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/leases.test.ts`。预期 FAIL，原因应为旧租约可写入或繁忙 tenant 导致其他 tenant 永久等待；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** leases.ts 使用 PostgreSQL 行锁领取 RunJob，lease epoch 单调增长；同一调度模块为附件索引与记忆清理提供独立 SystemJob 队列及 SystemLease，不伪装成 Agent Run；withFence 在执行业务写入的同一事务内锁定并校验当前租约。scheduler.ts 按 tenant 轮转，再按可执行时间和入队序号排序，限制房间 4 个片段和每实例 1 个片段；managed 等待释放，native 逻辑占用由 N07 管理。commitRunOutcome 在同一 fencing 事务提交终态、结果引用与 Outbox，再触发 Task 聚合。guardAction 每次新模型/工具/委派前读取当前授权、取消、deadline；预算预留仍由 F05 负责。租约初值建议 15 秒、心跳 5 秒、扫描 5 秒，属于实现默认值，O04 在目标负载下校验 60 秒接管目标。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/leases.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add apps/worker/package.json apps/worker/src/main.ts packages/core/src/jobs/leases.ts packages/core/src/jobs/scheduler.ts packages/core/src/jobs/action-guard.ts packages/infra/migrations/005-jobs.sql tests/integration/leases.test.ts
git commit -m "feat: f06 公平队列与租约隔离"
```

<a id="f07"></a>

### Task F07: 冻结动作的审批服务

**Depends on:** F05, F06

**Files:**
- Create: `packages/core/src/approvals/service.ts`
- Create: `packages/contracts/src/approvals.ts`
- Create: `packages/infra/migrations/006-approvals.sql`
- Create: `apps/api/src/routes/approvals.ts`
- Modify: `apps/api/src/app.ts`
- Test: `tests/integration/approvals.test.ts`

**Interfaces:**

- **Consumes:** `guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`（F06）；`authorize(actor: Actor, request: AccessRequest): Promise<Grant>`（F02）。
- **Produces:** `prepareApproval(lease: Lease, input: ApprovalInput): Promise<Approval>`；`decideApproval(actor: Actor, approvalId: Id, decision: "approve" | "deny"): Promise<Approval>`；`requireApproval(lease: Lease, approvalId: Id, expected: ApprovalInput): Promise<void>`。ApprovalInput 是 platform_tool、native_action、result_share 互斥联合，均含 operationId、对象/版本、argsDigest 或 resultRevision、audience、expiresAt；Approval 增加 id、决策人与状态。

- [ ] **Step 1: 编写失败测试。** 批准后修改参数、撤销成员、到期、审批对象类型替换和并发批准/拒绝分别建例；result_share 精确绑定私密产物 revision 与目标受众。

```ts
it("approval cannot be reused for changed action", async () => {
  await expect(requireApproval(lease, approvalId, changedArgs)).rejects.toMatchObject({code:"APPROVAL_MISMATCH"});
  await expect(requireApproval(lease, approvalId, revokedAction)).rejects.toMatchObject({code:"FORBIDDEN"});
});
it("sharing consent is revision and audience specific", () => {
  expect(widerAudienceDecision.code).toBe("APPROVAL_MISMATCH");
  expect(expiresAt).toBe(taskDeadline);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/approvals.test.ts`。预期 FAIL，原因应为旧批准放行新参数、新受众或已撤权动作；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** service.ts 保存规范化动作摘要、权限主体和有效期，过期时间取自身期限与 Task deadline 的较早者。POST /approvals/:id/decisions 支持幂等同一决定并拒绝相反决定；GET /approvals/:id 返回有权限查看的动作摘要。执行前 requireApproval 校验当前权限和完全相同对象，不把等待阶段结果作为动作结果。拒绝保留明确未执行结果。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/approvals.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/approvals/service.ts packages/contracts/src/approvals.ts packages/infra/migrations/006-approvals.sql apps/api/src/routes/approvals.ts apps/api/src/app.ts tests/integration/approvals.test.ts
git commit -m "feat: f07 冻结动作的审批服务"
```

<a id="f08"></a>

### Task F08: 结构化委派与持久等待账本

**Depends on:** F05, F06, F07

**Files:**
- Create: `packages/core/src/delegation/service.ts`
- Create: `packages/core/src/delegation/waits.ts`
- Create: `packages/core/src/delegation/resource-graph.ts`
- Create: `packages/contracts/src/delegation.ts`
- Create: `packages/infra/migrations/007-delegation.sql`

- Test: `tests/integration/delegation.test.ts`

**Interfaces:**

- **Consumes:** `guardAction(lease: ActionLease, request: AccessRequest): Promise<Grant>`（F06）；`intersectGrant(parent: Grant, child: AccessRequest): Grant`（F02）；`aggregateTask(taskId: Id): Promise<TaskStatus>`（F05）。
- **Produces:** `delegate(lease: Lease, input: DelegateInput): Promise<{childRunId: Id; waitId: Id}>`；`recordChildResult(childRunId: Id, outcome: RunOutcome): Promise<void>`；`markWaitReady(lease: Lease, waitId: Id, evidence: WaitEvidence): Promise<void>`；`takeResume(lease: Lease, waitId: Id): Promise<ResumeEnvelope | null>`；`ackResume(lease: Lease, envelopeId: Id): Promise<void>`；`reserveResources(lease: Lease, resources: string[], waitsFor: Id[]): Promise<void>`。DelegateInput/WaitEvidence 定义见总入口。

- [ ] **Step 1: 编写失败测试。** 安排 child-first、ready-first、重复事件和 resume 入队后崩溃四种顺序；构造同链祖先循环及不同 Task 互占 native session/workdir 的环。

```ts
it.each(["child-first","ready-first","duplicate-result"])("%s resumes once", () => {
  expect(resumeEnvelopes).toHaveLength(1);
  expect(consumerAcknowledgedBeforeAcceptance).toBe(false);
});
it("cycles and shared task limits stop expansion", () => {
  expect(ancestorCycle.code).toBe("DELEGATION_CYCLE");
  expect(crossTaskResourceCycle.code).toBe("RESOURCE_CYCLE");
  expect([maxDepth,maxFanout,maxTaskRuns]).toEqual([3,3,12]);
});
```

- [ ] **Step 2: 验证测试确实失败。** 运行 `pnpm exec vitest run tests/integration/delegation.test.ts`。预期 FAIL，原因应为丢失早到结果、重复恢复、提前确认消费或形成资源死锁；环境连接、语法或缺依赖报错不算有效红灯。

- [ ] **Step 3: 实现本任务接口。** service.ts 事务保存子 Run、Delegation 和 WaitRegistration，继承固定 Task release_map 并收窄 grant；新增实例不得混入旧 Task。waits.ts 用 run_id+wait_id+resume_generation 唯一约束形成 ResumeEnvelope 与恢复作业，等待证据必须来自对应后端；结果先到只保存，不假定已暂停。ackResume 仅在接纳阶段业务提交后写入。resource-graph.ts 在事务锁下检查跨 Task 资源等待图，形成环即拒绝；不自动复制个人 profile。取消后的结果仍记事实但不唤醒父级。

- [ ] **Step 4: 验证交付行为。** 运行 `pnpm exec vitest run tests/integration/delegation.test.ts` 和 `pnpm typecheck`，预期全部 PASS。

- [ ] **Step 5: 提交本任务。** 只提交下列文件及本任务实际生成的锁文件变更；提交前审阅差异。

```bash
git add packages/core/src/delegation/service.ts packages/core/src/delegation/waits.ts packages/core/src/delegation/resource-graph.ts packages/contracts/src/delegation.ts packages/infra/migrations/007-delegation.sql tests/integration/delegation.test.ts
git commit -m "feat: f08 结构化委派与持久等待账本"
```

