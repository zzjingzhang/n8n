# n8n 权限体系深度解析：从装饰器到 Scope 的推导全链路

## 目录

- [1. 概述](#1-概述)
- [2. 从装饰器到 Scope 检查的完整调用链](#2-从装饰器到-scope-检查的完整调用链)
  - [2.1 装饰器层：标记元数据](#21-装饰器层标记元数据)
  - [2.2 ControllerRegistry 层：中间件注入](#22-controllerregistry-层中间件注入)
  - [2.3 userHasScopes 层：三层权限推导](#23-userhasscopes-层三层权限推导)
- [3. 三层 Scope 组合机制](#3-三层-scope-组合机制)
  - [3.1 全局 Scope](#31-全局-scope)
  - [3.2 项目角色 Scope](#32-项目角色-scope)
  - [3.3 资源分享角色 Scope](#33-资源分享角色-scope)
  - [3.4 三层组合与 Mask 过滤](#34-三层组合与-mask-过滤)
- [4. 典型端点分析](#4-典型端点分析)
  - [4.1 GET /workflows/:workflowId](#41-get-workflowsworkflowid)
  - [4.2 POST /workflows/:workflowId/activate](#42-post-workflowsworkflowidactivate)
  - [4.3 PUT /workflows/:workflowId/share（凭证分享同构）](#43-put-workflowsworkflowidshare凭证分享同构)
- [5. WorkflowFinderService 与 userHasScopes 的对比](#5-workflowfinderservice-与-userhasscopes-的对比)
  - [5.1 设计定位](#51-设计定位)
  - [5.2 实现差异](#52-实现差异)
  - [5.3 适用场景](#53-适用场景)
- [6. 关键代码索引](#6-关键代码索引)

---

## 1. 概述

n8n 的权限体系是一个**三层嵌套**的权限模型：**全局角色 → 项目角色 → 资源分享角色**。系统通过装饰器（`@ProjectScope` / `@GlobalScope`）在端点上声明所需的权限 scope，然后由控制器注册中心自动注入权限检查中间件，最终通过 `userHasScopes` 函数完成从用户身份到"是否拥有某个 scope"的完整推导。

体系核心设计原则：

- **最短路径优先**：先检查全局角色（最快），再检查项目角色，最后检查资源分享角色
- **数据库层面过滤**：`WorkflowFinderService` 等服务将权限检查直接下推到 SQL 查询层面
- **Mask 机制**：资源分享角色对项目角色的 scope 起过滤（mask）作用，防止权限溢出

---

## 2. 从装饰器到 Scope 检查的完整调用链

### 2.1 装饰器层：标记元数据

装饰器定义在 [scoped.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/decorators/src/controller/scoped.ts) 中：

```typescript
// @ProjectScope 装饰器：检查全局 + 项目 + 资源三层
export const ProjectScope = (scope: Scope) => Scoped(scope);

// @GlobalScope 装饰器：仅检查全局角色
export const GlobalScope = (scope: Scope) => Scoped(scope, { globalOnly: true });
```

装饰器的核心作用是**向 `ControllerRegistryMetadata` 写入元数据**：

```typescript
const Scoped =
  (scope: Scope, { globalOnly } = { globalOnly: false }): MethodDecorator =>
  (target, handlerName) => {
    const routeMetadata = Container.get(ControllerRegistryMetadata)
      .getRouteMetadata(target.constructor as Controller, String(handlerName));
    routeMetadata.accessScope = { scope, globalOnly };
  };
```

此时**不做任何权限检查**，仅将 `{ scope: 'workflow:read', globalOnly: false }` 这样的元数据挂载到路由上。

### 2.2 ControllerRegistry 层：中间件注入

在 [controller.registry.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/controller.registry.ts) 中，`ControllerRegistry.activate()` 遍历所有控制器并注册路由。关键方法是 `buildMiddlewares()`：

```typescript
private buildMiddlewares(route, controllerMiddlewares, bodyDtoClass) {
  const middlewares: RequestHandler[] = [];

  // LAYER 1: IP 限流（在认证前）
  if (inProduction && route.ipRateLimit) { ... }

  // LAYER 2: 认证中间件
  if (!route.skipAuth) {
    middlewares.push(this.authService.createAuthMiddleware({...}), ...);
  }

  // LAYER 3: License 检查
  if (route.licenseFeature) {
    middlewares.push(this.createLicenseMiddleware(route.licenseFeature));
  }

  // LAYER 4: Scope 权限检查 ← 关键！
  if (route.accessScope) {
    middlewares.push(this.createScopedMiddleware(route.accessScope));
  }

  return middlewares;
}
```

`createScopedMiddleware` 方法将装饰器元数据转化为实际的权限检查：

```typescript
private createScopedMiddleware(accessScope: AccessScope): RequestHandler {
  return async (req, res, next) => {
    if (!isAuthenticatedRequest(req)) throw new UnauthenticatedError();

    const { scope, globalOnly } = accessScope;

    if (!(await userHasScopes(req.user, [scope], globalOnly, req.params))) {
      res.status(403).json({ status: 'error', message: MISSING_SCOPE });
      return;
    }
    next();
  };
}
```

注意这里传入了 `req.params`——它包含了 `workflowId` / `credentialId` / `projectId` 等 URL 参数，这些是后续推导项目级和资源级权限的关键。

### 2.3 userHasScopes 层：三层权限推导

[check-access.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/permissions.ee/check-access.ts) 中的 `userHasScopes` 函数是权限推导的核心算法：

```
用户请求 → ① 全局角色检查 → ② 项目角色 + 资源分享角色交叉检查 → 返回 boolean
```

详细流程：

```typescript
export async function userHasScopes(
  user: User,
  scopes: Scope[],
  globalOnly: boolean,
  { credentialId, workflowId, projectId, dataTableId }
): Promise<boolean> {

  // ========== 第一层：全局角色检查（最快路径） ==========
  if (hasGlobalScope(user, scopes, { mode: 'allOf' })) return true;
  // hasGlobalScope 检查用户的全局角色（user.role.scopes）是否包含所有所需 scopes
  // 全局 Owner/Admin 通常拥有所有 scope，此层直接放行

  if (globalOnly) return false; // @GlobalScope 到此为止

  // ========== 第二层：查找用户有权限的项目 ==========
  // 找出用户作为成员的项目中，哪些项目的项目角色包含所需 scopes
  const userProjectIds = (
    await ProjectRepository
      .createQueryBuilder('project')
      .innerJoin('project.projectRelations', 'relation')
      .innerJoin('relation.role', 'role')
      .innerJoin('role.scopes', 'scope')
      .where('relation.userId = :userId', { userId: user.id })
      .andWhere('scope.slug IN (:...scopes)', { scopes })
      .groupBy('project.id')
      .having('COUNT(DISTINCT scope.slug) = :scopeCount', { scopeCount: scopes.length })
      .select(['project.id AS id'])
      .getRawMany()
  ).map(row => row.id);

  // ========== 第三层：查找资源共享记录并交叉验证 ==========
  if (workflowId) {
    // 查出该 workflow 被共享到了哪些项目
    const workflows = await SharedWorkflowRepository.findBy({ workflowId });

    // 查出哪些资源分享角色包含所需 scopes
    const validRoles = await roleService.rolesWithScope('workflow', scopes);

    // 交叉判断：该资源是否被共享到了用户有权限的项目，且共享角色满足要求
    return workflows.some(
      (w) => userProjectIds.includes(w.projectId) && validRoles.includes(w.role),
    );
  }

  if (credentialId) { /* 同构逻辑，额外处理全局凭证 */ }
  if (projectId) { return userProjectIds.includes(projectId); }
}
```

---

## 3. 三层 Scope 组合机制

### 3.1 全局 Scope

全局 scope 由用户的**全局角色**（`user.role`）决定，定义在 [global-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/global-scopes.ee.ts) 中：

| 全局角色 | 典型 Scope 范围 |
|---------|---------------|
| `global:owner` | 几乎所有 scope（包括 `user:manage`, `license:manage` 等管理类） |
| `global:admin` | 与 owner 相同的全部 scope（GLOBAL_ADMIN_SCOPES = GLOBAL_OWNER_SCOPES.concat()） |
| `global:member` | 仅基础 scope（如 `tag:create/read`, `variable:read/list` 等） |
| `global:chatUser` | 仅 `chatHub:message` 和 `chatHubAgent:*` 相关 scope |

**检查方式**：`hasGlobalScope(user, scopes)` 直接读取 `user.role.scopes` 并判断是否包含目标 scope。这是最快的检查路径，Owner/Admin 用户几乎总是在此层被放行。

### 3.2 项目角色 Scope

项目角色定义在 [project-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/project-scopes.ee.ts) 中：

| 项目角色 | 典型 Scope |
|---------|-----------|
| `project:personalOwner` | 个人项目所有者，几乎所有工作流/凭证操作 scope |
| `project:admin` | 团队项目管理员，包含 `credential:share`, `workflow:move` 等 |
| `project:editor` | 编辑器，大部分工作流操作，但不含 `credential:share` |
| `project:viewer` | 仅只读 scope（`workflow:read`, `credential:read` 等） |
| `project:chatUser` | 仅 `agent:execute`, `workflow:execute-chat` |

**注意差异**：`project:personalOwner`（个人项目）**没有** `workflow:publish` 和 `workflow:share`，这是因为个人空间的发布/分享需要通过额外的安全设置控制（`PERSONAL_SPACE_PUBLISHING_SETTING` / `PERSONAL_SPACE_SHARING_SETTING`）。

### 3.3 资源分享角色 Scope

资源分享角色分为 workflow 和 credential 两类：

**Workflow 分享角色**（[workflow-sharing-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/workflow-sharing-scopes.ee.ts)）：

| 角色 | Scopes |
|-----|--------|
| `workflow:owner` | 全部 11 个 scope（read, export, update, publish, unpublish, delete, execute, share, unshare, move, execute-chat） |
| `workflow:editor` | 8 个 scope（不含 share, unshare, move） |

**Credential 分享角色**（[credential-sharing-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/credential-sharing-scopes.ee.ts)）：

| 角色 | Scopes |
|-----|--------|
| `credential:owner` | 全部 6 个 scope（read, update, delete, share, unshare, move） |
| `credential:user` | 仅 `credential:read` |

### 3.4 三层组合与 Mask 过滤

[role.service.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/services/role.service.ts) 中的 `combineResourceScopes` 方法展示了三层 scope 的合并逻辑：

```typescript
combineResourceScopes(type, user, shared, userProjectRelations): Scope[] {
  // 第一层：全局角色的 scopes
  const globalScopes = getAuthPrincipalScopes(user, [type]);
  const scopesSet: Set<Scope> = new Set(globalScopes);

  for (const sharedEntity of shared) {
    // 第二层：用户在该项目中的项目角色 scopes
    const pr = userProjectRelations.find(p => p.projectId === sharedEntity.projectId);
    let projectScopes: Scope[] = [];
    if (pr) {
      projectScopes = pr.role.scopes.map(s => s.slug);
    }

    // 第三层：资源分享角色的 scopes（同时作为 mask）
    const resourceMask = getRoleScopes(sharedEntity.role);

    // 合并：全局 + (项目 ∩ 资源分享角色) + 资源分享角色
    const mergedScopes = combineScopes(
      { global: globalScopes, project: projectScopes },
      { sharing: resourceMask },  // ← 这就是 mask！
    );
    mergedScopes.forEach(s => scopesSet.add(s));
  }
  return [...scopesSet].sort();
}
```

**核心概念 —— Mask 机制**：`combineScopes` 函数（[combine-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/utilities/combine-scopes.ee.ts)）将项目角色的 scopes 用资源分享角色的 scopes 做过滤：

```typescript
export function combineScopes(userScopes: ScopeLevels, masks?: MaskLevels): Set<Scope> {
  // 复制项目级和资源级 scopes
  const maskedScopes = { ...userScopes };

  if (masks?.sharing) {
    // 项目角色的 scopes 必须被资源分享角色的 scopes 包含
    maskedScopes.project = maskedScopes.project.filter(
      v => masks.sharing.includes(v)
    );
    // 资源角色的 scopes 同理
    maskedScopes.resource = maskedScopes.resource.filter(
      v => masks.sharing.includes(v)
    );
  }

  return new Set(Object.values(maskedScopes).flat());
}
```

**图示说明**：

```
全局角色 Scopes  (始终全量添加)
    ↓
项目角色 Scopes  →  被 sharing mask 过滤  →  交集部分添加
    ↓
资源分享角色 Scopes → 被 sharing mask 过滤 →  交集部分添加
    ↓
最终 = 全局 ∪ (项目 ∩ 分享) ∪ (资源 ∩ 分享)
```

**实际案例**：假设用户在某个项目中的角色是 `project:admin`（包含 `workflow:share`），但该工作流只以 `workflow:editor` 角色共享给该项目。`workflow:editor` 的 sharing mask 不包含 `workflow:share`，所以即使用户的项目角色有分享权限，也不能分享这个被以 editor 角色共享的工作流。

---

## 4. 典型端点分析

### 4.1 GET /workflows/:workflowId

**装饰器声明**：`@ProjectScope('workflow:read')`

```typescript
@Get('/:workflowId')
@ProjectScope('workflow:read')
async getWorkflow(req: WorkflowRequest.Get) {
  const { workflowId } = req.params;

  // 装饰器检查通过后，仍需用 WorkflowFinderService 获取有权限的 workflow
  const workflow = await this.workflowFinderService.findWorkflowForUser(
    workflowId, req.user, ['workflow:read'], {...}
  );

  if (!workflow) {
    throw new NotFoundError(`Workflow with ID "${workflowId}" does not exist`);
  }
  // ... 返回 workflow
}
```

**推导链路**：

```
用户请求 GET /workflows/abc123
  → 装饰器中间件调用 userHasScopes(user, ['workflow:read'], false, {workflowId: 'abc123'})
    → ① hasGlobalScope(user, ['workflow:read']) → false（非 admin）
    → ② 查 DB：用户在哪些项目中有包含 'workflow:read' 的项目角色 → ['proj-A', 'proj-C']
    → ③ 查 DB：workflow abc123 被共享到了哪些项目 → [{projectId: 'proj-A', role: 'workflow:editor'}, ...]
    → ④ 查 DB：哪些 workflow 分享角色包含 'workflow:read' → ['workflow:owner', 'workflow:editor']
    → ⑤ 交叉：proj-A 在 userProjectIds 中，且 workflow:editor 在 validRoles 中 → true
  → 中间件放行
  → 控制器调用 workflowFinderService.findWorkflowForUser(...)
    → hasGlobalScope 快速检查 → false
    → 构建 SQL WHERE：role IN (workflow:owner, workflow:editor)
              AND projectRelations.role IN (project:admin, project:editor, project:viewer, project:personalOwner)
              AND projectRelations.userId = user.id
    → 执行 SQL 查询并返回有权限的 workflow
```

### 4.2 POST /workflows/:workflowId/activate

**装饰器声明**：`@ProjectScope('workflow:publish')`

```typescript
@Post('/:workflowId/activate')
@ProjectScope('workflow:publish')
async activate(req, _res, @Param('workflowId') workflowId, @Body body) {
  // 装饰器检查已通过
  const workflow = await this.workflowService.activateWorkflow(req.user, workflowId, {...});
  // ...
}
```

**推导链路**：与 `getWorkflow` 类似，但所需 scope 为 `workflow:publish`。关键差异：

- `workflow:editor` 的分享角色**包含** `workflow:publish`，所以以 editor 角色共享的工作流也可被激活
- 但 `project:personalOwner` 的项目角色**不包含** `workflow:publish`（个人项目需额外安全设置控制）
- 个人项目的工作流如果要激活，需要通过全局角色（admin）或通过发布设置允许

### 4.3 PUT /workflows/:workflowId/share（凭证分享同构）

**注意**：分享端点**没有**使用 `@ProjectScope` 装饰器，而是在方法体内手动调用 `userHasScopes`：

```typescript
@Licensed('feat:sharing')
@Put('/:workflowId/share')
async share(req: WorkflowRequest.Share) {
  const { workflowId } = req.params;
  const { shareWithIds } = req.body;

  // 首先检查是否有读取权限
  const workflow = await this.workflowFinderService.findWorkflowForUser(
    workflowId, req.user, ['workflow:read']
  );
  if (!workflow) throw new ForbiddenError();

  // 对新增的分享，检查是否有 share 权限
  if (toShare.length > 0) {
    const canShare = await userHasScopes(req.user, ['workflow:share'], false, { workflowId });
    if (!canShare) throw new ForbiddenError();
  }

  // 对取消的分享，检查是否有 unshare 权限
  if (toUnshare.length > 0) {
    const canUnshare = await userHasScopes(req.user, ['workflow:unshare'], false, { workflowId });
    if (!canUnshare) throw new ForbiddenError();
  }
  // ... 执行分享操作
}
```

**为什么不用装饰器？** 因为分享端点需要对**不同操作**使用**不同 scope**（`workflow:share` vs `workflow:unshare`），而装饰器只能对整个端点声明一种 scope。因此分享端点选择跳过装饰器，在方法体内根据实际操作类型分别检查。

凭证分享端点（`PUT /credentials/:credentialId/share`）的逻辑完全同构，只是 scope 换成了 `credential:share` / `credential:unshare`，资源类型换成 `credentialId`。

---

## 5. WorkflowFinderService 与 userHasScopes 的对比

### 5.1 设计定位

| 维度 | `userHasScopes` | `WorkflowFinderService.findWorkflowForUser` |
|-----|-----------------|------------------------------------------|
| **定位** | 权限判定函数（回答"能不能"） | 资源获取服务（回答"能拿到什么"） |
| **返回值** | `boolean` | `WorkflowEntity \| null` |
| **关注点** | 是否有权限 | 有权限的具体资源实体 |
| **使用场景** | 装饰器中间件、分享/取消分享等需要精确判定的操作 | 读取/更新/删除等需要获取资源实体的操作 |

### 5.2 实现差异

**`userHasScopes` 的实现路径**：

1. 调用 `hasGlobalScope()` 做快速检查（纯内存操作，读 `user.role.scopes`）
2. 如果需要项目级检查：执行 SQL 查询用户有哪些项目包含所需 scopes
3. 执行 SQL 查询资源的共享记录（`SharedWorkflow` / `SharedCredentials`）
4. 在应用层做 `Array.some()` 交叉判断

**`WorkflowFinderService.findWorkflowForUser` 的实现路径**：

```typescript
async findWorkflowForUser(workflowId, user, scopes, options) {
  let where = {};

  if (!hasGlobalScope(user, scopes, { mode: 'allOf' })) {
    // 并行查询项目角色和资源角色中包含所需 scopes 的角色
    const [projectRoles, workflowRoles] = await Promise.all([
      this.roleService.rolesWithScope('project', scopes),
      this.roleService.rolesWithScope('workflow', scopes),
    ]);

    // 将权限条件直接下推到 SQL WHERE 子句
    where = {
      role: In(workflowRoles),
      project: {
        projectRelations: {
          role: In(projectRoles),
          userId: user.id,
        },
      },
    };
  }

  // 单次 SQL 查询即可获取有权限的资源
  const sharedWorkflow = await this.sharedWorkflowRepository
    .findWorkflowWithOptions(workflowId, { where, ... });

  return sharedWorkflow?.workflow ?? null;
}
```

**核心差异**：

- `userHasScopes` 是**两步查询 + 应用层交叉**（先查用户有权的项目，再查资源共享记录，再在 JS 中做 some 判断）
- `WorkflowFinderService` 是**单步 SQL 查询 + WHERE 过滤**（将权限条件编译成 SQL，一次查出结果）

### 5.3 适用场景

**使用 `userHasScopes` 的场景**：

- **装饰器中间件**（[controller.registry.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/controller.registry.ts#L232-L256)）：需要统一的权限拦截，返回 boolean 即可决定放行还是 403
- **分享/取消分享操作**（[workflows.controller.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflows.controller.ts#L593-L607)）：需要根据操作类型分别检查不同的 scope
- **外部 secrets 权限验证**（[validation.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/credentials/validation.ts#L73)）：检查 `externalSecret:list` 权限
- **项目级操作前的权限预检**：不需要获取资源实体，只需要判断"能不能做这件事"

**使用 `WorkflowFinderService` 的场景**：

- **读取工作流**（`getWorkflow`）：需要返回有权限的工作流实体
- **更新/删除工作流**（`update` / `delete`）：需要先获取目标工作流再执行操作
- **分享前的存在性检查**（`share` 端点）：确认工作流存在且用户有读取权限
- **批量获取工作流**（`findWorkflowsByIdsForUser`）：从一批 ID 中过滤出有权限的

**何时同时使用两者？**

```typescript
// 典型模式：先用 WorkflowFinderService 获取资源，再用 userHasScopes 做精确判定
const workflow = await this.workflowFinderService.findWorkflowForUser(
  workflowId, req.user, ['workflow:read']  // 用 read scope 获取资源
);
if (!workflow) throw new ForbiddenError();

// 但真正的操作需要更高权限（share）
const canShare = await userHasScopes(req.user, ['workflow:share'], false, { workflowId });
if (!canShare) throw new ForbiddenError();
```

**选择原则**：
- 如果**只需要判断权限**，不需要资源实体 → 用 `userHasScopes`
- 如果**需要获取资源实体**，同时自然地过滤无权限 → 用 `WorkflowFinderService`
- 如果**先需要资源实体，后续操作需要不同的更高权限** → 先用 FinderService，再用 `userHasScopes` 检查操作权限

---

## 6. 关键代码索引

| 文件 | 作用 |
|-----|------|
| [scoped.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/decorators/src/controller/scoped.ts) | `@ProjectScope` / `@GlobalScope` 装饰器定义 |
| [controller.registry.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/controller.registry.ts) | 控制器注册与 scope 中间件注入 |
| [controller-registry-metadata.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/decorators/src/controller/controller-registry-metadata.ts) | 路由元数据存储（accessScope 等） |
| [check-access.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/permissions.ee/check-access.ts) | `userHasScopes` 核心权限推导函数 |
| [workflow-finder.service.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflow-finder.service.ts) | `WorkflowFinderService` 数据库层面的权限过滤 |
| [credentials-finder.service.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/credentials/credentials-finder.service.ts) | 凭证版本的 FinderService |
| [role.service.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/services/role.service.ts) | `combineResourceScopes` / `rolesWithScope` 等角色相关方法 |
| [has-global-scope.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/utilities/has-global-scope.ee.ts) | `hasGlobalScope` 全局角色 scope 检查 |
| [combine-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/utilities/combine-scopes.ee.ts) | `combineScopes` 三层 scope 合并与 mask 过滤 |
| [has-scope.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/utilities/has-scope.ee.ts) | `hasScope` 通用 scope 判断（支持 oneOf/allOf） |
| [global-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/global-scopes.ee.ts) | 全局角色的 scope 定义 |
| [project-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/project-scopes.ee.ts) | 项目角色的 scope 定义 |
| [workflow-sharing-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/workflow-sharing-scopes.ee.ts) | Workflow 分享角色的 scope 定义 |
| [credential-sharing-scopes.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/roles/scopes/credential-sharing-scopes.ee.ts) | Credential 分享角色的 scope 定义 |
| [constants.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/constants.ee.ts) | 资源-操作映射常量（RESOURCES） |
| [types.ee.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/permissions/src/types.ee.ts) | 核心类型定义（Scope, ScopeLevels, AuthPrincipal 等） |
| [workflows.controller.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/workflows/workflows.controller.ts) | Workflows 控制器（典型端点使用案例） |
| [credentials.controller.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/credentials/credentials.controller.ts) | Credentials 控制器（凭证分享端点案例） |
