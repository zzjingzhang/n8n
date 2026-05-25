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

`webhookUrl` 取自 `rootStore.webhookUrl