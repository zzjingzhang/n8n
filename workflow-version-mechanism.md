# n8n Workflow Draft/Publish 版本机制深度解析

## 一、核心数据模型

n8n 的 workflow 系统采用 **双轨存储模型**：草稿内容与发布版本分离存储，互不干扰。

```
┌─────────────────────────────────────────────────────────────────┐
│ WorkflowEntity (workflow_entity 表)                            │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ nodes, connections, nodeGroups  ← 草稿内容（可频繁修改）     │ │
│ │ settings                     ← 运行时配置（可热更新）        │ │
│ │ versionId                    ← 当前草稿版本ID              │ │
│ │ active, activeVersionId      ← 发布状态（受保护，不直接改）  │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ activeVersionId ──→ ┌─────────────────────────────────────────┐ │
│                     │ WorkflowHistory (workflow_history 表)   │ │
│                     │ ┌─────────────────────────────────────┐ │ │
│                     │ │ versionId (PK)                     │ │ │
│                     │ │ nodes, connections, nodeGroups      │ │ │
│                     │ │ ← 不可变快照（发布后冻结）           │ │ │
│                     │ └─────────────────────────────────────┘ │ │
│                     └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

关键实体定义：

- **WorkflowEntity** ([workflow-entity.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/db/src/entities/workflow-entity.ts))：主实体，包含草稿和发布状态
- **WorkflowHistory** ([workflow-history.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/db/src/entities/workflow-history.ts))：版本历史，每个 `versionId` 是不可变快照
- **WorkflowPublishedVersion** ([workflow-published-version.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/db/src/entities/workflow-published-version.ts))：发布版本映射（当启用 publication service 时）

---

## 二、哪些字段变化会触发新 versionId 生成并保存历史版本？

核心逻辑位于 [workflow.service.ts#L358-L385](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflow.service.ts#L358-L385)：

```typescript
const hasNodesKey = 'nodes' in workflowUpdateData;
const hasConnectionsKey = 'connections' in workflowUpdateData;
const hasNodeGroupsKey = 'nodeGroups' in workflowUpdateData;
const nodesChanged = hasNodesKey && !isEqual(workflowUpdateData.nodes, workflow.nodes);
const connectionsChanged =
    hasConnectionsKey && !isEqual(workflowUpdateData.connections, workflow.connections);
const nodeGroupsChanged =
    hasNodeGroupsKey && !isEqual(workflowUpdateData.nodeGroups, workflow.nodeGroups);
const saveNewVersion = nodesChanged || connectionsChanged || nodeGroupsChanged;
```

**触发新 versionId 的三个字段：**

| 字段 | 触发条件 | 说明 |
|------|----------|------|
| `nodes` | 传入的 `nodes` 与数据库中现有 `nodes` 不相等 | 节点增删改、参数修改、位置移动等 |
| `connections` | 传入的 `connections` 与数据库中现有 `connections` 不相等 | 连线增删改 |
| `nodeGroups` | 传入的 `nodeGroups` 与数据库中现有 `nodeGroups` 不相等 | 节点分组增删改 |

**满足任意一个**即：
1. 生成新的 `versionId = uuid()`
2. 将该版本通过 `WorkflowHistoryService.saveVersion()` 写入 `workflow_history` 表
3. 将新 `versionId` 同步更新到 `workflow_entity` 表

**不会触发新 versionId 的字段：**

- `name`、`description`、`meta`、`settings`、`staticData`、`pinData` — 这些字段的修改不会产生版本快照，也不会保存到历史记录中。它们属于工作流的元数据或运行时配置，不影响逻辑结构。

---

## 三、哪些字段永远不会通过普通 update 直接修改？

核心逻辑位于 [workflow.service.ts#L459-L477](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflow.service.ts#L459-L477)：

```typescript
const fieldsToUpdate = [
    'name',
    'nodes',
    'connections',
    'nodeGroups',
    'meta',
    'settings',
    'staticData',
    'pinData',
    'versionId',
    'description',
    'updatedAt',
    // do not update active fields ← 显式注释，禁止更新！
];

const updatePayload = pick(workflowUpdateData, fieldsToUpdate);
await this.workflowRepository.update(workflowId, updatePayload);
```

使用 `lodash.pick` 从 `workflowUpdateData` 中仅提取白名单字段，其余字段一律丢弃。

**被永久保护的字段（不在白名单中）：**

| 字段 | 说明 | 仅能通过什么方式修改 |
|------|------|---------------------|
| `active` | 工作流是否已发布（布尔标记） | `activateWorkflow()` / `deactivateWorkflow()` / `archive()` |
| `activeVersionId` | 指向哪个历史版本作为生产运行版本 | `activateWorkflow()` / `archive()` |
| `activeVersion` | 关联的 WorkflowHistory 实体（TypeORM 关系） | 由 `activeVersionId` 自动解析 |
| `isArchived` | 软删除标记 | `archive()` / `unarchive()` |
| `triggerCount` | 触发器计数（用于计费） | 由 `ActiveWorkflowManager` 自动计算 |
| `versionCounter` | 版本计数器（数据库自增） | 数据库自动维护 |

此外，Controller 层也做了安全保护。在 [workflows.controller.ts#L307-L321](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflows.controller.ts#L307-L321) 中：

```typescript
let updateData = new WorkflowEntity();
const { tags, parentFolderId, aiBuilderAssisted, expectedChecksum, autosaved, ...rest } = body;
// Security: Object.assign is now safe because the DTO validates and filters all input
// Only fields defined in UpdateWorkflowDto are assigned; internal fields like
// triggerCount, versionCounter, isArchived, active, activeVersionId, etc. are never set from user input
Object.assign(updateData, rest);
```

DTO 校验 + `lodash.pick` 白名单，构成了 **双层防护**：即使 API 请求体中包含 `activeVersionId` 字段，也会被 DTO 校验拒绝（不在 schema 中），或即使通过 DTO，也会被 `pick` 丢弃。

---

## 四、为什么 settings 变化或 publishIfActive 时要重新调用 activateWorkflow？

核心逻辑位于 [workflow.service.ts#L553-L560](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflow.service.ts#L553-L560)：

```typescript
// Activate workflow if requested, or
// Reactivate workflow if settings changed and workflow has an active version
if (updatedWorkflow.activeVersionId && (publishCurrent || settingsChanged)) {
    await this.activateWorkflow(user, workflowId, {
        versionId: updatedWorkflow.activeVersionId,
        source,
    });
}
```

### 4.1 `ActiveWorkflowManager` 读取数据的关键差异

在 [active-workflow-manager.ts#L690-L710](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts#L690-L710) 中：

```typescript
// Get workflow data from the active version
if (!dbWorkflow.activeVersion) {
    throw new UnexpectedError('Active version not found for workflow', { ... });
}

const { nodes, connections } = dbWorkflow.activeVersion;  // ← 从 WorkflowHistory 读取！
dbWorkflow.nodes = nodes;
dbWorkflow.connections = connections;

workflow = new Workflow({
    id: dbWorkflow.id,
    nodes,
    connections,
    settings: dbWorkflow.settings,    // ← 但 settings 仍从 WorkflowEntity 读取！
    ...
});
```

**关键发现：**

| 数据来源 | 字段 | 存储位置 |
|----------|------|----------|
| ✅ 始终指向已发布版本 | `nodes`, `connections` | `WorkflowHistory`（由 `activeVersionId` 关联） |
| ⚠️ 直接读主表 | `settings`, `staticData`, `name` | `WorkflowEntity` 主表 |

这意味着：`ActiveWorkflowManager` 中的触发器/轮询器/Webhook 注册使用的是 **冻结的节点/连接**，但 **热的配置** 来自主表。

### 4.2 重新激活的两种场景

#### 场景 A：`settingsChanged === true`

用户修改了 settings（如超时时间、错误工作流、执行模式等），但没有改 nodes/connections。此时：

- `saveNewVersion = false`（因为 nodes/connections/nodeGroups 没变）
- `versionId` 不变
- `activeVersionId` 不变（因为 publishIfActive 为 false）
- 但 `WorkflowEntity.settings` 已被更新

**为什么需要重新激活？**

`ActiveWorkflowManager` 中已注册的触发器/轮询器持有的是旧 settings 的快照。新的 settings（如 `executionTimeout`）需要在下次执行时生效。虽然每次执行时 `getBase()` 会重新读数据库，但某些设置（如 Webhook 路径、轮询间隔）在注册时就已固定。重新激活确保：

1. Webhook 路径、触发器注册参数使用最新 settings
2. 旧的注册被清理，新的注册使用最新配置

**但节点/连接仍然是已发布的那个版本** —— 因为 `activateWorkflow` 传入的 `versionId` 是 `updatedWorkflow.activeVersionId`，它指向的还是原来的 `WorkflowHistory` 记录。

#### 场景 B：`publishIfActive === true`（仅公共 API 使用）

在 [workflows.handler.ts#L305-L314](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/public-api/v1/handlers/workflows/workflows.handler.ts#L305-L314) 中，公共 API 的 update 始终设置 `publishIfActive: true`：

```typescript
const updatedWorkflow = await Container.get(WorkflowService).update(
    req.user,
    updateData,
    id,
    {
        forceSave: true,
        publicApi: true,
        publishIfActive: true,  // ← 公共 API 自动发布
        source: 'api',
    },
);
```

这意味着公共 API 的任何 update 都会：
1. 如果 nodes/connections 变了 → 生成新 versionId → 保存历史
2. 设置 `activeVersionId = 新 versionId`（`publishCurrent` 逻辑）
3. 重新激活，将新版本投入生产

**设计意图**：公共 API 用户期望 "更新即生效"，不需要再调用单独的激活接口。

---

## 五、设计如何防止草稿内容意外进入生产版本？

### 5.1 三层隔离机制

```
  用户编辑（草稿）                  生产运行（发布版本）
  ┌──────────────────────┐         ┌──────────────────────────────┐
  │ WorkflowEntity       │         │ WorkflowHistory              │
  │  .nodes/connections  │         │  .nodes/connections           │
  │  (频繁被修改)        │  解耦    │  (不可变快照，写入后不再改)  │
  │                      │ ←──────→ │                              │
  │  .versionId          │  指向    │  .versionId (PK)             │
  │  .activeVersionId ──────────────→                              │
  └──────────────────────┘         └──────────────────────────────┘
           │                                    │
           ▼                                    ▼
  用户在编辑器看到的内容         ActiveWorkflowManager 实际执行的内容
```

**第一层：字段隔离** — `activeVersionId` 与 `versionId` 是两个独立字段

- `versionId`：跟随草稿，每次 nodes/connections 变化都会更新
- `activeVersionId`：只有通过 `activateWorkflow()` 才会改变，指向一个已保存的 `WorkflowHistory` 快照

即使草稿频繁修改，`versionId` 不断变化，但 `activeVersionId` 保持不变，生产执行的仍是旧版本。

**第二层：白名单保护** — 普通 update 永远不碰 active 字段

`fieldsToUpdate` 白名单 + `lodash.pick`，确保任何 API 请求都无法直接修改 `active` 或 `activeVersionId`。必须走专门的激活流程。

**第三层：ActiveWorkflowManager 始终从 activeVersion 读取**

```typescript
const { nodes, connections } = dbWorkflow.activeVersion;  // 不是 dbWorkflow.nodes
```

即使有人直接修改了 `workflow_entity.nodes`（绕过服务层），`ActiveWorkflowManager` 也不会使用那些修改。它有自己的内存缓存和 `WebhookEntity` 表存储，注册时使用的是 `activeVersion` 中的数据。

### 5.2 允许 settings 热更新的设计考量

**为什么不让 settings 也走发布流程？**

1. **用户体验**：修改执行超时、错误工作流等配置，应该立即对未来执行生效，而不需要重新发布整个工作流
2. **变更粒度**：settings 变化不影响节点逻辑结构，只影响执行行为
3. **运行时需要**：`ActiveWorkflowManager` 的触发器/Webhook 注册参数依赖 settings，注册后可能需要刷新

**关键权衡**：

```
nodes/connections  →  必须走发布流程（安全优先）
settings           →  可热更新（体验优先）
```

这通过"重新激活但使用原 activeVersionId"的方式实现：新 settings 生效了，但 nodes/connections 仍然是已冻结的那个版本。

### 5.3 完整的安全调用链

```
PATCH /workflows/:id (Controller)
    │
    ├── DTO 校验（拒绝 active/activeVersionId 等字段）
    ├── Object.assign(updateData, rest) → 仅 DTO 白名单字段
    │
    ▼
WorkflowService.update()
    │
    ├── 检测 nodes/connections/nodeGroups 变化
    │   ├── 是 → 生成新 versionId，保存 WorkflowHistory
    │   └── 否 → 保留原 versionId
    │
    ├── fieldsToUpdate 白名单 pick（再次过滤）
    │   └── 绝对不包含 active/activeVersionId
    │
    ├── publishIfActive 检查
    │   └── 仅公共 API 使用时才设置 activeVersionId
    │
    ├── settingsChanged 检查
    │   └── 如果 settings 变了 → 需要重新激活
    │
    └── 最终条件：activeVersionId 存在 && (publishCurrent || settingsChanged)
        │
        └── activateWorkflow(versionId=activeVersionId)
            │
            └── ActiveWorkflowManager.add()
                │
                └── 从 dbWorkflow.activeVersion 读取 nodes/connections（不是草稿！）
```

### 5.4 边界场景分析

| 场景 | nodes 变了 | settings 变了 | publishIfActive | 结果 |
|------|-----------|--------------|-----------------|------|
| 用户在编辑器中保存草稿（UI） | ✅ | ❌ | ❌ | 新 versionId + 历史记录，activeVersionId 不变，不重新激活 |
| 用户只改名称 | ❌ | ❌ | ❌ | 不生成版本，不重新激活 |
| 用户只改超时设置 | ❌ | ✅ | ❌ | 不生成版本，activeVersionId 不变，**重新激活**（原版本 + 新 settings） |
| 用户改了节点后点击"发布" | ✅ | ❌ | ❌（走 activate 接口） | 新 versionId + 历史记录，调用 activateWorkflow(新versionId)，**生产切换到新版本** |
| 公共 API 更新节点 | ✅ | ❌ | ✅ | 新 versionId + 历史记录，设置 activeVersionId = 新 versionId，重新激活（新版本） |
| 公共 API 只改 settings | ❌ | ✅ | ✅ | 不生成版本，但 publishIfActive 触发重新激活（原版本 + 新 settings） |

---

## 六、总结

n8n 的版本设计通过 **"草稿-发布"双轨模型** 实现了安全与灵活性的平衡：

1. **nodes/connections/nodeGroups 变化 → 新 versionId + 历史快照**（版本不可变原则）
2. **active/activeVersionId 严格受保护**（白名单 + 专门 API 双通道防护）
3. **settings 热更新通过重新激活实现**（新配置生效，但执行逻辑仍是发布版本）
4. **ActiveWorkflowManager 始终从 activeVersion 读取**（内存级别的最终防线）

这套设计确保了：草稿编辑再多再频繁，都不会意外影响生产运行；而运行时配置又能灵活调整，无需重新发布。
