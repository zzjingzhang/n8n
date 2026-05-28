# Webhook 冲突检测全链路说明

本文档描述 n8n 在发布（activate）workflow 时，如何发现不同 workflow 之间或同一 workflow 内部存在的 webhook 冲突，并把冲突信息通过弹窗呈现给用户。

---

## 1. 触发入口：前端的 `publishWorkflow` 流程

用户在 UI 上点击 “Publish/激活”，会走到前端 `useWorkflowActivate` composable 的 `publishWorkflow` 方法：

- `packages/frontend/editor-ui/src/app/composables/useWorkflowActivate.ts` 的 `publishWorkflow`
- 它通过 `workflowsStore.publishWorkflow(...)` 发起 `POST /workflows/{id}/activate`
- `workflowsStore.publishWorkflow` 实现在 `packages/frontend/editor-ui/src/app/stores/workflows.store.ts#L655-L667`

后端在控制器 `packages/cli/src/workflows/workflows.controller.ts#L468` 调用 `workflowService.activateWorkflow(...)`。

---

## 2. 后端如何构造 conflict hint

### 2.1 冲突检测的调用链

`workflowService.activateWorkflow` 内会调用私有方法：

```
activateWorkflow
  └─ _detectWebhookConflicts(workflowEntity, versionToActivate)
       └─ _findConflictingWebhooks(workflowEntity, versionToActivate)
            └─ webhookService.findWebhookConflicts(workflow, additionalData)
                 └─ _findWebhookConflicts(workflow, checkEntries)
```

代码位置：`packages/cli/src/workflows/workflow.service.ts#L627-L668`

### 2.2 `_detectWebhookConflicts` 的行为

```typescript
const conflicts = await this._findConflictingWebhooks(workflowEntity, versionToActivate);

if (conflicts.length > 0) {
  throw new ConflictError(
    'There is a conflict with one of the webhooks.',
    JSON.stringify(
      conflicts.map(({ trigger, conflict }) => ({ trigger, conflict })),
    ),
  );
}
```

- 冲突列表被 **序列化成 JSON 字符串**，作为 `ConflictError` 的第二参数（即后端响应中的 `hint` 字段）。
- `ConflictError` 是一个 HTTP 409 响应错误，实现在 `packages/cli/src/errors/response-errors/conflict.error.ts`。

### 2.3 `_findWebhookConflicts` 的检测算法

位于 `packages/cli/src/webhooks/webhook.service.ts#L303-L357`。它同时检测两种冲突：

1. **同一 workflow 内部的节点冲突**：用 `processedWebhooks: Map<string, IWebhookData>` 保存已处理 webhook，key 是 `` `${httpMethod} ${getWebhookPath(webhook)}` ``。遇到相同 key 就立即加入 conflicts。
2. **与其它已发布 workflow 的冲突**：对每个 webhook 调用 `findWebhook(method, getWebhookPath(webhook))`，如果查到的 webhook 所属 `workflowId` 与当前 workflow 不同，则加入 conflicts。

冲突条目结构：

```typescript
{
  trigger: INode;         // 当前 workflow 中产生冲突的节点（整个 INode 对象）
  conflict: {             // 被冲突对象的关键信息
    workflowId: string;
    webhookPath: string;
    method: HttpMethod;
    node: string;
    webhookId?: string;
  }
}
```

返回的 `conflict` 对象是 `Partial<WebhookEntity>`，其字段恰好能匹配前端弹窗 `WorkflowActivationConflictingWebhookModal` 所需的 `workflowId / node / webhookPath / ...`。

---

## 3. 前端弹窗如何消费 hint

### 3.1 解析后端返回

在 `useWorkflowActivate.publishWorkflow` 的 `catch` 分支中：

- `isWebhookConflictError(error)` 通过 `errorCode === 409` 且 hint 能被解析为包含 `trigger` 字段的数组来判定。
- `parseWebhookConflictError` 位于 `packages/frontend/editor-ui/src/app/composables/useWorkflowActivate.ts#L39-L59`。

### 3.2 打开弹窗

```typescript
const { trigger, conflict } = parseWebhookConflictError(error)?.pop() || {};
// 用 conflict.workflowId 再请求一次 workflowsListStore.fetchWorkflow 拿到冲突 workflow 的 name
uiStore.openModalWithData({
  name: WORKFLOW_ACTIVATION_CONFLICTING_WEBHOOK_MODAL_KEY,
  data: {
    triggerType: trigger?.type,  // 决定弹窗显示 "Webhook" / "Form" / "Chat" 文案
    workflowName,                // 被冲突 workflow 的 name
    ...conflict,                 // workflowId, node, webhookPath, method, webhookId
  },
});
```

### 3.3 弹窗组件

组件：`packages/frontend/editor-ui/src/app/components/WorkflowActivationConflictingWebhookModal.vue`

Props `data` 结构：

```typescript
{
  workflowName: string;
  triggerType: string;
  workflowId: string;
  webhookPath: string;
  node: string;
}
```

组件会根据 `triggerType`（如 `WEBHOOK_NODE_TYPE` / `FORM_TRIGGER_NODE_TYPE` / `CHAT_TRIGGER_NODE_TYPE`）切换标题和提示文案，并展示完整冲突 URL：

```
<text-light>{{ webhookUrl }}/</text-light><text-dark bold>{{ data.webhookPath }}</text-dark>
```

`webhookUrl` 取自 `rootStore.webhookUrl`，并提供链接跳转至冲突 workflow 的编辑页 `/workflow/{workflowId}`，让用户去停用或修改它。

---

## 4. 静态路径 vs 动态路径

n8n 把 webhook 分为两类，两者在匹配、缓存、以及数据库字段使用上差别很大。

### 4.1 判定条件

`webhook.service.ts` 的 `isDynamicPath(rawPath)`：

```typescript
if (path === '' || path === ':' || path === '/:') return false;
return path.startsWith(':') || path.includes('/:');
```

即：包含 `:` 开头的段（如 `:id`）才视为动态路径。`':'` 与 `'/:'` 这样的“整段就是冒号”被特殊处理为静态路径。

`WebhookEntity.isDynamic`（`packages/@n8n/db/src/entities/webhook-entity.ts#L50-L52`）用相同逻辑实现。

### 4.2 三类核心字段

| 字段 | 静态路径 | 动态路径 |
| --- | --- | --- |
| `webhookId` | 不设置（`undefined`） | 必须设置，值为 `node.webhookId`（由 `NodeHelpers.getNodeWebhookId` 生成，基于 workflowId + nodeName 的 UUID v5） |
| `pathLength` | 不设置（`undefined`） | 设置为 `webhookPath.split('/').length`，即“模板路径”的段数 |
| `staticSegments` | `webhookPath.split('/')` 过滤掉 `:` 开头的段后的全部段 | 同上，用于动态匹配时按“静态段匹配度”选最具体的那个 |

在 `active-workflow-manager.ts#L186-L189` 注册 webhook 时只有动态路径才填充这两个字段：

```typescript
if ((path.startsWith(':') || path.includes('/:')) && node.webhookId) {
  webhook.webhookId = node.webhookId;
  webhook.pathLength = webhook.webhookPath.split('/').length;
}
```

### 4.3 匹配逻辑

`WebhookService.findWebhook(method, path)` 会先走 `findCached`：

1. 先查缓存 `cacheKey = webhook:${method}-${path}`。
2. 未命中 → `findStaticWebhook(method, path)`，用 `webhookPath + method` 作为主键查 DB。
3. 仍未命中 → `findDynamicWebhook(path, method)`：
   - 把 `path` 拆成 `[uuidSegment, ...otherSegments]`
   - DB 查询条件：`{ webhookId: uuidSegment, method, pathLength: otherSegments.length }`
   - 在候选集中通过 `staticSegments.every(s => requestSegments.has(s))` 过滤，取 `staticSegments.length` 最大的那个；若候选只有 `:var` 这种全动态段，也能匹配任何路径。

对应代码：`packages/cli/src/webhooks/webhook.service.ts#L46-L124`。

### 4.4 缓存策略

- **静态 webhook**：被缓存。缓存 key 形如 `webhook:GET-user/profile`，在 `populateCache`、`storeWebhook`、`findCached` 等路径均会写入。
- **动态 webhook**：**不缓存**。因为请求路径中的动态段是变量（如 `user/123/posts`），没有办法把所有可能的请求路径都预缓存；因此只有静态 webhook 能缓存命中，动态 webhook 每次都会走 DB。
- 缓存类型：`CacheService`（默认是 Redis，本地 fallback 为内存）。

`populateCache` 只拉取 `webhookRepository.getStaticWebhooks()`，也印证了这一点。

### 4.5 WebhookEntity 的派生属性

`packages/@n8n/db/src/entities/webhook-entity.ts`：

```typescript
private get uniquePath() {
  return this.webhookPath.includes(':')
    ? [this.webhookId, this.webhookPath].join('/')
    : this.webhookPath;
}
get cacheKey() {
  return `webhook:${this.method}-${this.uniquePath}`;
}
get staticSegments() {
  return this.webhookPath.split('/').filter((s) => !s.startsWith(':'));
}
get isDynamic() {
  return this.webhookPath.split('/').some((s) => s.startsWith(':'));
}
```

- 静态路径的 `uniquePath = webhookPath`，如 `user/profile`。
- 动态路径的 `uniquePath = webhookId + '/' + webhookPath`，如 `abc-uuid/user/:id/posts`；其中 `webhookId` 是该节点的 webhook 唯一 ID（UUID v5）。
- 数据库表 `webhook_entity` 主键是 `(webhookPath, method)`，同时有索引 `(webhookId, method, pathLength)` 用于加速动态查找。

### 4.6 冲突检测中如何使用

在 `_findWebhookConflicts` 中，每个 webhook 的对比 key 是：

```typescript
const webhookKey = `${webhook.httpMethod} ${this.getWebhookPath(webhook)}`;
```

其中 `getWebhookPath`（`webhook.service.ts#L180-L184`）：

```typescript
return [webhook.path.includes(':') ? webhook.webhookId : undefined, webhook.path]
  .filter(Boolean)
  .join('/');
```

- 对静态路径，key 形如 `GET user/profile`；
- 对动态路径，key 形如 `GET abc-uuid/user/:id/posts`；

因此同一 workflow 内两个节点只要 `method + uniquePath` 相同就会被视为冲突；跨 workflow 的冲突通过 `findWebhook` 在 DB 中查找，依赖上节所述的静态/动态两套查找路径。

---

## 5. 完整数据链路示意

```
User 点击 Publish
  └─ useWorkflowActivate.publishWorkflow (FE)
       └─ workflowsStore.publishWorkflow (FE)  POST /workflows/{id}/activate
            └─ WorkflowsController.activateWorkflow (BE)
                 └─ WorkflowService.activateWorkflow
                      └─ _detectWebhookConflicts
                           └─ _findConflictingWebhooks
                                └─ WebhookService.findWebhookConflicts
                                     └─ _findWebhookConflicts
                                          ├─ 本地 Map: ${method} ${uniquePath}
                                          └─ DB: findStaticWebhook / findDynamicWebhook
                      ├─ 无冲突 → 写入 activeVersionId
                      └─ 有冲突 → throw ConflictError(message, JSON.stringify(hint))
  HTTP 409
  └─ parseWebhookConflictError (FE) 取出 trigger / conflict
       └─ uiStore.openModalWithData(WORKFLOW_ACTIVATION_CONFLICTING_WEBHOOK_MODAL_KEY, data)
            └─ WorkflowActivationConflictingWebhookModal (FE) 渲染冲突提示弹窗
```

---

## 6. 关键文件索引

- 前端激活逻辑：`packages/frontend/editor-ui/src/app/composables/useWorkflowActivate.ts`
- 前端弹窗：`packages/frontend/editor-ui/src/app/components/WorkflowActivationConflictingWebhookModal.vue`
- 前端 Store：`packages/frontend/editor-ui/src/app/stores/workflows.store.ts`
- 后端控制器：`packages/cli/src/workflows/workflows.controller.ts`
- 后端服务：`packages/cli/src/workflows/workflow.service.ts`（`_detectWebhookConflicts`）
- Webhook 服务（核心匹配/检测逻辑）：`packages/cli/src/webhooks/webhook.service.ts`
- Webhook 实体与派生属性：`packages/@n8n/db/src/entities/webhook-entity.ts`
- 注册 webhook（填充 webhookId / pathLength）：`packages/cli/src/active-workflow-manager.ts`
- Conflict 响应错误：`packages/cli/src/errors/response-errors/conflict.error.ts`
