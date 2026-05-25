# Pin Data 可保存性与 Workflow 更新校验分层说明

本文档分四个部分说明 n8n 中 pin data（已钉住的节点数据）在前端、后端两层的保护逻辑，以及手动模式下为什么会用 pinData 替换最后节点的执行输出。

- 第一部分：前端如何限制单次 pin data 大小
- 第二部分：导入/添加节点时如何处理过大的 pinData
- 第三部分：后端保存 workflow 时如何同时限制 pinData 自身和 workflow 总大小
- 第四部分：手动模式下 `getLastExecutedNodeData` 为什么会用 pinData 替换最后节点输出

---

## 0. 共享的常量（前后端共用）

所有关键阈值都定义在共享的 `@n8n/api-types` 中，保证前后端对 "太大" 的理解一致：

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `MAX_PINNED_DATA_SIZE` | `1024 * 1024 * 12` (12 MB) | 允许写入 workflow 的 pinData 总上限 |
| `MAX_WORKFLOW_SIZE` | `1024 * 1024 * 16` (16 MB) | 单个 workflow 请求体（含 pinData）上限 |
| `MAX_EXPECTED_REQUEST_SIZE` | `2048` (~2 KB) | 请求头/元数据的预留余量 |

定义位置：
[base-workflow.dto.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/api-types/src/dto/workflows/base-workflow.dto.ts#L8-L15)

前端通过再导出暴露这些常量：
[limits.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/constants/limits.ts)

---

## 1. 前端如何限制单次 pin data 大小

前端的 pin 入口统一经过 composable `usePinnedData`，在它的 `setData()` 中对 "待钉的数据" 做三道检查：

1. `isValidJSON(data)`：对字符串形式的数据做 `JSON.parse`，确保是合法 JSON。
2. `isValidSize(data)`：**字节级大小校验**。
3. `isTrimmedNodeExecutionData(...)`：防止把已经被裁剪过的执行数据再钉住（那会把截断传播到下游）。

### 1.1 `isValidSize` 的实现

关键代码：
[usePinnedData.ts #isValidSize](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePinnedData.ts#L177-L225)

具体流程：

- 构造 `newPinData = { ...currentPinData, [nodeName]: data }`，即把当前节点新数据合入后整体计算。
- 用 `getPinDataSize(newPinData)` 计算合并后 pinData 的字节总和（定义在
  [useWorkflowDocumentPinData.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowDocument/useWorkflowDocumentPinData.ts#L61-L67)，
  它用 `stringSizeInBytes` 对每个节点的 pin data 累加）。
- **检查 1：pinData 自身不能超过 `MAX_PINNED_DATA_SIZE`**（`window.maxPinnedDataSize ?? MAX_PINNED_DATA_SIZE`）。超限时弹 toast：
  `ndv.pinData.error.tooLarge.title` / `ndv.pinData.error.tooLarge.description`。
- **检查 2：workflow （去掉 pinData）+ 新 pinData 总大小不能超过 `MAX_WORKFLOW_SIZE - MAX_EXPECTED_REQUEST_SIZE`**。
  这一步把当前 workflow（不含 pinData）JSON 序列化后取字节数，再加上 `newPinDataSize` 作为总大小。
  超限则弹另一组 toast：
  `ndv.pinData.error.tooLargeWorkflow.title` / `ndv.pinData.error.tooLargeWorkflow.description`。

任何一个检查不通过，`setData` 会抛错，pinnedData 不会写入 store，也不会被标记为 dirty。

### 1.2 `getMaxPinnedDataSize` 支持实例级覆盖

```ts
function getMaxPinnedDataSize() {
  return window.maxPinnedDataSize ?? MAX_PINNED_DATA_SIZE;
}
```

宿主页面可以通过在 `window` 上设置 `maxPinnedDataSize` 下调单实例的 pin 阈值（例如嵌入 n8n 编辑器时）。基础值仍然是 12 MB。

### 1.3 不可钉的节点类型

`PIN_DATA_NODE_TYPES_DENYLIST` 明确排除 `SplitInBatches`、`StickyNote`，
[nodeTypes.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/constants/nodeTypes.ts#L142)
中定义，`canPinNode` 与 `isValidNodeType` 都会检查，避免在这些节点上意外产生/保留 pinData。

---

## 2. 导入 / 添加节点时如何处理过大的 pinData

前端在 "导入" 场景下**不做大小拦截**，而是交给后端的保存校验兜底；这正是"校验分层"的体现——前端只在"主动钉"的路径上做即时反馈，导入/添加节点走的是"写 workflow 整体"的路径。

### 2.1 导入 workflow（iframe/postMessage 路径）

- `usePostMessageHandler.handleOpenWorkflow`（[usePostMessageHandler.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePostMessageHandler.ts#L79-L117)）
  直接把传入的 `workflow` 交给 `importWorkflowExact`，再由 `initializeWorkspace` 写入 workflow document store：
  [useWorkflowImport.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/useWorkflowImport.ts#L24-L58)
- 在 `useWorkflowDocument.store.ts` 中，`setPinData(workflow.pinData ?? {})` 把 pinData 原样放入
  [workflowDocument.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowDocument.store.ts#L277)。
  真正限制大小是在用户点击 Save 时——后端的 `validatePinDataSize` 会做硬校验。

### 2.2 从模板创建

`createWorkflowFromTemplate`（
[templateActions.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/features/workflows/templates/utils/templateActions.ts#L30-L67)
）对模板的 pinData 按 `template.readyToDemo` 条件保留：

```ts
pinData: template.readyToDemo ? (template.workflow.pinData ?? {}) : {},
```

非 readyToDemo 模板会直接丢弃 pinData，避免模板导入时把演示数据带进来。是否 "太大" 最终仍由后端保存判断。

### 2.3 添加节点（节点自身无 pinData）

`useCanvasOperations.addNodes` → `addNode` 路径只负责把新节点的配置/位置写入，节点本身不携带 pinData，
因此"添加节点"不会触发 pin 相关的大小校验。真正在节点上钉数据是用户在 NDV 中主动点击 Pin 按钮的行为，
走 `usePinnedData.setData` 路径，由 1.1 的三道检查拦截。

### 2.4 保存时的兜底

无论是导入还是模板创建，最终保存都走 `workflowsStore.updateWorkflow` →
[workflows.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflows.store.ts#L612-L653)，
以 `PATCH /workflows/:id` 发到后端；后端在 `WorkflowService.update()` 中执行 `validatePinDataSize`，
任何超限都会以 `BadRequestError` 返回，前端 `makeRestApiRequest` 会抛出错误。

也就是说，**前端的 "导入" 路径没有即时大小提示，只能靠后端保存时的错误反馈**，
这是本设计的"不完全在同一层"——前端只在 `usePinnedData.setData` 这条交互路径做保护，其它路径把校验责任交给后端。

---

## 3. 后端保存 workflow 时如何同时限制 pinData 自身和 workflow 总大小

后端把前端逻辑的后端版本统一放到 `WorkflowHelpers.validatePinDataSize`：

[workflow-helpers.ts #validatePinDataSize](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflow-helpers.ts#L27-L55)

函数做了和前端 `isValidSize` 相同的两步检查，但使用 Node 的 `Buffer.byteLength(..., 'utf8')`，
比前端的 `stringSizeInBytes` 更可靠（JS 层的 `Blob`/`TextEncoder` 在不同浏览器下有细微差异）。

### 3.1 检查 1：pinData 自身 ≤ 12 MB

```ts
const pinDataStr = JSON.stringify(workflow.pinData);
const pinDataSize = Buffer.byteLength(pinDataStr, 'utf8');

if (pinDataSize > MAX_PINNED_DATA_SIZE) {
  throw new BadRequestError(
    `Pinned data exceeds the maximum allowed size of ${MAX_PINNED_DATA_SIZE / (1024 * 1024)} MB`,
  );
}
```

### 3.2 检查 2：workflow（去 pinData） + pinData ≤ 16 MB − 2 KB

```ts
const { pinData: _, ...workflowWithoutPinData } = workflow;
const workflowSize =
  Buffer.byteLength(JSON.stringify(workflowWithoutPinData), 'utf8') + pinDataSize;
const limit = MAX_WORKFLOW_SIZE - MAX_EXPECTED_REQUEST_SIZE;
if (workflowSize > limit) {
  const limitMB = Math.floor(limit / (1024 * 1024));
  throw new BadRequestError(
    `Workflow with pinned data exceeds the maximum allowed size of ${limitMB} MB`,
  );
}
```

预留的 `MAX_EXPECTED_REQUEST_SIZE`（2048 字节）对应 HTTP 请求头、zod 校验附带的元数据等开销，
避免出现 "workflow 本身刚满 16 MB，加上 header 就被网关或 body parser 拒收" 的情况。

### 3.3 在哪些入口调用

- **新建 workflow**：`WorkflowCreationService.createWorkflow()` 在所有修改（补 ID、补 webhookId、验结构）之后、
  写入数据库之前调用：
  [workflow-creation.service.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflow-creation.service.ts#L115-L117)
- **更新 workflow**：`WorkflowService.update()` 在合并完新旧数据后调用：
  [workflow.service.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflow.service.ts#L451-L454)
- **public API**：`public-api/v1/handlers/workflows/workflows.handler.ts` 在 `getWorkflow` 里支持
  `excludePinnedData=true` 主动删掉 pinData 再返回（读路径，不是保存入口），
  但写路径仍由上面两个 service 把关。

### 3.4 与前端的差异总结

| 关注点 | 前端（`isValidSize`） | 后端（`validatePinDataSize`） |
| --- | --- | --- |
| 触发点 | 仅在 `usePinnedData.setData` | 所有新建/更新 workflow 的入口 |
| 度量方式 | `stringSizeInBytes`（JS） | `Buffer.byteLength(utf8)` |
| 单 pinData 上限 | `window.maxPinnedDataSize ?? 12 MB` | 固定 12 MB |
| 总 workflow 上限 | `16 MB − 2 KB` | `16 MB − 2 KB` |
| 错误形式 | toast + 抛错中断 | `BadRequestError` → HTTP 400 |

这种分层使得 "用户在前端主动钉" 能立刻得到反馈，而 "从外部导入 / 从 API 创建" 则由后端保证一致性，
不需要前端在每条导入路径上重复做同一套大小判断。

---

## 4. 手动模式下 `getLastExecutedNodeData` 为什么会用 pinData 替换最后节点输出

函数定义：
[workflow-helpers.ts #getLastExecutedNodeData](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflow-helpers.ts#L69-L105)

```ts
export function getLastExecutedNodeData(inputData: IRun): ITaskData | undefined {
  const { runData, lastNodeExecuted } = inputData.data.resultData;
  const pinData = inputData.data.resultData.pinData ?? {};

  if (lastNodeExecuted === undefined) return undefined;
  if (runData[lastNodeExecuted] === undefined) return undefined;

  const lastNodeRunData = runData[lastNodeExecuted][runData[lastNodeExecuted].length - 1];
  let lastNodePinData = pinData[lastNodeExecuted];

  if (lastNodePinData && inputData.mode === 'manual') {
    if (!Array.isArray(lastNodePinData)) lastNodePinData = [lastNodePinData];
    const itemsPerRun = lastNodePinData.map((item, index) => ({
      json: item,
      pairedItem: { item: index },
    }));
    return {
      startTime: 0,
      executionIndex: 0,
      executionTime: 0,
      data: { main: [itemsPerRun] },
      source: lastNodeRunData.source,
    };
  }
  return lastNodeRunData;
}
```

### 4.1 这段代码做了什么

- 取最后被执行的节点 `lastNodeExecuted` 的最后一次 `runData`（`lastNodeRunData`）。
- 如果该节点存在 `pinData` **且**本次运行是 `inputData.mode === 'manual'`，**用 pinData 重新构造一个 `ITaskData` 返回**：
  - `data.main[0]` 里塞的是 pinData 的每条记录，包装成 `{ json, pairedItem: { item } }`；
  - `startTime/executionIndex/executionTime` 全部归零（强调 "这不是一次真实执行"）；
  - 只保留原 `source`，方便日志/调试面板还能看到节点的来源描述。
- 其它情况（生产/触发器触发/evaluation 等非 `manual`，或没钉数据）返回原始 `lastNodeRunData`。

### 4.2 为什么只在 `manual` 模式替换

`manual` 模式是用户在编辑器里点击 "Execute node" 或 "Execute workflow" 的调试路径。
在这种模式下，**pinData 的语义是 "用这组固定数据代替该节点的真实输出"**——
这样下游节点可以在不重新执行该节点、甚至该节点因错误无法执行时，仍然能以稳定的输入继续调试。

如果在生产/触发器模式也做替换，会出现两个问题：

1. **生产数据失真**：被钉住的静态数据会覆盖真实的业务输出，导致下游节点处理错误的数据。
2. **缓存/幂等性被破坏**：生产环境的 `lastNodeExecuted` 经常被作为 "真正跑完的最后节点" 使用
   （例如 webhook 返回、子工作流结果回填），把它换成 pinData 会让父工作流拿到错误的最终产物。

因此条件被写成 `lastNodePinData && inputData.mode === 'manual'`，
只有当用户明确处于 "构建中" 的调试上下文时才做替换。

### 4.3 它被谁调用、有什么下游影响

典型调用方：

- `updateParentExecutionWithChildResults`（
  [workflow-helpers.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflow-helpers.ts#L421-L453)
  ）：子工作流完成后把 "子 workflow 最后节点的输出" 写回父工作流 `nodeExecutionStack[0].data`，
  让父工作流在 disabled 模式下重跑 Execute Workflow 节点时能拿到正确结果。
  - 当子 workflow 是 `manual` 模式跑的并且最后节点被钉，父工作流看到的就是 pinData，
    行为与 "我在子工作流钉了数据，调试父工作流" 这个心智保持一致。
- 日志、执行面板、以及前端调试 UI 会把 `getLastExecutedNodeData` 当作 "节点最后一次可展示输出"，
  钉住数据后用户看到的就是钉的数据，而不是上一次真实执行的残留输出，避免 UI 与执行引擎的实际行为不一致。

### 4.4 一个直观的例子

假设工作流 `A → B → C`，用户在 `B` 上钉了 `[{json: {x:1}}]`：

- 执行 `C`（从 `B` 的输入开始跑）：`B` 实际不执行，`C` 从 pinData 取值；
  `lastNodeExecuted` 是 `C`，`getLastExecutedNodeData` 仍然返回 `C` 的真实输出。
- 单独执行 `B`：`lastNodeExecuted` 是 `B`，runData 里可能是上次成功跑 B 的旧结果；
  `getLastExecutedNodeData` 在 manual 模式下把它替换为 pinData，保证调试面板显示的是 "我钉的数据"，
  避免混淆 "钉住的输入" 和 "上一次真实执行的输出"。

这就是为什么替换只发生在 "最后被执行的节点 + manual 模式" 上——只有在这两个条件同时满足时，
钉住的数据才应当被视为 "节点输出的权威版本"。
