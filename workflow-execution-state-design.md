# 工作流执行状态设计文档

## 一、Store 架构概览

n8n 前端对工作流执行状态的管理已从单一的 `workflows.store` 拆分为三个专门的 store，形成分层架构：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     workflows.store (兼容层)                         │
│  - 暴露兼容 computed，路由到下层 store                               │
│  - 维护向后兼容性，避免大规模重构                                    │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          │                                       │
┌─────────▼─────────────────┐        ┌────────────▼──────────────────┐
│ workflowExecutionState    │        │ executionData.store          │
│ .store (会话级状态)       │        │ (执行级数据)                  │
│ - activeExecutionId       │        │ - 按 executionId 分实例       │
│ - displayedExecutionId    │        │ - 存储 execution 完整 payload │
│ - pendingExecution        │        │ - runData, startedData 等     │
│ - currentWorkflowExecutions│       │                               │
│ - lastSuccessfulExecution │        │                               │
└───────────────────────────┘        └──────────────────────────────┘
```

**迁移说明：**
- 大量执行状态已从 [workflows.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflows.store.ts) 迁移至
  [workflowExecutionState.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowExecutionState.store.ts) 和
  [executionData.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/executionData.store.ts)
- `workflows.store` 仍暴露兼容 computed（见 [workflows.store.ts#L88-L229](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflows.store.ts#L88-L229)），通过 `currentState` 计算属性路由到下层 store

---

## 二、activeExecutionId 三态含义

`activeExecutionId` 是执行状态机的核心，使用三态设计来精确表达执行生命周期中的不同阶段。

### 2.1 三态定义

| 状态       | 类型     | 含义说明                                                                 |
|------------|----------|--------------------------------------------------------------------------|
| `undefined`| 初始态   | 没有活动执行被跟踪，或执行已结束清理                                     |
| `null`     | 过渡态   | 执行已启动（前端已发起请求），但后端分配的真实 executionId 尚未返回       |
| `string`   | 确定态   | 后端已分配真实的 executionId，当前正在跟踪该执行                         |

源码定义见 [workflowExecutionState.store.ts#L68-L74](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowExecutionState.store.ts#L68-L74)：
```typescript
/**
 * Tri-state semantics:
 *   undefined -> no active execution being tracked
 *   null      -> execution started but backend id not yet known
 *   string    -> active backend execution id
 */
const activeExecutionId = ref<string | null | undefined>();
```

### 2.2 状态转换图

```
          开始执行                       收到后端ID
undefined ──────────► null ───────────────► string
   ▲                    │                       │
   │                    │ 执行取消/失败         │ 执行完成/取消/失败
   │                    ▼                       ▼
   └───────────────────────────────────────────┘
                        清理
```

### 2.3 状态判定逻辑

`isWorkflowRunning` 计算属性根据三态判断执行是否在进行中（[workflowExecutionState.store.ts#L200-L211](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowExecutionState.store.ts#L200-L211)）：

```typescript
const isWorkflowRunning = computed(() => {
    if (activeExecutionId.value === null) return true;  // 过渡态也视为运行中
    if (activeExecutionId.value && activeExecution.value) {
        if (
            ['waiting', 'running'].includes(activeExecution.value.status) &&
            !activeExecution.value.finished
        ) {
            return true;
        }
    }
    return false;
});
```

**关键设计：** `null` 状态也返回 `true`，确保用户在点击执行按钮后立即看到"运行中"状态。

---

## 三、IN_PROGRESS_EXECUTION_ID 乐观 UI 机制

### 3.1 常量定义

在 [placeholders.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/constants/placeholders.ts) 中定义：
```typescript
export const IN_PROGRESS_EXECUTION_ID = '__IN_PROGRESS__';
```

这是一个特殊的 sentinel 值（哨兵值），用于在后端返回真实 executionId 之前，让前端能够展示执行数据。

### 3.2 核心流程

当用户点击执行按钮时，`useRunWorkflow` 执行以下流程（[useRunWorkflow.ts#L380-L413](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/useRunWorkflow.ts#L380-L413)）：

```typescript
// 1. 立即将 activeExecutionId 设为 null，标记"正在启动"
workflowState.setActiveExecutionId(null);

// 2. 构造一个带有 IN_PROGRESS_EXECUTION_ID 的完整 execution payload
const executionData: IExecutionResponse = {
    id: IN_PROGRESS_EXECUTION_ID,
    finished: false,
    mode: 'manual',
    status: 'running',
    createdAt: new Date(),
    startedAt: new Date(),
    // ... 其他字段包括已有的 runData, pinData 等
};

// 3. 立即存入 store，UI 可以立即展示
workflowState.setWorkflowExecutionData(executionData);

// 4. 发起后端请求
const runWorkflowApiResponse = await runWorkflowApi(startRunData);

// 5. 收到真实 executionId 后，升级数据
if (response.executionId && workflowExecutionIdIsNew && workflowExecutionIdIsPending) {
    workflowState.setActiveExecutionId(response.executionId);
}
```

### 3.3 数据迁移（promotePendingExecution）

当后端返回真实 ID 时，`promotePendingExecution` 方法将占位数据迁移到真实 ID 下（[workflowExecutionState.store.ts#L302-L312](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowExecutionState.store.ts#L302-L312)）：

```typescript
function promotePendingExecution(executionId: string) {
    const scaffold = pendingExecution.value;
    pendingExecution.value = null;
    const promoted: IExecutionResponse = scaffold
        ? { ...scaffold, id: executionId }
        : ({ id: executionId } as IExecutionResponse);
    
    trackExecutionId(executionId);
    useExecutionDataStore(createExecutionDataId(executionId)).setExecution(promoted);
    setActiveExecutionId(executionId);
    fireChange(CHANGE_ACTION.UPDATE, 'pendingExecution');
}
```

### 3.4 解析链：resolveActiveExecId

`resolveActiveExecId` 函数负责在三态下解析出当前应该使用的执行 ID（[workflows.store.ts#L415-L422](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflows.store.ts#L415-L422)）：

```typescript
function resolveActiveExecId(): string | undefined {
    // 有真实 ID，使用真实 ID
    if (typeof currentState.value.activeExecutionId === 'string')
        return currentState.value.activeExecutionId;
    
    // 过渡态（null），使用占位符 ID
    if (currentState.value.activeExecutionId === null) return IN_PROGRESS_EXECUTION_ID;
    
    // 没有活动执行，尝试使用 displayedExecutionId（查看历史执行时）
    const displayedExecutionId = currentState.value.displayedExecutionId;
    if (typeof displayedExecutionId === 'string') return displayedExecutionId;
    
    return undefined;
}
```

### 3.5 activeExecution 回退链

`activeExecution` 计算属性按照优先级返回执行数据（[workflowExecutionState.store.ts#L134-L143](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowExecutionState.store.ts#L134-L143)）：

```typescript
const activeExecution = computed(() => {
    // 1. 过渡态：返回 pendingExecution 脚手架数据
    if (activeExecutionId.value === null) return pendingExecution.value;
    
    // 2. 确定态：从 executionData store 获取真实数据
    if (typeof activeExecutionId.value === 'string') {
        return useExecutionDataStore(createExecutionDataId(activeExecutionId.value)).execution;
    }
    
    // 3. 无活动执行但有展示 ID：返回展示的执行数据
    if (typeof displayedExecutionId.value === 'string') {
        return useExecutionDataStore(createExecutionDataId(displayedExecutionId.value)).execution;
    }
    
    return null;
});
```

---

## 四、Push Handler 更新 active/published 状态流程

Push 连接通过事件驱动机制，实时同步后端的工作流状态变化到前端。

### 4.1 Push Handler 注册

在 [usePushConnection.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/usePushConnection.ts) 中注册了以下与工作流激活相关的 handler：

- `workflowActivated` - 工作流被激活
- `workflowDeactivated` - 工作流被停用
- `workflowAutoDeactivated` - 工作流被自动停用
- `workflowFailedToActivate` - 工作流激活失败

### 4.2 执行启动：executionStarted handler

当后端开始执行时，`executionStarted` handler 处理状态初始化（[executionStarted.ts#L19-L72](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/executionStarted.ts#L19-L72)）：

```typescript
export async function executionStarted({ data }: ExecutionStarted) {
    const stateStore = useWorkflowExecutionStateStore(
        createWorkflowExecutionStateId(workflowsStore.workflowId),
    );
    
    // 判断是否需要初始化
    const needsInit =
        stateStore.activeExecutionId === null ||
        typeof stateStore.activeExecutionId === 'undefined' ||
        (isIframe && stateStore.activeExecutionId !== data.executionId);
    
    if (needsInit) {
        // 将 pendingExecution 升级为真实 ID 存储
        stateStore.promotePendingExecution(data.executionId);
    }
    
    // 初始化或重置 executionData store
    const executionDataStore = useExecutionDataStore(createExecutionDataId(data.executionId));
    if (!executionDataStore.execution?.data || needsInit) {
        executionDataStore.setExecution({
            id: data.executionId,
            finished: false,
            mode: 'manual',
            status: 'running',
            // ...
        });
    }
}
```

### 4.3 执行完成：executionFinished handler

执行完成后，`executionFinished` handler 清理状态并更新最终数据（[executionFinished.ts#L69-L534](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/executionFinished.ts#L69-L534)）：

```typescript
export async function executionFinished(
    { data }: ExecutionFinished,
    options: ExecutionFinishedOptions,
) {
    // 1. 从后端拉取完整执行数据
    const execution = await fetchExecutionData(data.executionId);
    
    // 2. 根据执行状态进行处理
    if (execution.status === 'error' || execution.status === 'canceled') {
        handleExecutionFinishedWithErrorOrCanceled(execution, runExecutionData);
    } else {
        handleExecutionFinishedWithSuccessOrOther(
            options.workflowState,
            execution.status,
            successToastAlreadyShown,
        );
    }
    
    // 3. 更新 executionData store 中的最终数据
    const executionDataStore = useExecutionDataStore(createExecutionDataId(execution.id));
    executionDataStore.setExecution({
        ...workflowExecution,
        status: execution.status,
        id: execution.id,
        stoppedAt: execution.stoppedAt,
    });
    
    // 4. 重置 activeExecutionId 为 undefined，表示执行结束
    stateStore.setActiveExecutionId(undefined);
}
```

### 4.4 工作流激活：workflowActivated handler

当工作流被激活（published）时推送的事件（[workflowActivated.ts#L9-L35](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/workflowActivated.ts#L9-L35)）：

```typescript
export async function workflowActivated({ data }: WorkflowActivated) {
    const { workflowId, activeVersionId } = data;
    
    const workflowIsBeingViewed = workflowsStore.workflowId === workflowId;
    const activeVersionChanged = workflowDocumentStore?.value?.activeVersionId !== activeVersionId;
    
    if (workflowIsBeingViewed && activeVersionChanged) {
        // 无未保存更改时，重新加载工作流数据
        if (!uiStore.stateIsDirty) {
            const updatedWorkflow = await workflowsListStore.fetchWorkflow(workflowId);
            await initializeWorkspace(updatedWorkflow);
        }
    }
    
    // 移除自动停用横幅
    if (workflowIsBeingViewed) {
        bannersStore.removeBannerFromStack('WORKFLOW_AUTO_DEACTIVATED');
    }
}
```

### 4.5 工作流停用：workflowDeactivated handler

当工作流被停用时推送的事件（[workflowDeactivated.ts#L8-L27](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/workflowDeactivated.ts#L8-L27)）：

```typescript
export async function workflowDeactivated({ data }: WorkflowDeactivated) {
    if (workflowsStore.workflowId === data.workflowId) {
        if (!uiStore.stateIsDirty) {
            // 无未保存更改时，重新加载工作流数据
            const updatedWorkflow = await workflowsListStore.fetchWorkflow(data.workflowId);
            await initializeWorkspace(updatedWorkflow);
        } else {
            // 有未保存更改时，仅更新 active 状态，不覆盖用户修改
            workflowDocumentStore?.value?.setActiveState({ 
                activeVersionId: null, 
                activeVersion: null 
            });
        }
    }
}
```

### 4.6 主动激活流程：useWorkflowActivate

用户在前端点击"发布"按钮时，`publishWorkflow` 方法发起请求并更新状态（[useWorkflowActivate.ts#L85-L176](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/useWorkflowActivate.ts#L85-L176)）：

```typescript
const publishWorkflow = async (workflowId: string, versionId: string, options?: {...}) => {
    // 1. 调用后端激活 API
    const updatedWorkflow = await workflowsStore.publishWorkflow(workflowId, {
        versionId,
        expectedChecksum,
        // ...
    });
    
    // 2. 更新 workflowsListStore 缓存
    workflowsStore.setWorkflowActive(workflowId, updatedWorkflow.activeVersion, true);
    
    // 3. 更新 workflowDocumentStore 的激活状态
    workflowDocumentStore.setActiveState({
        activeVersionId: updatedWorkflow.activeVersion.versionId,
        activeVersion: updatedWorkflow.activeVersion,
    });
    
    // 4. 更新版本数据和 checksum
    if (workflowId === workflowsStore.workflowId) {
        workflowDocumentStore.setVersionData({...});
        workflowDocumentStore.setChecksum(updatedWorkflow.checksum);
    }
};
```

### 4.7 完整状态同步流程

```
          用户点击"执行"                          后端返回 executionId
┌─────────┐        ┌─────────┐        ┌─────────┐        ┌─────────┐
│undefined│───────►│  null   │───────►│ string  │───────►│undefined│
└─────────┘        └─────────┘        └─────────┘        └─────────┘
     │                  │                    │                  ▲
     │                  │                    │                  │
     │                  ▼                    ▼                  │
     │          setPendingExecution    promotePendingExecution  │
     │          创建 IN_PROGRESS       数据迁移到真实 ID        │
     │          占位数据                                        │
     │                                                          │
     └─────────────────────── setActiveExecutionId(undefined) ──┘
                                  executionFinished handler
```

---

## 五、关键设计权衡

### 5.1 三态 vs 两态

**为什么不用简单的 `string | undefined` 两态？**

- **过渡态可视化**：用户点击执行后立即看到"运行中"状态，而不是等待网络往返
- **取消支持**：在后端返回 ID 之前，用户可以点击取消，此时可以应用停止数据到 pendingExecution
- **数据完整性**：前端已有的 runData、pinData 可以立即展示，无需等待后端返回

### 5.2 IN_PROGRESS sentinel 的价值

- **乐观 UI**：零延迟反馈，显著提升用户体验
- **统一接口**：`resolveActiveExecId` 总是返回一个 string（或 undefined），下游代码无需特殊处理
- **无缝升级**：`promotePendingExecution` 对下游透明，数据平滑迁移

### 5.3 多实例 Store 设计

`executionData.store` 按 executionId 创建多个实例：
- 支持同时展示多个执行（活动执行 + 历史执行 + 上次成功执行）
- Pinia 自动处理去重，相同 ID 共享同一实例
- 独立的生命周期管理，切换工作流时批量 dispose

### 5.4 兼容层的意义

`workflows.store` 中的兼容 computed：
- 避免大规模重构，支持增量迁移
- 保持测试兼容性（`createTestingPinia` 的直接状态赋值仍可工作）
- 清晰的迁移路径，可逐步移除旧代码

---

## 六、相关文件索引

| 模块 | 文件路径 | 核心内容 |
|------|----------|----------|
| 常量定义 | [placeholders.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/constants/placeholders.ts) | `IN_PROGRESS_EXECUTION_ID` 定义 |
| 执行状态 Store | [workflowExecutionState.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflowExecutionState.store.ts) | `activeExecutionId` 三态、`promotePendingExecution` |
| 执行数据 Store | [executionData.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/executionData.store.ts) | 按 executionId 分实例的数据存储 |
| 兼容层 Store | [workflows.store.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/stores/workflows.store.ts) | 兼容 computed、`resolveActiveExecId` |
| 执行启动逻辑 | [useRunWorkflow.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/useRunWorkflow.ts) | `runWorkflow`、`runWorkflowApi` |
| Push: 执行开始 | [executionStarted.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/executionStarted.ts) | 执行开始状态初始化 |
| Push: 执行完成 | [executionFinished.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/executionFinished.ts) | 执行完成数据更新与清理 |
| Push: 工作流激活 | [workflowActivated.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/workflowActivated.ts) | 激活状态同步 |
| Push: 工作流停用 | [workflowDeactivated.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/usePushConnection/handlers/workflowDeactivated.ts) | 停用状态同步 |
| 主动激活逻辑 | [useWorkflowActivate.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/frontend/editor-ui/src/app/composables/useWorkflowActivate.ts) | `publishWorkflow`、`unpublishWorkflowFromHistory` |
