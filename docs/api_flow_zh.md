# API 流程概述

本文档概述 KMFDDM 项目的主要业务流程，并列出每个 API 请求的大致函数调用顺序，帮助理解整体逻辑。

## 1. 声明（Declaration）管理流程

* **PUT /v1/declarations** → `PutDeclarationHandler`
  1. 读取请求体并使用 `ddm.ParseDeclaration` 解析声明。
  2. 调用 `store.StoreDeclaration` 保存声明，若有变更则依据查询参数决定是否调用 `notifier.Changed` 通知设备。

* **GET /v1/declarations/:id** → `GetDeclarationHandler`
  1. 通过 `store.RetrieveDeclaration` 获取指定声明。
  2. 直接返回原始声明 JSON。

* **DELETE /v1/declarations/:id** → `DeleteDeclarationHandler`
  1. 调用 `store.DeleteDeclaration` 删除声明。

* **POST /v1/declarations/:id/touch** → `TouchDeclarationHandler`
  1. 调用 `store.TouchDeclaration` 更新声明的 `ServerToken`。
  2. 若设置通知则调用 `notifier.Changed`。

* **GET /v1/declarations** → `GetDeclarationsHandler`
  1. 调用 `store.RetrieveDeclarations` 列出所有声明 ID。

## 2. 集合（Set）相关流程

* **GET /v1/sets** → `GetSetsHandler`
  1. 调用 `store.RetrieveSets` 获取全部集合名称。

* **PUT /v1/set-declarations/:id?declaration=DECL** → `PutSetDeclarationHandler`
  1. 从查询参数获取声明 ID。
  2. 调用 `store.StoreSetDeclaration` 将声明加入集合。
  3. 如有变更并允许通知，调用 `notifier.Changed`。

* **DELETE /v1/set-declarations/:id?declaration=DECL** → `DeleteSetDeclarationHandler`
  1. 调用 `store.RemoveSetDeclaration` 将声明从集合移除。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

* **GET /v1/set-declarations/:id** → `GetSetDeclarationsHandler`
  1. 调用 `store.RetrieveSetDeclarations` 获取集合内的声明列表。

* **GET /v1/declaration-sets/:id** → `GetDeclarationSetsHandler`
  1. 调用 `store.RetrieveDeclarationSets` 获取指定声明所在的集合列表。

## 3. Enrollment 与集合关系

* **GET /v1/enrollment-sets/:id** → `GetEnrollmentSetsHandler`
  1. 调用 `store.RetrieveEnrollmentSets` 获取指定 EnrollmentID 的集合列表。

* **PUT /v1/enrollment-sets/:id?set=NAME** → `PutEnrollmentSetHandler`
  1. 调用 `store.StoreEnrollmentSet` 将集合关联到 EnrollmentID。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

* **DELETE /v1/enrollment-sets/:id?set=NAME** → `DeleteEnrollmentSetHandler`
  1. 调用 `store.RemoveEnrollmentSet` 解除集合关联。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

* **DELETE /v1/enrollment-sets-all/sets/:id** → `DeleteAllEnrollmentSetsHandler`
  1. 调用 `store.RemoveAllEnrollmentSets` 移除该 EnrollmentID 的所有集合关系。
  2. 如有变更并允许通知，调用 `notifier.Changed`。

## 4. 状态查询流程

* **GET /v1/declaration-status/:id** → `GetDeclarationStatusHandler`
  1. 根据多个 EnrollmentID 调用 `store.RetrieveDeclarationStatus` 获取最后一次声明状态。

* **GET /v1/status-errors/:id** → `GetStatusErrorsHandler`
  1. 调用 `store.RetrieveStatusErrors` 获取错误列表。

* **GET /v1/status-values/:id?prefix=PATH** → `GetStatusValuesHandler`
  1. 调用 `store.RetrieveStatusValues` 获取指定前缀的值列表。

* **GET /v1/status-report/:id** → `GetStatusReportHandler`
  1. 构造 `StatusReportQuery`，调用 `store.RetrieveStatusReport` 取回完整报告并设置相关响应头。

## 5. Notifier 接口

* **POST /v1/notify** → `NotifyHandler`
  1. 直接调用 `notifier.Changed`，根据查询参数传入声明、集合或 EnrollmentID 列表。

## 6. 设备端 DDM 流程

* **GET /v1/ddm/declaration/<type>/<id>** → `ddm.DeclarationHandler`
  1. 从请求头获取 EnrollmentID，解析路径得到声明类型与 ID。
  2. 调用 `store.RetrieveEnrollmentDeclarationJSON` 获取声明 JSON 并返回。

* **GET /v1/ddm/items** → `ddm.TokensOrDeclarationItemsHandler`
  1. 调用 `store.RetrieveDeclarationItemsJSON` 返回 Declaration Items。

* **GET /v1/ddm/tokens** → `ddm.TokensOrDeclarationItemsHandler`（tokens=true）
  1. 调用 `store.RetrieveTokensJSON` 返回同步 Token。

* **POST /v1/ddm/status** → `ddm.StatusReportHandler`
  1. 读取并解析状态报告 `ddm.ParseStatus`。
  2. 调用 `store.StoreDeclarationStatus` 记录状态。

以上流程构成了 KMFDDM 的主要业务逻辑：管理员通过 API 管理声明、集合及其与 Enrollment 的关系；设备端通过 DDM 接口获取声明和 Token 并上报状态；当数据变更时 `Notifier` 会向对应设备发送 MDM 命令以触发更新。业务的核心即是围绕这些 API 请求与存储层的交互。
