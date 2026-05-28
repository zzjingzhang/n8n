# n8n 二进制数据模式说明

本文说明 n8n 对二进制数据（workflow 执行过程中节点产生的 `IBinaryData` 文件，例如 Webhook 上传、HTTP 响应体、Read/Write Files 节点等）的四种存储模式，以及每种模式如何把抽象的 `store / copy / get / delete / rename` 操作映射到具体的后端存储（本地磁盘、数据库、对象存储）。最后分析从 queue 模式默认 database，到签名 URL secret 初始化之间的依赖顺序。

核心实现位于：

- 配置：[binary-data.config.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/binary-data.config.ts)
- 服务层：[binary-data.service.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/binary-data.service.ts)
- Filesystem 管理器：[file-system.manager.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/file-system.manager.ts)
- ObjectStore/S3 管理器：[object-store.manager.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/object-store.manager.ts)
- Database 管理器：[database.manager.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/binary-data/database.manager.ts)
- 启动装配：[base-command.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/commands/base-command.ts)
- `IBinaryData.id` 还原：[restore-binary-data-id.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/execution-lifecycle/restore-binary-data-id.ts)

## 1. 配置如何决定最终使用的 manager

`BinaryDataConfig` 暴露三个关键值：

| 字段 | 环境变量 | 默认值 | 作用 |
| --- | --- | --- | --- |
| `mode` | `N8N_DEFAULT_BINARY_DATA_MODE` | 见下方 | 写入时使用的后端 |
| `availableModes` | （暂未直接 env 驱动） | `['filesystem','s3','database']` | 允许装配的后端集合 |
| `localStoragePath` | `N8N_BINARY_DATA_STORAGE_PATH` → `N8N_STORAGE_PATH` → `~/.n8n/storage` | filesystem 模式的根目录 |

`mode` 的自动默认：

```ts
this.mode ??= executionsConfig.mode === 'queue' ? 'database' : 'filesystem';
```

- `EXECUTIONS_MODE=regular`（主进程内执行）→ 默认 `filesystem`
- `EXECUTIONS_MODE=queue`（worker 分布式执行）→ 默认 `database`

这样做的原因是：在 queue 模式下，主、worker、webhook 三个进程都可能读写同一个二进制数据，只有 DB（以及可选的 S3）是所有进程天然共享的存储介质。本地文件系统只在单进程 regular 模式下是可靠的。

`mode` 的最终可用值由 `BINARY_DATA_MODES = ['default', 'filesystem', 's3', 'database']` 约束，其中 `default` 表示“不持久化”，二进制数据以 base64 形式直接内联到 `IBinaryData.data`。

### 1.1 manager 的装配顺序

所有 manager 都挂在 `BinaryDataService.managers` 这个 Record 上。装配发生在 `initBinaryDataService()`（[base-command.ts#L246-L288](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/commands/base-command.ts#L246-L288)）：

1. **无条件注册 DatabaseManager**（`setManager('database', …)`），因为 queue 模式会默认选它，并且其它 manager 的 temp 文件也可能要走 DB 迁移。
2. 如果 `ObjectStoreConfig.bucket.name !== ''`，尝试连接 S3；成功就 `setManager('s3', new ObjectStoreManager(…))`。
3. `await binaryDataService.init()` 内部：
   - 若 `config.mode === 'filesystem'`，转成 `filesystem-v2`（内部 mode 名称，逻辑一致，只是和历史 `filesystem` 区分）。
   - `new FileSystemManager(localStoragePath)` 同时注册到 `filesystem` 和 `filesystem-v2` 两个 key，保持对老数据的兼容性。
   - 不注册 `default` manager，因为 `default` 根本不走 manager，而是把 `IBinaryData.data` 写在内存里。

因此最终参与选择的 manager 集合为：

| mode | 是否有 manager | 实现 |
| --- | --- | --- |
| `default` | 否，直接内联到 `IBinaryData.data` | — |
| `filesystem` / `filesystem-v2` | 是 | `FileSystemManager` |
| `s3` | 有则是（已配置 `N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME`） | `ObjectStoreManager` |
| `database` | 是 | `DatabaseManager` |

## 2. 为什么 `binaryData.id` 要带 mode 前缀

在 `BinaryDataService.createBinaryDataId(fileId)` 里：

```ts
return `${this.mode}:${fileId}`;
```

这样 `IBinaryData.id` 的形态是 `<mode>:<fileId>`，例如：

- `filesystem-v2:workflows/123/executions/390/binary_data/69055-83c4-…`
- `s3:workflows/123/executions/390/binary_data/69055-83c4-…`
- `database:550e8400-e29b-41d4-a716-446655440000`
- `default` 模式下 `binaryData.id` 为空，数据直接塞在 `binaryData.data` 里

这样设计有两个目的：

1. **读取时可自路由**。`getAsStream / getAsBuffer / getPath / getMetadata / deleteManyByBinaryDataId` 都是先 `split(':')` 得到 `[mode, fileId]`，再用 `getManager(mode)` 分派到对应后端。
2. **跨模式兼容**。n8n 在升级过程中可能切换后端（例如 `filesystem` → `s3`，或 `default` → `database`），历史 execution JSON 里的 `binaryData.id` 仍能被新代码正确读取。

注意：`BinaryDataConfig.mode` 只决定"写入用哪个 manager"；读取完全由 `binaryData.id` 前缀决定。这也是为什么 `restore-binary-data-id.ts` 会保留原来的 mode，只替换 `temp` 为真实 executionId。

## 3. 三种后端如何映射文件路径 / 数据库行 / 对象存储 key

三种后端都共享 `BinaryData.Manager` 接口，但对同一个抽象概念的具体实现差异很大。下面按操作逐一列出。

### 3.1 统一抽象

- `FileLocation`（[types.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/types.ts#L37-L39)）：
  - `{ type: 'execution', workflowId, executionId }` —— 代表一次 workflow 执行
  - `{ type: 'custom', pathSegments, sourceType?, sourceId? }` —— 代表自定义分类（chat-hub attachments 等）
- `fileId`：后端自己生成的 key（可能是 `uuid`，也可能是相对路径）
- `binaryData.id = <mode>:<fileId>`

### 3.2 FileSystemManager（`filesystem` / `filesystem-v2`）

根目录由 `BinaryDataConfig.localStoragePath` 决定（默认 `~/.n8n/storage`），所有路径都经 `resolvePath(...)` 再走一次路径穿越校验，防止 `fileId` 中出现 `..` 跳出根目录。

| 操作 | 行为 |
| --- | --- |
| `store` | `fileId = "<toRelativePath(location)>/binary_data/<uuid>"`，写 `storagePath + fileId`；同目录下多一份 `<fileId>.metadata` JSON（`{fileName, mimeType, fileSize}`） |
| `copyByFilePath` | 从外部 temp 文件 `cp` 到 `storagePath + targetFileId`，再写 `.metadata` |
| `copyByFileId` | 读取原 `.metadata`，`fs.copyFile` 到新路径，再写新 `.metadata` |
| `getAsBuffer / getAsStream` | `fs.readFile` / `createReadStream(storagePath + fileId)` |
| `getMetadata` | `jsonParse(fs.readFile(storagePath + fileId + '.metadata'))` |
| `getPath` | `storagePath + fileId` |
| `deleteMany(locations)` | 对每个 location 计算对应目录，`fs.rm(recursive, force)` 整目录删除 |
| `deleteManyByFileId(ids)` | 先 `parseFileId` 还原 location，再复用 `deleteMany` |
| `rename` | `fs.rename(oldPath, newPath)` + `fs.rename(oldPath.metadata, newPath.metadata)`；若旧路径含 `/temp/`，顺带 `rm` 掉 temp 目录 |

`toRelativePath` 的行为：

- `execution` → `workflows/<workflowId>/executions/<executionId or 'temp'>`
- `custom` → `pathSegments.join('/')`

`parseFileId` 通过正则 `^workflows/([^/]+)/executions/([^/]+)/` 反向还原 location；匹配不到就退回 custom 路径，取 `/binary_data/` 之前的部分切分。

### 3.3 DatabaseManager（`database`）

使用 `BinaryDataRepository`（TypeORM），每一条记录即一个二进制文件。schema 关键字段：

- `fileId`（PK，uuid）
- `sourceType`（`execution` / chat-hub 等）
- `sourceId`（executionId 或 chat-hub sessionId 等）
- `data`（BYTEA / BLOB）
- `fileName / mimeType / fileSize`

| 操作 | 行为 |
| --- | --- |
| `store` | `fileId = uuid()`，校验 `fileSize <= dbMaxFileSize`（默认 512 MiB，上限 1024 MiB），然后 `repository.insert(...)` |
| `copyByFilePath` | 同上：`fs.readFile` 临时文件 → `repository.insert`，因为 BYTEA 不支持“文件到列”的复制 |
| `copyByFileId` | 调用 `repository.copyStoredFile(sourceFileId, targetFileId, sourceType, sourceId)`（SQL 层 `INSERT ... SELECT`） |
| `getAsBuffer` | `findOneOrFail({where:{fileId}, select:['data']})` |
| `getAsStream` | `Readable.from(getAsBuffer(fileId))`（因为 BYTEA 本身是整列，不支持游标式流式读取） |
| `getMetadata` | `findOneOrFail({..., select:['fileName','mimeType','fileSize']})` |
| `getPath` | 返回字符串 `"database://<fileId>"`（仅用于日志/标识，不是可访问路径） |
| `deleteMany(locations)` | `repository.delete({ sourceType:'execution', sourceId: In(executionIds) })`（只对 execution 类型有效） |
| `deleteManyByFileId(ids)` | `repository.delete({ fileId: In(ids) })` |
| `rename` | `repository.update({fileId:old}, {fileId:new})`，`affected===0` 抛 `BinaryDataFileNotFoundError` |

与 filesystem / s3 不同，DatabaseManager 的 `fileId` 与 `location` 是**解耦**的：`location` 被压成 `sourceType + sourceId` 作为查询维度（方便一次性清理某个 execution），而文件本身的身份是 uuid。这也是为什么 `copyByFileId` 仍然需要再传一次 `targetLocation`，以便写入新行的 `sourceType/sourceId`。

### 3.4 ObjectStoreManager（`s3`）

委托给 `ObjectStoreService`（S3 兼容对象存储）。S3 key 直接等同于 `fileId`，因此 `getPath(fileId)` 只是原样返回。

| 操作 | 行为 |
| --- | --- |
| `store` | `fileId = "<prefix>/binary_data/<uuid>"`，`objectStoreService.put(fileId, buffer, metadata)` → `PutObjectCommand` |
| `copyByFilePath` | `fs.readFile(sourcePath)` → `put(targetFileId, buffer, metadata)` |
| `copyByFileId` | `get(sourceFileId, buffer)` → `put(targetFileId, buffer)`（S3 CopyObject 没有用，因为要保证和 put 的元数据逻辑一致） |
| `getAsBuffer` | `GetObjectCommand` → Buffer |
| `getAsStream` | `GetObjectCommand` Body 是 `Readable`，按需用 `createFixedSizeChunker` 切块 |
| `getMetadata` | `HeadObjectCommand`，读取 `content-length / content-type / x-amz-meta-filename` |
| `getPath` | 直接返回 `fileId` |
| `deleteMany` | **没有实现**（注释说明"S3 生命周期配置负责"）。所以 `BinaryDataService.deleteMany` 会检查 `manager.deleteMany` 是否存在，不存在就跳过。 |
| `deleteManyByFileId` | 也没有实现 |
| `rename` | `get(oldFileId)` → `put(newFileId, oldFile, oldFileMetadata)` → `deleteOne(oldFileId)`（S3 无原生 rename） |

`toFileId` 的行为：

- `execution` → `workflows/<workflowId>/executions/<executionId or 'temp'>/binary_data/<uuid>`
- `custom` → `<pathSegments.join('/')>/binary_data/<uuid>`

和 filesystem 完全同构，只是不再拼本地根目录，key 直接就是上面的相对路径。

### 3.5 差异对照

| 维度 | filesystem | database | s3 |
| --- | --- | --- | --- |
| fileId 构成 | 相对路径 + uuid | uuid | 相对路径 + uuid |
| location 编码 | 写入 fileId | 写成 sourceType/sourceId 列 | 写入 fileId |
| metadata 存放 | 伴生 `.metadata` JSON 文件 | 同表列 | S3 user-defined metadata 头 |
| 流式读取 | 原生 | 全量读再转 stream | 原生 |
| 按 execution 删除 | `rm -rf` 目录 | `WHERE sourceType='execution' AND sourceId IN (...)` | 依赖生命周期策略，不实现 |
| 按 fileId 批量删除 | 先反查 location 再 `rm -rf` | `WHERE fileId IN (...)` | 未实现 |
| rename | 原子 `fs.rename`（主+metadata） | `UPDATE fileId` | 下载+上传+删除（非原子） |

## 4. queue 模式默认 database → 签名 URL secret 初始化之间的依赖顺序

本节只针对 **main（start.ts）和 worker（worker.ts）** 的启动流程；webhook 命令走同一套（见 [webhook.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/commands/webhook.ts)）。

### 4.1 配置阶段：决定默认 mode

在 `ExecutionsConfig`（[executions.config.ts#L81-L89](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/@n8n/config/src/configs/executions.config.ts#L81-L89)）里，`EXECUTIONS_MODE` 决定 execution 运行方式：

```ts
mode: ExecutionMode = 'regular'; // 'regular' | 'queue'
```

然后 `BinaryDataConfig.constructor` 用它决定二进制数据的默认后端：

```ts
this.mode ??= executionsConfig.mode === 'queue' ? 'database' : 'filesystem';
```

同时 `signingSecret` 先从 `encryptionKey` 派生：

```ts
this.signingSecret = createHash('sha256').update(`url-signing:${encryptionKey}`).digest('base64');
```

这一步在 `@n8n/config` 构建期同步执行，不依赖数据库。所以"queue→默认 database"和"signingSecret 默认值"是两个独立的默认值，都不依赖对方。

### 4.2 启动阶段：异步初始化顺序

`start.ts` / `worker.ts` 的关键调用顺序：

```ts
// start.ts (节选)
await this.instanceSettings.initialize(DeploymentKeyRepository);         // ① hostId/角色协商
await Container.get(JwtService).initialize(DeploymentKeyRepository);      // ② JWT 密钥
await Container.get(BinaryDataConfig).initialize(DeploymentKeyRepository); // ③ 签名 secret 初始化
...
await this.initBinaryDataService();       // ④ 注册 DB/S3 manager 并 init()
```

其中 ③ `BinaryDataConfig.initialize(repo)`（[binary-data.config.ts#L72-L97](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/binary-data.config.ts#L72-L97)）做的事：

1. 如果 `N8N_BINARY_DATA_SIGNING_SECRET` 已设置，直接返回（**不** 落库，保持用户覆盖）。
2. 否则查 `deployment_key` 表的 `type='signing.binary_data'` 激活行；存在就改用库里的值。
3. 否则把 constructor 阶段派生的值 `insertOrIgnore` 进 `deployment_key`；然后再读一次 winner（因为多 main 并发下可能被别人先写入了），最后采用 winner。

这样，**所有实例最终使用同一个 signing secret**（要么都是 env 覆盖，要么都是 DB 里同一行的值）。

### 4.3 依赖链

```
EXECUTIONS_MODE=queue
        │
        ▼
BinaryDataConfig.mode 默认为 'database'            (同步、无 DB 依赖)
        │
        ▼
initBinaryDataService() 注册 DatabaseManager        (需要 DB 连接已就绪)
        │
        └──────────────┐
                       │ (两条链在这里汇合)
                       ▼
BinaryDataConfig.initialize(DeploymentKeyRepository) (需要 DB migration 已完成)
        │
        ▼
signingSecret (env → DB → derive-from-encryptionKey 三选一)
        │
        ▼
BinaryDataService.createSignedToken(...)             (运行时生成 JWT)
        │
        ▼
GET /binary-data/signed?token=... (BinaryDataController)
                                   └─ validateSignedToken(token)
                                   └─ getAsStream(binaryDataId)
```

几个关键点：

1. **`mode === 'database'` 与 signing secret 是两件事**。前者决定"二进制数据写哪"，后者决定"签名 URL 的 HMAC/JWT 密钥是什么"。两者都在 `BinaryDataConfig` 里，但初始化路径独立。
2. **`BinaryDataConfig.initialize` 必须在 DB migrations 之后**（它对 `deployment_key` 表做查询和插入）。这也是为什么它在 `start.ts` 里放在 `initBinaryDataService` 之前、但又在 `instanceSettings.initialize` 之后 —— 前者要求 DB 可用，后者用同一个 `DeploymentKeyRepository` 保证顺序。
3. **签名 secret 的优先级**：env > DB active row > 从 encryptionKey 派生（并尝试持久化）。这使得 docker/K8s 部署可以用 env 注入，而没有 env 的单机模式能在 DB 中稳定复用第一次派生的值。
4. **signed URL 与 mode 无关**：`createSignedToken` 只依赖 `binaryData.id`（要求非空，即非 `default` 模式）和 `signingSecret`；不管 mode 是 database/s3/filesystem，最终都是把一个 `<mode>:<fileId>` 放进 JWT payload 里，请求到达 `/binary-data/signed` 时再按前缀路由。
5. **worker 命令强制 queue**：`worker.ts` 构造函数里 `if (executions.mode !== 'queue') executions.mode = 'queue'`，所以 worker 必然走 database 默认。webhook 命令同理会把自己的 executions mode 提升到 queue（webhook 命令里也会设置），以便和主进程共用 binary data 存储。
6. **多 main 环境下的一致性**：`insertOrIgnore` + 二次读取 winner 让多个 main 并发启动时，最终都收敛到同一行 `signing.binary_data`，避免多实例签发的 token 互相不可验。

## 5. 小结

- n8n 二进制数据支持 `default`（内联到 `IBinaryData.data`）、`filesystem` / `filesystem-v2`（本地磁盘）、`s3`（对象存储，需要企业版 license）和 `database`（BYTEA/BLOB）四种模式。
- 最终 manager 由 `N8N_DEFAULT_BINARY_DATA_MODE` 决定；未设置时，regular 模式默认 filesystem，queue 模式默认 database。
- `IBinaryData.id = <mode>:<fileId>` 让同一条历史 execution 在后端切换后仍能自路由到正确的 manager。
- 三个后端对 `store / copy / get / delete / rename` 的具体映射完全不同：filesystem 以 `storagePath + fileId` 为文件，database 以 uuid 为主键、按 `sourceType/sourceId` 聚合，s3 以同构相对路径作为 key。
- queue → 默认 database 和 signing secret 的初始化是两条独立的配置链；`BinaryDataConfig.initialize` 在 DB migrations 之后执行，负责把 signing secret 统一到 `deployment_key` 表，从而让多实例/多进程对签名 URL 的验签保持一致。
