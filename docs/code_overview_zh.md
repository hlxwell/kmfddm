# 项目代码结构概览

本文档简要介绍 KMFDDM 项目的主要目录及其功能，供快速了解整个代码库的组成。

## 总体说明

KMFDDM 是一个用于实现 Apple Declarative Device Management (DDM) 的服务器端项目，提供与开源 MDM 服务器（如 NanoMDM、MicroMDM）配合的管理能力。项目使用 Go 语言编写，提供 HTTP API、DDM 协议接口及多种存储后端。

## 目录结构

- **cmd/kmfddm**：主程序入口，`main.go` 负责解析命令行参数、初始化存储后端和通知模块并启动 HTTP 服务。
- **ddm**：DDM 协议相关的数据结构与工具，例如声明（Declaration）、声明项（Declaration Items）、同步 token 生成等。
- **ddm/build**：用于构建声明项和 token 的辅助组件，实现增量构建和哈希计算。
- **http**：HTTP 相关代码。
  - `ddm` 子目录提供 DDM 协议处理，包括获取声明、声明项、token 以及接收状态报告等接口。
  - `api` 子目录实现面向管理员的 REST API，可对声明、集合（Set）和设备与集合的关联关系进行管理，同时可查询状态信息。
- **notifier**：变更通知模块，负责在声明或集合变化时向 MDM 服务器发送 `DeclarativeManagement` 命令。`foss` 子目录提供与 NanoMDM/MicroMDM 兼容的实现。
- **storage**：存储层接口及实现。
  - 提供文件系统、内存、diskv、MySQL 等多种后端，统一封装为接口以便于切换。
  - `shard` 子目录提供动态分片声明的实现，可根据设备 ID 生成管理属性。
- **jsonpath**：一个基于 `fastjson` 的 JSON 路径处理工具，用于解析状态报告等结构化数据。
- **logkeys**：定义项目统一的日志键，便于结构化日志输出。
- **tools**：辅助脚本，主要为 shell/Python 脚本，用于调用 API 批量管理声明和集合。
- **docs**：项目文档，包括快速开始、操作指南以及 OpenAPI 定义等。
- **test/e2e**：端到端测试，验证 API、DDM 协议及通知流程的完整性。

## 功能概览

1. **声明和集合管理**：通过 API 创建、更新或删除声明，并将其加入集合，再将集合关联到具体的 Enrollment ID。
2. **自动生成 token**：声明及声明集合更新后，会自动重新计算 `ServerToken` 和 `DeclarationsToken`，确保设备能够感知到变化。
3. **通知机制**：当声明或集合发生变更时，`notifier` 模块负责向相关设备发送 MDM 指令，触发它们重新获取最新声明。
4. **状态报告处理**：设备上报的声明状态和错误会被存储，可通过 API 查询，便于排查问题。
5. **多种存储后端**：可根据需要选择文件、磁盘 KV、内存或 MySQL 等不同的存储实现，也可以通过 `shard` 机制动态生成管理属性。

以上即为 KMFDDM 项目的代码结构与主要功能简介。
