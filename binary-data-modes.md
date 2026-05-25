# n8n 二进制数据模式说明

本文说明 n8n 对二进制数据（workflow 执行过程中节点产生的 `IBinaryData` 文件，例如 Webhook 上传、HTTP 响应体、Read/Write Files 节点等）的四种存储模式，以及每种模式如何把抽象的 `store / copy / get / delete / rename` 操作映射到具体的后端存储（本地磁盘、数据库、对象存储）。最后分析从 queue 模式默认 database，到签名 URL secret 初始化之间的依赖顺序。

核心实现位于：

- 配置：[binary-data.config.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/binary-data.config.ts)
- 服务层：[binary-data.service.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/binary-data.service.ts)
- Filesystem 管理器：[file-system.manager.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/file-system.manager.ts)
- ObjectStore/S3 管理器：[object-store.manager.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/core/src/binary-data/object-store.manager.ts)
- Database 管理器：[database.manager.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/binary-data/database.manager.ts)
- 启动装配：[base-command.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages/cli/src/commands/base-command.ts)
- `IBinaryData.id` 还原：[restore-binary-data-id.ts](file:///c:/Users/10244/Desktop/0508-under/n8n/packages