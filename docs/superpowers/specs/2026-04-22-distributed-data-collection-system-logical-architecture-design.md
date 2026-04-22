# Distributed Data Collection System 总体逻辑架构方案

## 1. 文档定位

本文档是 `Distributed Data Collection System` 的总体逻辑架构方案，目标是明确系统的核心角色、职责边界、控制链路、数据链路、状态链路、可靠性设计和动态分发原则，为后续详细设计、接口设计和部署设计提供统一基础。

本文档关注以下问题：

- 系统由哪些核心组件构成
- 各组件分别负责什么
- 控制流、数据流和状态流如何流转
- `Data Gateway`、`Client Agent`、`Connector` 如何分层
- 系统如何在异常场景下尽量保证数据不丢失
- 后续动态分发如何在不破坏数据完整性的前提下演进

本文档不展开以下内容：

- 具体接口字段定义
- 数据库表结构设计
- 部署脚本与运维脚本
- 具体技术选型和中间件实现细节

## 2. 设计目标

系统设计目标如下：

- 支撑一个客户现场部署多个 `Client Agent`
- 支撑一个 `Agent` 根据配置运行多个 `Connector`
- 支撑多种 `Connector` 类型及其程序包管理
- 支撑同一 `Connector` 程序包启动多个实例
- 由中心 `Data Gateway` 统一控制所有 `Agent` 和 `Connector`
- 采集数据必须先经过 `Data Gateway` 再进入 `Data Storage`
- 在各类异常场景下尽量保证采集数据不丢失
- 为基于负载的 `Connector` 动态分发保留可演进空间

## 3. 原始需求映射

本方案针对以下原始需求展开：

1. 客户 `Agent` 是位于客户现场的独立服务器，同一个客户可能有多个客户 `Agent`
2. 一个 `Agent` 上根据配置运行多个 `Connector`，`Connector` 的作用是从数据提供服务器采集数据
3. `Connector` 有不同的类型，不同类型的 `Connector` 是不同的运行程序包，同一个程序包可以根据需要启动多个实例进行数据采集
4. 客户 `Agent` 要负责 `Connector` 的创建、停止、监控等管理工作
5. 采集的数据要进入 `Data Storage`
6. `Data Gateway` 位于公司内部，负责对所有 `Agent` 的控制、`Connector` 程序包下发、`Agent` 和 `Connector` 运行监控，以及通知 `Agent` 启动或停止 `Connector`
7. 系统需要考虑各种异常情况下的稳定性，最大可能保证采集数据不丢失
8. 系统需要考虑根据负载情况进行 `Connector` 动态分发，这是选做增强能力，且必须保证分发过程的数据完整性

## 4. 总体架构概述

系统采用“分布式采集、集中治理、统一接入、统一存储”的总体逻辑架构。

客户现场部署一个或多个 `Client Agent`。每个 `Agent` 根据中心下发的目标配置，在本机托管运行多个 `Connector` 实例。`Connector` 负责连接外部数据提供服务器并执行实际采集。采集结果先进入 `Agent` 本地持久化队列，再上传到公司内部的 `Data Gateway`。`Data Gateway` 负责统一控制、统一数据接入、统一处理和统一入库，最终由 `Data Storage` 落地并供 `Business System` 查询和消费。

该架构具有以下特征：

- 分布式采集
- 集中治理
- 插件式源端接入
- 统一数据入口
- 横向扩展
- 源系统解耦

## 5. 核心组件

- `Business System`
- `Data Gateway`
- `Data Storage`
- `Client Agent`
- `Connector`
- `External Data Provider Server`

## 6. 核心概念定义

### 6.1 Connector Type

`Connector Type` 表示一种采集能力类型，例如数据库采集器、文件采集器、接口采集器、消息采集器等。它是逻辑类型定义。

### 6.2 Connector Package

`Connector Package` 表示某个 `Connector Type` 对应的可部署运行程序包。不同类型通常对应不同程序包。

### 6.3 Connector Instance

`Connector Instance` 表示某个 `Agent` 上按具体配置启动的一个实际运行实例。同一个 `Connector Package` 可以根据业务需要在同一 `Agent` 或不同 `Agent` 上启动多个实例。

### 6.4 Task

`Task` 表示业务上定义的一条采集任务，用于描述采什么、怎么采、由谁执行以及使用哪个 `Connector` 类型。

### 6.5 Checkpoint

`Checkpoint` 表示 `Connector` 针对源端采集进度维护的位点语义。它不限定必须是数字 offset，也可以是时间戳、cursor、主键值、文件位置、日志偏移等。

### 6.6 Message

`Message` 表示一条统一采集消息，是 `Connector` 采集并由 `Agent` 可靠传输到 `Data Gateway` 的标准数据载体。

## 7. 组件职责

### 7.1 Business System

`Business System` 负责：

- 配置采集任务与采集策略
- 查看任务、`Agent`、`Connector` 运行状态
- 查询和消费已入库数据

`Business System` 不负责具体执行调度、程序包管理和现场运行恢复。

### 7.2 Data Gateway

`Data Gateway` 负责：

- 对所有 `Agent` 的统一控制
- 采集任务管理
- `Connector` 程序包下发与版本管理
- 配置管理与命令下发
- `Agent` 和 `Connector` 的运行监控与状态汇聚
- 接收 `Agent` 上传的数据并执行校验、标准化、路由和入库

`Data Gateway` 是系统的中心控制面和中心数据接入面。

### 7.3 Data Storage

`Data Storage` 负责：

- 存储原始数据、标准化数据和历史记录
- 为业务查询和后续消费提供支撑

### 7.4 Client Agent

`Client Agent` 是客户现场的独立运行节点，负责：

- 作为本机 `Connector` 的宿主与管理控制器
- 根据配置托管多个 `Connector Instance`
- 负责 `Connector` 的创建、启动、停止、重启、销毁和监控
- 负责本地持久化缓冲和数据上传
- 负责本地运行状态上报

### 7.5 Connector

`Connector` 负责：

- 连接外部数据提供服务器
- 执行源端认证、协议处理和采集逻辑
- 维护源端采集位点语义
- 将采集结果封装为统一采集消息并交给 `Agent`

`Connector` 不负责全局调度、跨网络可靠传输、统一标准化和入库。

### 7.6 External Data Provider Server

`External Data Provider Server` 是外部原始数据源，由 `Connector` 访问和采集。

## 8. 组件关系与职责边界

系统内核心关系如下：

- `Data Gateway` 管 `Client Agent`
- `Client Agent` 管 `Connector`
- `Connector` 负责“采”
- `Client Agent` 负责“管和送”
- `Data Gateway` 负责“控、收、处理、入库”
- `Data Storage` 负责“存”
- `Business System` 负责“配、看、用”

这一定义直接支撑原始需求中的多 `Agent`、多 `Connector`、程序包分发、实例管理和统一监控要求。

## 9. 三条主链路

### 9.1 控制链路

`Business System -> Data Gateway -> Client Agent -> Connector`

该链路用于任务编排、程序包下发、配置更新、实例启停和运行控制。

### 9.2 数据链路

`External Data Provider Server -> Connector -> Client Agent -> Data Gateway -> Data Storage -> Business System`

该链路用于采集数据的可靠传输、统一处理和最终落库。

### 9.3 状态链路

`Connector -> Client Agent -> Data Gateway -> Business System`

该链路用于心跳、状态、指标和告警信息上报。

## 10. Data Gateway 逻辑架构

`Data Gateway` 逻辑上划分为以下模块：

### 10.1 Task Control

负责采集任务定义、任务状态管理、目标 `Agent` 选择、任务启停和控制编排。

### 10.2 Agent Management

负责 `Agent` 注册、心跳维护、能力信息、负载信息、实例映射和运行状态汇聚。

### 10.3 Package & Config Management

负责 `Connector Package`、版本、配置模板、配置下发和版本切换。

### 10.4 Command Dispatch

负责向 `Agent` 下发控制指令，例如创建、启动、停止、重启、更新配置、升级程序包。

### 10.5 Data Ingestion

负责接收 `Agent` 上传的数据，完成鉴权、基础校验、接入缓冲写入和接收确认。

### 10.6 Data Processing

负责标准化、格式转换、幂等去重、聚合、路由和最终入库。

### 10.7 Monitoring & Alerting

负责统一监控任务、`Agent`、`Connector`、接入链路和处理链路的运行状态，并输出告警。

## 11. Client Agent 逻辑架构

`Client Agent` 逻辑上划分为以下模块：

### 11.1 Command Handler

接收 `Data Gateway` 下发的目标配置和控制指令，并触发本地执行动作。

### 11.2 Connector Manager

根据配置维护本机 `Connector Instance` 集合，负责实例创建、启动、停止、重启和销毁。

### 11.3 Local Persistent Queue

作为本地持久化消息队列，负责采集结果落盘、积压、重试、断点续传和上传确认。

### 11.4 Runtime Monitor

负责监控 `Connector` 进程状态、采集吞吐、异常次数、积压量和延迟。

### 11.5 Local State Store

负责保存本地任务状态、实例状态、命令执行记录、`Checkpoint` 和未完成消息索引。

`Client Agent` 的本质不是简单的命令转发器，而是本地“配置驱动的 `Connector` 宿主与管理控制器”。

## 12. Connector 的逻辑边界

`Connector` 是特定源类型的采集执行插件，其职责边界如下：

- 负责源端连接、认证和协议处理
- 负责具体采集逻辑执行
- 负责维护源端 `Checkpoint` 语义
- 负责将源端数据转换为统一采集消息并交给 `Agent`

`Connector` 不承担以下职责：

- 不负责决定在哪个 `Agent` 上运行
- 不负责全局任务调度
- 不负责跨网络可靠传输
- 不负责统一标准化、去重和入库
- 不直接与 `Data Storage` 交互

## 13. Agent 与 Connector 的关系模型

`Client Agent` 与 `Connector` 不是平级关系，而是“宿主-执行实例”关系。

系统以“目标配置驱动 + Agent 本地收敛执行”为主。`Data Gateway` 下发给 `Agent` 的核心不是一次性启动命令，而是该 `Agent` 的目标运行配置。`Agent` 根据配置决定本机应运行哪些 `Connector`、运行几个实例、使用哪个程序包版本以及采用什么采集参数，并将目标状态收敛为实际运行状态。

该模型天然支撑以下原始需求：

- 一个 `Agent` 上运行多个 `Connector`
- 不同 `Connector Type` 对应不同 `Connector Package`
- 同一个 `Connector Package` 可按配置启动多个实例
- `Agent` 负责 `Connector` 的创建、停止和监控

## 14. 数据流转模型

采集数据的最小逻辑流转如下：

`Source -> Connector -> Agent Local Persistent Queue -> Gateway Ingestion Queue -> Gateway Processing -> Data Storage`

建议定义以下数据状态：

- `Collected`
- `AgentBuffered`
- `GatewayAccepted`
- `GatewayBuffered`
- `Processed`
- `Stored`

状态含义如下：

- `Collected`：`Connector` 已从源端成功读取数据
- `AgentBuffered`：数据已被 `Agent` 本地持久化
- `GatewayAccepted`：`Data Gateway` 已可靠接收并返回确认
- `GatewayBuffered`：数据已进入 `Gateway` 接入持久化缓冲
- `Processed`：数据已完成标准化、校验、去重和路由
- `Stored`：数据已成功写入 `Data Storage`

## 15. 任务与实例状态模型

### 15.1 任务状态

任务状态由 `Data Gateway` 维护，建议使用：

- `Draft`
- `Scheduled`
- `Deploying`
- `Running`
- `Paused`
- `Failed`
- `Stopped`

### 15.2 Connector 实例状态

实例状态由 `Client Agent` 维护，建议使用：

- `Created`
- `Starting`
- `Running`
- `Degraded`
- `Retrying`
- `Stopped`
- `Failed`
- `Destroyed`

该状态模型用于清晰区分“业务层任务状态”和“现场执行实例状态”。

## 16. 可靠性设计

本系统第一版不追求复杂的 exactly-once，而采用“至少一次传递 + 幂等去重”的可靠性策略。

### 16.1 Agent 侧可靠性

- `Agent` 侧采用本地持久化消息队列，不依赖纯内存缓存
- 采集结果只有在写入 `Local Persistent Queue` 后才算被现场可靠保存
- `Agent` 重启后应能够恢复未完成消息并继续上传

### 16.2 Connector 侧可靠性

- `Connector` 侧采用 `Checkpoint` 机制维护源端采集进度
- `Checkpoint` 语义归 `Connector`
- `Checkpoint` 持久化由 `Agent` 的 `Local State Store` 负责
- `Checkpoint` 推进边界定义为“`Agent` 本地持久化消息成功”

### 16.3 Gateway 侧可靠性

- `Gateway` 只有在将上传数据可靠写入自身接入持久化缓冲后，才返回 `Accepted`
- `Gateway` 接入成功后，由内部处理链继续执行标准化、路由和入库
- 已确认的数据后续处理失败由 `Gateway` 内部重试，不要求 `Agent` 重发

### 16.4 幂等与去重

- 每条采集消息必须有唯一 `messageId`
- `Gateway` 处理层执行幂等判断
- `Storage` 写入侧应支持唯一键或等效幂等写入策略

该设计目标是优先保证不丢数，允许故障恢复过程中出现少量重复，并通过幂等去重避免重复数据最终生效。

## 17. Data Gateway 内部处理链路

`Data Gateway` 的数据接入与处理链路定义如下：

`Agent Upload -> Gateway Access -> Auth/Validation -> Ingress Persistent Queue -> Ack to Agent -> Normalize/Transform -> Dedup/Idempotency -> Route -> Storage Writer -> Data Storage`

该链路中最关键的责任边界如下：

- `Agent` 的责任终点：`Gateway Accepted`
- `Gateway` 的责任起点：`Ingress Persistent Queue Write Success`

该边界避免了“`Agent` 认为已送达而 `Gateway` 实际未可靠接收”的问题。

## 18. 异常与恢复原则

- `Connector` 短时错误由实例内部执行有限重试
- 实例多次失败后由 `Agent` 执行重启
- `Agent` 重启后依赖本地状态和持久化队列恢复运行
- `Gateway` 处理失败由网关内部重试，不要求 `Agent` 重发已确认数据
- 长时间失败、重复异常或恢复失败后，由 `Gateway` 统一告警并标记任务异常

该恢复模型遵循“现场先保住数据，中心再完成处理”的原则。

## 19. 动态分发设计

动态分发是选做增强能力，不作为第一版核心必做能力，但架构上应预留。

### 19.1 设计原则

- 同一采集任务或同一采集分区在同一时刻只能有一个有效 owner
- 动态分发采用“单主归属 + Checkpoint 交接”模型
- 迁移过程中优先保证数据完整性，允许少量重复但不允许无界丢失

### 19.2 基本迁移流程

1. `Data Gateway` 根据负载或策略决定迁移任务
2. `Gateway` 将任务状态标记为 `Rebalancing`
3. 原 `Agent` 停止继续拉取新数据
4. 原 `Agent` 刷完本地队列或完成安全封存
5. 原 `Agent` 上报最后可靠 `Checkpoint`
6. `Gateway` 将该 `Checkpoint` 下发给目标 `Agent`
7. 目标 `Agent` 从交接 `Checkpoint` 启动新的 `Connector Instance`
8. 迁移完成后由 `Gateway` 切换 owner 并销毁旧实例

### 19.3 第一版实现边界

第一版建议只支持任务级迁移和单实例迁移，不建议直接实现：

- 双活抢占式迁移
- 同一采集流多 `Agent` 并行消费
- 无停机热迁移

## 20. 非功能性要求

### 20.1 可扩展性

支持一个客户部署多个 `Agent`，支持多种 `Connector Type`、多程序包版本和多实例横向扩展。

### 20.2 可用性

`Data Gateway` 和 `Data Storage` 应按逻辑中心、物理集群方式建设，避免单点故障。

### 20.3 可观测性

应监控以下关键指标：

- `Agent` 心跳
- `Connector Instance` 状态
- 本地队列积压
- 采集吞吐
- 传输延迟
- 失败率
- `Checkpoint` 推进进度

### 20.4 可运维性

系统应支持：

- 程序包版本管理
- 配置下发
- 灰度升级
- 命令回执
- 统一告警

## 21. 第一版实现边界

第一版建议实现以下能力：

- 多 `Agent` 统一纳管
- `Agent` 托管多 `Connector Instance`
- `Connector Type`、`Package`、`Instance` 三层模型
- `Agent` 本地持久化消息队列
- `Connector` `Checkpoint` 机制
- `Gateway` 接入持久化与幂等去重
- 任务状态与实例状态管理
- `Gateway` 程序包下发和运行监控

第一版不建议实现以下能力：

- 复杂双活分发
- 同一采集流多 `Agent` 并行消费
- 无停机热迁移
- 过于复杂的本地编排系统

## 22. 结论

本方案可以完整支撑原始需求，且职责边界清晰：

- `Business System` 负责配、看、用
- `Data Gateway` 负责控、收、处理、入库
- `Client Agent` 负责宿主、管理、缓冲、上传
- `Connector` 负责源端采集与 `Checkpoint` 语义
- `Data Storage` 负责最终存储与查询支撑

该方案是一版完整的总体逻辑架构方案，已经具备进入下一阶段详细设计的基础。后续若继续深化，应补充接口模型、消息模型、部署拓扑和关键时序图。
