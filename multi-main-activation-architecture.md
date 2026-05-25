# Multi-Main 部署：工作流激活架构深度解析

## 一、为什么只有 Leader 负责 Triggers 和 Pollers 的内存注册？

### 1.1 核心原因：避免重复执行

Triggers（触发器）和 Pollers（轮询器）是**持续运行**的内存中资源，一旦被注册就会不断监听外部事件或周期性地执行。如果多个 main 实例同时注册同一个 workflow 的 trigger/poller，将导致：

- **重复触发**：同一个事件被多个实例同时响应，产生重复执行
- **竞态条件**：多个实例竞争处理同一条消息或同一次定时任务
- **资源浪费**：不必要的内存消耗和连接占用

### 1.2 Leader 选举机制

在 multi-main 部署中，通过 Redis 实现 Leader 选举（见 [multi-main-setup.ee.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/scaling/multi-main-setup.ee.ts)）：

1. 每个 main 实例启动时尝试在 Redis 中设置一个带 TTL 的 leader key
2. 第一个成功设置 key 的实例成为 leader
3. Leader 定期（`multiMainSetup.interval` 秒）续租 TTL
4. Follower 定期检查 Redis，若 leader key 不存在则尝试接管

```
Instance A ──┐
             ├── Redis (leader key + TTL) ──→ Leader 负责注册 triggers/pollers
Instance B ──┘                              Follower 仅处理 webhook HTTP 请求
```

### 1.3 代码层的分工判断

在 [active-workflow-manager.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts) 中，有两个关键的守卫方法：

```typescript
// 只有 leader 可以注册 triggers 和 pollers 到内存
shouldAddTriggersAndPollers() {
  return this.instanceSettings.isLeader;
}

// webhook 在 init/leadershipChange 时所有实例都可操作，
// 但在 update/activate 模式下只有 leader
shouldAddWebhooks(activationMode: WorkflowActivateMode) {
  if (['init', 'leadershipChange'].includes(activationMode)) return true;
  return this.instanceSettings.isLeader;
}
```

---

## 二、普通 Main 激活 Workflow 时如何通过 PubSub 转交给 Leader？

### 2.1 完整流程

当用户在任意 main 实例（可能是 follower）上点击"激活"按钮时，流程如下：

```
用户点击激活
    │
    ▼
Follower (或任意 main) 收到 HTTP 请求
    │
    ▼
add(workflowId, 'activate') 被调用
    │
    ├─ isMultiMain && shouldPublish === true ?
    │       │
    │       ▼
    │   publisher.publishCommand({
    │     command: 'add-webhooks-triggers-and-pollers',
    │     payload: { workflowId, activeVersionId, activationMode }
    │   })
    │       │
    │       ▼
    │   Redis PubSub ──────► 所有 main 实例都收到消息
    │                          │
    │                          ▼
    │              @OnPubSubEvent('add-webhooks-triggers-and-pollers',
    │                            { instanceType: 'main', instanceRole: 'leader' })
    │                          │
    │                          ▼
    │              只有 Leader 实例的 handleAddWebhooksTriggersAndPollers() 被调用
    │                          │
    │                          ▼
    │              add(workflowId, activationMode, undefined, { shouldPublish: false })
    │                          │
    │                          ├─ addWebhooks() → 写入 webhook_entity 表
    │                          └─ addTriggersAndPollers() → 注册到内存 ActiveWorkflows
    │
    ▼
  Leader 完成后再通过 PubSub 广播 display-workflow-activation
         │
         ▼
  所有 main 实例的 @OnPubSubEvent('display-workflow-activation') 被触发
         │
         ▼
  push.broadcast() → 通知所有连接的前端 UI 显示激活状态
```

### 2.2 关键代码路径

**Follower 端**（[active-workflow-manager.ts#L660-L673](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts#L660-L673)）：

```typescript
if (this.instanceSettings.isMultiMain && shouldPublish) {
  void this.publisher.publishCommand({
    command: 'add-webhooks-triggers-and-pollers',
    payload: { workflowId, activeVersionId: dbWorkflow.activeVersionId, activationMode },
  });
  return added; // 直接返回，不在本地执行
}
```

**Leader 端**（[active-workflow-manager.ts#L806-L871](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts#L806-L871)）：

```typescript
@OnPubSubEvent('add-webhooks-triggers-and-pollers', {
  instanceType: 'main',
  instanceRole: 'leader',
})
async handleAddWebhooksTriggersAndPollers({ workflowId, activeVersionId, activationMode }) {
  await this.add(workflowId, activationMode, undefined, { shouldPublish: false });
  // ... 广播结果给所有实例
}
```

### 2.3 PubSub 消息流转机制

PubSub 系统基于 Redis pub/sub 实现，核心组件位于 `packages/cli/src/scaling/pubsub/`：

- `Publisher`：发送命令到 Redis pub/sub channel `n8n.commands`
- `Subscriber`：监听 Redis pub/sub channel，根据 `senderId` 和 `targets` 过滤
- `@OnPubSubEvent` 装饰器：声明式注册事件处理器，支持 `instanceType` 和 `instanceRole` 过滤

事件映射定义在 [pubsub.event-map.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/scaling/pubsub/pubsub.event-map.ts)：

| 事件名 | payload | 触发场景 |
|--------|---------|----------|
| `add-webhooks-triggers-and-pollers` | `{workflowId, activeVersionId, activationMode}` | 激活 workflow |
| `remove-triggers-and-pollers` | `{workflowId}` | 停用 workflow |
| `display-workflow-activation` | `{workflowId, activeVersionId}` | 激活成功后通知 UI |
| `display-workflow-deactivation` | `{workflowId}` | 停用成功后通知 UI |
| `display-workflow-activation-error` | `{workflowId, errorMessage, ...}` | 激活失败后通知 UI |

---

## 三、Leader 切换、启动初始化、手动 Deactivate 分别会注册/清理哪些资源？

### 3.1 启动初始化（init）

**触发时机**：`ActiveWorkflowManager.init()` 被调用时

**所有实例（leader + follower）**：

```
addActiveWorkflows('init')
    │
    ├─ 从 DB 读取所有 active=true 的 workflow IDs
    │
    └─ 对每个 workflowId 调用 activateWorkflow()
            │
            └─ add(workflowId, 'init', dbWorkflow, { shouldPublish: false })
                    │
                    ├─ shouldAddWebhooks('init') → true（所有实例）
                    │       └─ addWebhooks() → 写入 webhook_entity 表
                    │
                    └─ shouldAddTriggersAndPollers() → isLeader
                            └─ 仅 Leader: addTriggersAndPollers() → 注册到内存
```

- **所有实例**：在 `webhook_entity` 表中注册 webhooks（幂等操作，已存在则忽略错误）
- **仅 Leader**：将 triggers 和 pollers 注册到内存中的 `ActiveWorkflows`

### 3.2 Leader 切换（Takeover / Stepdown）

#### 3.2.1 Leader Takeover（新 Leader 接管）

通过 `@OnLeaderTakeover()` 装饰器触发（[active-workflow-manager.ts#L606-L609](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts#L606-L609)）：

```
新 Leader 触发 leader-takeover 事件
    │
    ▼
addAllTriggerAndPollerBasedWorkflows()
    │
    └─ addActiveWorkflows('leadershipChange')
            │
            ├─ 从 DB 读取所有 active=true 的 workflow IDs
            │
            └─ 对每个 workflowId 调用 activateWorkflow()
                    │
                    └─ add(workflowId, 'leadershipChange', ...)
                            │
                            ├─ shouldAddWebhooks('leadershipChange') → true
                            │       └─ addWebhooks() → 写入 webhook_entity 表
                            │
                            └─ shouldAddTriggersAndPollers() → true（新 Leader）
                                    └─ addTriggersAndPollers() → 注册到内存
```

**注册的资源**：
- Webhooks → `webhook_entity` 表
- Triggers → 内存 `ActiveWorkflows.activeWorkflows` Map
- Pollers → 内存 `ScheduledTaskManager.cronsByWorkflow` Map + `ActiveWorkflows.activeWorkflows` Map

#### 3.2.2 Leader Stepdown（Leader 降级为 Follower）

通过 `@OnLeaderStepdown()` 和 `@OnShutdown()` 装饰器触发（[active-workflow-manager.ts#L611-L616](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts#L611-L616)）：

```
Leader 触发 leader-stepdown 事件
    │
    ▼
removeAllTriggerAndPollerBasedWorkflows()
    │
    ├─ removeAllQueuedWorkflowActivations() → 清除所有重试用定时器
    │
    └─ activeWorkflows.removeAllTriggerAndPollerBasedWorkflows()
            │
            └─ 对每个内存中活跃的 workflowId 调用 remove()
                    │
                    ├─ scheduledTaskManager.deregisterCrons(workflowId)
                    │       → 清除所有 CronJob
                    │
                    ├─ 对每个 triggerResponse 调用 closeFunction()
                    │       → 关闭 trigger 的连接/监听
                    │
                    └─ delete activeWorkflows[workflowId]
                            → 从内存 Map 中移除
```

**清理的资源**：
- Triggers → 调用 `closeFunction()` 关闭外部连接（如 MQTT、WebSocket 等），从内存 Map 中移除
- Pollers → 从 `ScheduledTaskManager.cronsByWorkflow` 中注销 CronJob，从内存 Map 中移除
- Webhooks → **保留在 `webhook_entity` 表中**（Webhooks 是数据库持久化的，不需要清理）

### 3.3 手动 Deactivate（用户点击"停用"）

在 multi-main 模式下（[active-workflow-manager.ts#L987-L1004](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts#L987-L1004)）：

```
任意 main 实例收到 deactivate 请求
    │
    ▼
remove(workflowId)
    │
    ├─ clearWebhooks(workflowId)  ← 在本地执行（所有实例都可以）
    │       │
    │       ├─ 遍历 webhook 定义，调用 webhookService.deleteWebhook()
    │       │       → 通知外部服务（如 Stripe、GitHub 等）删除 webhook
    │       │
    │       └─ webhookService.deleteWorkflowWebhooks(workflowId)
    │               → 从 webhook_entity 表中删除记录
    │
    └─ publisher.publishCommand({
         command: 'remove-triggers-and-pollers',
         payload: { workflowId }
       })
            │
            ▼
       Leader 的 handleRemoveTriggersAndPollers() 被触发
            │
            ├─ activationErrorsService.deregister(workflowId)
            ├─ removeWorkflowTriggersAndPollers(workflowId)
            │       └─ activeWorkflows.remove(workflowId)
            │               → 关闭 triggers, 注销 poller cron, 从内存移除
            │
            ├─ push.broadcast({ type: 'workflowDeactivated' })
            │
            └─ publisher.publishCommand({
                 command: 'display-workflow-deactivation'
               })
```

**清理的资源**：
- Webhooks → 从 `webhook_entity` 表删除 + 通知外部服务删除 webhook
- Triggers → 从内存 `ActiveWorkflows` 移除 + 关闭外部连接
- Pollers → 从内存 `ActiveWorkflows` 移除 + 注销 CronJob
- 激活错误记录 → 从 `activation_errors` 表清除

---

## 四、Webhooks、Triggers、Pollers 三类激活资源的存储位置和生命周期差异

### 4.1 对比总览

| 维度 | Webhooks | Triggers | Pollers |
|------|----------|----------|---------|
| **存储位置** | `webhook_entity` 数据库表 | `ActiveWorkflows.activeWorkflows` 内存 Map | `ActiveWorkflows.activeWorkflows` 内存 Map + `ScheduledTaskManager.cronsByWorkflow` 内存 Map |
| **持久化** | ✅ 持久化到数据库 | ❌ 仅在内存中 | ❌ 仅在内存中 |
| **注册者** | 所有实例（init）/ 仅 Leader（activate） | 仅 Leader | 仅 Leader |
| **典型节点** | Webhook, Stripe Trigger, GitHub Trigger | MQTT Trigger, SSE Trigger, Custom Trigger | Schedule Trigger, Cron 节点, 定时轮询类节点 |
| **启动方式** | HTTP 请求到达时按需执行 | 持续运行的事件监听器 | 定时（Cron）触发的轮询 |
| **leader-stepdown 时** | 保留在 DB 中 | 关闭连接 + 从内存移除 | 注销 CronJob + 从内存移除 |
| **init/leadershipChange 时** | 写入 DB（幂等） | 注册到内存 | 注册到内存 + 创建 CronJob |

### 4.2 Webhooks 详解

**存储位置**：`webhook_entity` 数据库表

表结构（[webhook-entity.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/db/src/entities/webhook-entity.ts)）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `webhookPath` (PK) | string | Webhook 路径，如 `/webhook/uuid` 或 `/user/:id/posts` |
| `method` (PK) | string | HTTP 方法（GET/POST/PUT/DELETE） |
| `workflowId` | string | 所属 workflow ID |
| `node` | string | 节点名称 |
| `webhookId` | string | 动态路径 webhook 的唯一标识 |
| `pathLength` | number | 动态路径分段数 |

**生命周期**：

```
激活时：
  addWebhooks(workflow, ...)
    → webhookService.storeWebhook(webhook) → 写入 webhook_entity
    → webhookService.createWebhookIfNotExists() → 调用外部服务创建 webhook

请求到达时：
  WebhookController 收到 HTTP 请求
    → webhookService.findCached(method, path) → 从缓存/DB 查找
    → 找到后创建 Workflow 实例执行

停用时：
  clearWebhooks(workflowId)
    → webhookService.deleteWebhook() → 调用外部服务删除 webhook
    → webhookService.deleteWorkflowWebhooks() → 从 webhook_entity 表删除
```

**Key Design Decision**：Webhooks 持久化到数据库，是因为它们是 HTTP 端点。任何 main 实例都可能收到 webhook HTTP 请求（取决于负载均衡器路由），因此 webhook 的路由信息需要在所有实例之间共享。而 webhook 本身不需要持续运行，只有当 HTTP 请求到达时才需要执行。

### 4.3 Triggers 详解

**存储位置**：`ActiveWorkflows.activeWorkflows` 内存 Map

```typescript
// packages/core/src/execution-engine/active-workflows.ts
private activeWorkflows: { [workflowId: string]: IWorkflowData } = {};

// IWorkflowData 结构
interface IWorkflowData {
  triggerResponses: ITriggerResponse[];  // 每个 trigger 的 close 函数
}
```

**生命周期**：

```
激活时（仅 Leader）：
  addTriggersAndPollers(dbWorkflow, workflow, ...)
    → activeWorkflows.add(workflowId, workflow, ...)
        → 对每个 triggerNode 调用 triggersAndPollers.runTrigger()
        → trigger 节点的 trigger() 方法被执行，建立外部连接
        → 返回 ITriggerResponse { closeFunction, ... }
        → 存入 activeWorkflows[workflowId].triggerResponses

运行时：
  trigger 持续监听外部事件（MQTT 消息、WebSocket 消息、队列消费等）
  → 事件到达时调用 emit() 回调
  → emit() 触发 workflow 执行

停用时（Leader Stepdown 或手动 Deactivate）：
  activeWorkflows.remove(workflowId)
    → 对每个 triggerResponse 调用 closeFunction()
    → 关闭外部连接（断开 MQTT、关闭 WebSocket、取消订阅等）
    → delete activeWorkflows[workflowId]
```

**Key Design Decision**：Triggers 必须在内存中持续运行，因为它们建立了与外部服务的持久连接。只有 Leader 可以注册它们，避免多个实例同时消费同一条消息。

### 4.4 Pollers 详解

**存储位置**：双存储 —— 内存 Map + CronJob

```typescript
// ActiveWorkflows 中存储 poll 函数引用
activeWorkflows: { [workflowId: string]: IWorkflowData }

// ScheduledTaskManager 中存储 CronJob
cronsByWorkflow: Map<WorkflowId, Map<CronKey, Cron>>
// Cron = { job: CronJob, summary: string, ctx: CronContext }
```

**生命周期**：

```
激活时（仅 Leader）：
  addTriggersAndPollers(dbWorkflow, workflow, ...)
    → activeWorkflows.add(workflowId, workflow, ...)
        → 对每个 pollNode:
            → activatePolling(pollNode, workflow, ...)
                → pollFunctions.getNodeParameter('pollTimes') → 获取 cron 表达式
                → executeTrigger(true) → 立即执行一次（测试）
                → scheduledTaskManager.registerCron(ctx, onTick)
                    → 创建 CronJob 并存入 cronsByWorkflow

运行时：
  CronJob 按 cron 表达式定时触发
    → executeTrigger() 被调用
    → triggersAndPollers.runPoll(workflow, pollNode, pollFunctions)
    → 调用 poll 节点的 poll() 方法
    → 如果有新数据则触发 workflow 执行

停用时（Leader Stepdown 或手动 Deactivate）：
  activeWorkflows.remove(workflowId)
    → scheduledTaskManager.deregisterCrons(workflowId)
        → 对 cronsByWorkflow[workflowId] 中每个 cron:
            → cron.job.stop()
            → 删除 cronsByWorkflow 中的 entry
    → delete activeWorkflows[workflowId]
```

**Key Design Decision**：Pollers 结合了"定时任务"（CronJob）和"轮询逻辑"（poll 函数）。CronJob 在 `ScheduledTaskManager` 中管理，轮询函数在 `ActiveWorkflows` 中管理。两者都在内存中，且都只有 Leader 能创建。

### 4.5 三类资源的协作关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        n8n Multi-Main Cluster                       │
│                                                                     │
│  ┌─────────────────────────────┐    ┌───────────────────────────┐  │
│  │       Follower Instance      │    │       Leader Instance       │  │
│  │                              │    │                             │  │
│  │  WebhookEntity (DB Read)     │    │  ActiveWorkflows (Memory)   │  │
│  │  - 查找 webhook 路由          │    │  ┌───────────────────────┐  │  │
│  │  - 执行 webhook workflow     │    │  │ Triggers:              │  │  │
│  │                              │    │  │  - MQTT connections    │  │  │
│  │                              │    │  │  - WebSocket listeners │  │  │
│  │                              │    │  ├───────────────────────┤  │  │
│  │                              │    │  │ Pollers:               │  │  │
│  │                              │    │  │  - CronJob (定时)       │  │  │
│  │                              │    │  │  - poll() 函数          │  │  │
│  │                              │    │  └───────────────────────┘  │  │
│  │                              │    │                             │  │
│  │                              │    │  ScheduledTaskManager       │  │
│  │                              │    │  (Memory)                   │  │
│  │                              │    │  ┌───────────────────────┐  │  │
│  │                              │    │  │ cronsByWorkflow        │  │  │
│  │                              │    │  │  workflowId → Cron[]   │  │  │
│  │                              │    │  └───────────────────────┘  │  │
│  └─────────────────────────────┘    └───────────────────────────┘  │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                        Redis                              │    │
│  │  - Leader Key (选举)                                        │    │
│  │  - PubSub (add-webhooks-triggers-and-pollers, ...)         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     Database (PostgreSQL/SQLite)           │    │
│  │  - webhook_entity (webhook 持久化)                         │    │
│  │  - workflow_entity (workflow 定义, active 标记)            │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键代码文件索引

| 文件 | 作用 |
|------|------|
| [active-workflow-manager.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/active-workflow-manager.ts) | 工作流激活/停用的核心协调器，包含 PubSub 消息处理和 Leader 事件响应 |
| [multi-main-setup.ee.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/scaling/multi-main-setup.ee.ts) | Leader 选举与切换的实现 |
| [active-workflows.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/core/src/execution-engine/active-workflows.ts) | Triggers 和 Pollers 的内存存储与生命周期管理 |
| [scheduled-task-manager.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/core/src/execution-engine/scheduled-task-manager.ts) | Pollers 的 CronJob 管理 |
| [webhook.service.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/webhooks/webhook.service.ts) | Webhooks 的数据库操作和缓存 |
| [webhook-entity.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/db/src/entities/webhook-entity.ts) | Webhook 数据库实体定义 |
| [pubsub.event-map.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/scaling/pubsub/pubsub.event-map.ts) | PubSub 事件类型与 payload 定义 |
| [pubsub.types.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/scaling/pubsub/pubsub.types.ts) | PubSub 命令类型定义 |
| [instance-settings.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/core/src/instance-settings/instance-settings.ts) | 实例角色（leader/follower/unset）和 multi-main 标志 |
| [triggers-and-pollers.ts](file:///C:/Users/10244/Desktop/0508-under/n8n/packages/core/src/execution-engine/triggers-and-pollers.ts) | Trigger/Poller 节点的执行封装 |
