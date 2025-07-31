# API 流程概述

本文档概述 KMFDDM 项目的主要业务流程，并列出每个 API 请求的大致函数调用顺序，帮助理解整体逻辑。

## 1. 声明（Declaration）管理流程

* **PUT /v1/declarations** → `PutDeclarationHandler`
  1. 读取请求体并使用 `ddm.ParseDeclaration` 解析声明。
  2. 调用 `store.StoreDeclaration` 保存声明。
     - 文件存储实现中继续执行 `writeDeclarationFiles`，写入 JSON、Token 和声明项，随后
       `writeDeclarationDDM` 根据关联的 Enrollment 生成各自的声明文件与同步 Token。
     - MySQL 实现则在 SQL 语句中直接计算 `server_token` 并插入或更新记录。
  3. 若有变更则依据查询参数决定是否调用 `notifier.Changed` 通知设备。

* **GET /v1/declarations/:id** → `GetDeclarationHandler`
  1. 调用 `store.RetrieveDeclaration` 根据 ID 读取声明。
     - 文件存储的 `RetrieveDeclaration` 会进一步调用 `readDeclarationFile` 从磁盘
       读取 JSON 并通过 `ddm.ParseDeclaration` 解析。
     - 数据库存储则执行查询并组装 `ddm.Declaration` 结构。
  2. 处理完毕后直接返回原始声明 JSON。

* **DELETE /v1/declarations/:id** → `DeleteDeclarationHandler`
  1. 调用 `store.DeleteDeclaration` 删除声明。
     - 在文件存储中该过程会检查声明是否仍在任何集合中，并删除相关 JSON、Token 和索引文
       件。
     - MySQL 实现则直接执行 `DELETE` 语句移除记录。

* **POST /v1/declarations/:id/touch** → `TouchDeclarationHandler`
  1. 调用 `store.TouchDeclaration` 更新声明的 `ServerToken`。
     - 文件存储会重新计算声明哈希并重写磁盘文件，同时触发 `writeDeclarationDDM` 更新对
       应 Enrollment 的 Token。
     - MySQL 实现通过 `UPDATE` 语句递增 `touched_ct` 字段并更新 `server_token`。
  2. 若设置通知则调用 `notifier.Changed`。

* **GET /v1/declarations** → `GetDeclarationsHandler`
  1. 调用 `store.RetrieveDeclarations` 列出所有声明 ID。
     - 文件存储通过遍历磁盘文件名获得声明列表。
     - MySQL 存储执行 `SELECT identifier FROM declarations` 返回结果。

## 2. 集合（Set）相关流程

* **GET /v1/sets** → `GetSetsHandler`
  1. 调用 `store.RetrieveSets` 获取全部集合名称。
     - 文件存储扫描磁盘上的 `set.*` 文件名以得到集合列表。
     - MySQL 存储执行 `SELECT DISTINCT set_name FROM set_declarations`。

* **PUT /v1/set-declarations/:id?declaration=DECL** → `PutSetDeclarationHandler`
  1. 从查询参数获取声明 ID。
  2. 调用 `store.StoreSetDeclaration` 将声明加入集合。
     - 文件后端在集合与声明的索引文件中增加记录，并调用 `writeSetDDM` 重新生成所
       有相关 Enrollment 的声明项和 Token。
     - MySQL 后端执行 `INSERT INTO set_declarations` 完成关联。
  3. 如有变更并允许通知，调用 `notifier.Changed`。

* **DELETE /v1/set-declarations/:id?declaration=DECL** → `DeleteSetDeclarationHandler`
  1. 调用 `store.RemoveSetDeclaration` 将声明从集合移除。
     - 文件存储修改索引文件并调用 `writeSetDDM` 更新相关 Enrollment 的声明文件。
     - MySQL 存储执行 `DELETE FROM set_declarations` 完成解除关联。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

* **GET /v1/set-declarations/:id** → `GetSetDeclarationsHandler`
  1. 调用 `store.RetrieveSetDeclarations` 获取集合内的声明列表。
     - 文件存储从 `set.declarations.*` 文本文件读取声明 ID 列表。
     - MySQL 实现执行查询 `SELECT declaration_identifier FROM set_declarations WHERE set_name=?`。

* **GET /v1/declaration-sets/:id** → `GetDeclarationSetsHandler`
  1. 调用 `store.RetrieveDeclarationSets` 获取指定声明所在的集合列表。
     - 文件存储读取 `declaration.<id>.sets.txt` 以得到集合名。
     - MySQL 存储执行查询 `SELECT set_name FROM set_declarations WHERE declaration_identifier=?`。

## 3. Enrollment 与集合关系

* **GET /v1/enrollment-sets/:id** → `GetEnrollmentSetsHandler`
  1. 调用 `store.RetrieveEnrollmentSets` 获取指定 EnrollmentID 的集合列表。
     - 文件存储读取 Enrollment 目录下的 `sets.txt`。
     - MySQL 实现查询 `SELECT set_name FROM enrollment_sets WHERE enrollment_id=?`。

* **PUT /v1/enrollment-sets/:id?set=NAME** → `PutEnrollmentSetHandler`
  1. 调用 `store.StoreEnrollmentSet` 将集合关联到 EnrollmentID。
     - 文件存储在集合与 enrollment 的索引文件中新增记录，并调用 `writeEnrollmentDDM`
       生成新 Token。
     - MySQL 后端执行 `INSERT INTO enrollment_sets`。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

* **DELETE /v1/enrollment-sets/:id?set=NAME** → `DeleteEnrollmentSetHandler`
  1. 调用 `store.RemoveEnrollmentSet` 解除集合关联。
     - 文件存储会更新索引并重新生成 enrollment 相关的 DDM 文件。
     - MySQL 后端执行 `DELETE FROM enrollment_sets`。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

* **DELETE /v1/enrollment-sets-all/sets/:id** → `DeleteAllEnrollmentSetsHandler`
  1. 调用 `store.RemoveAllEnrollmentSets` 移除该 EnrollmentID 的所有集合关系。
     - 文件存储删除 `sets.txt` 并更新 `writeEnrollmentDDM`。
     - MySQL 实现调用预定义查询批量删除。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

## 4. 状态查询流程

* **GET /v1/declaration-status/:id** → `GetDeclarationStatusHandler`
  1. 根据多个 EnrollmentID 调用 `store.RetrieveDeclarationStatus` 获取最后一次声明状态。
     - 文件存储从 CSV 文件读取并组合 `ddm.DeclarationQueryStatus` 列表。
     - MySQL 存储查询视图 `GetDeclarationStatus` 返回当前声明状态及 `Current` 字段。

* **GET /v1/status-errors/:id** → `GetStatusErrorsHandler`
  1. 调用 `store.RetrieveStatusErrors` 获取错误列表。
     - 文件存储解析保存的 CSV 并反序列化错误 JSON。
     - MySQL 存储按时间顺序查询 `status_errors`。

* **GET /v1/status-values/:id?prefix=PATH** → `GetStatusValuesHandler`
  1. 调用 `store.RetrieveStatusValues` 获取指定前缀的值列表。
     - 文件存储读取并过滤 `status.values` CSV。支持简单的前缀/后缀匹配语法。
     - MySQL 存储在 `status_values` 表中使用 `LIKE` 检索。

* **GET /v1/status-report/:id** → `GetStatusReportHandler`
  1. 构造 `StatusReportQuery`，调用 `store.RetrieveStatusReport` 取回完整报告并设置相关响应头。
     - 文件存储仅保存最后一次报告，在磁盘读取 `status.last.json`。
     - MySQL 存储根据索引或状态 ID 查询 `status_reports`。

## 5. Notifier 接口

* **POST /v1/notify** → `NotifyHandler`
  1. 直接调用 `notifier.Changed`，根据查询参数传入声明、集合或 EnrollmentID 列表。

## 6. 设备端 DDM 流程

* **GET /v1/ddm/declaration/<type>/<id>** → `ddm.DeclarationHandler`
  1. 从请求头获取 EnrollmentID，解析路径得到声明类型与 ID。
  2. 调用 `store.RetrieveEnrollmentDeclarationJSON` 获取声明 JSON 并返回。
     - 文件存储直接读取 Enrollment 目录下生成的声明文件。
     - MySQL 实现通过 `GetDDMDeclaration` 查询确认声明与设备的关联后返回 JSON。

* **GET /v1/ddm/items** → `ddm.TokensOrDeclarationItemsHandler`
  1. 调用 `store.RetrieveDeclarationItemsJSON` 返回 Declaration Items。
     - `storage.JSONAdapt` 根据 Enrollment 关联的所有声明构建 `build.DIBuilder` 并计算 Tokens。
     - 结果 JSON 由存储适配层或后端返回。

* **GET /v1/ddm/tokens** → `ddm.TokensOrDeclarationItemsHandler`（tokens=true）
  1. 调用 `store.RetrieveTokensJSON` 返回同步 Token。
     - 与上一接口类似，适配层使用 `build.TokensBuilder` 动态计算 `DeclarationsToken`。

* **POST /v1/ddm/status** → `ddm.StatusReportHandler`
  1. 读取并解析状态报告 `ddm.ParseStatus`。
  2. 调用 `store.StoreDeclarationStatus` 记录状态。
     - 文件存储将数据追加到 CSV，并保存最后一次报告至 `status.last.json`。
     - MySQL 存储在多张表（`status_reports`、`status_values` 等）中写入数据，并可设置保留行数。

以上流程构成了 KMFDDM 的主要业务逻辑：管理员通过 API 管理声明、集合及其与 Enrollment 的关系；设备端通过 DDM 接口获取声明和 Token 并上报状态；当数据变更时 `Notifier` 会向对应设备发送 MDM 命令以触发更新。业务的核心即是围绕这些 API 请求与存储层的交互。
