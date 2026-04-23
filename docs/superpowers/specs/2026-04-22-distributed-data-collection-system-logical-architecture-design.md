# Distributed Data Collection System 总体逻辑架构方案

## 文档信息

- 文档名称：`Distributed Data Collection System 总体逻辑架构方案`
- 文档版本：`v0.3`
- 文档状态：`Reviewed Draft`
- 创建日期：`2026-04-22`
- 最后更新：`2026-04-23`

## 修订历史

| 版本 | 日期 | 修订摘要 |
| --- | --- | --- |
| v0.1 | 2026-04-22 | 建立逻辑架构骨架，明确核心组件、基本链路和职责边界 |
| v0.2 | 2026-04-22 | 补充 Gateway / Agent / Connector 分层、可靠性设计和动态分发初步方案 |
| v0.3 | 2026-04-23 | 根据审阅意见补齐安全边界、Control Plane / Data Plane、网络分区、背压、消息标识、配置版本、程序包分发、日志审计、时间语义与需求追溯 |

## 1. 文档定位

本文档是 `Distributed Data Collection System` 的总体逻辑架构方案，目标是明确系统的核心角色、职责边界、控制链路、数据链路、状态链路、可靠性设计和动态分发原则，为后续详细设计、接口设计和部署设计提供统一基础。

本文档主要回答以下问题：

- 系统由哪些核心组件构成
- 各组件分别负责什么
- 控制流、数据流和状态流如何流转
- `Data Gateway`、`Client Agent`、`Connector` 如何分层
- 系统如何在异常场景下尽量保证数据不丢失
- 动态分发如何在不破坏数据完整性的前提下演进

本文档不展开以下内容：

- 具体接口字段定义
- 数据库表结构设计
- 部署脚本与运维脚本
- 具体技术选型和中间件实现细节
- 明确性能容量数值

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

## 3. 原始需求与追溯关系

### 3.1 原始需求

- `R1` 客户 `Agent` 是位于客户现场的独立服务器，同一个客户可能有多个客户 `Agent`
- `R2` 一个 `Agent` 上根据配置运行多个 `Connector`，`Connector` 的作用是从数据提供服务器采集数据
- `R3` `Connector` 有不同类型，不同类型对应不同运行程序包，同一个程序包可以按需启动多个实例
- `R4` 客户 `Agent` 要负责 `Connector` 的创建、停止、监控等管理工作
- `R5` 采集的数据要进入 `Data Storage`
- `R6` `Data Gateway` 位于公司内部，负责对所有 `Agent` 的控制、程序包下发、运行监控以及通知 `Agent` 启停 `Connector`
- `R7` 系统需要考虑各种异常情况下的稳定性，最大可能保证采集数据不丢失
- `R8` 系统需要考虑根据负载情况进行 `Connector` 动态分发，且分发过程必须保证数据完整性

### 3.2 需求追溯矩阵

| 需求 | 摘要 | 对应章节 | 设计落实说明 |
| --- | --- | --- | --- |
| R1 | 多 Agent 现场部署 | 4、7、8、10、24 | 明确单客户多 Agent 形态、Agent 注册、Agent 管理和横向扩展 |
| R2 | Agent 按配置运行多个 Connector | 6、8、11、12、14 | 明确目标配置驱动、Agent 宿主管理、Connector 托管运行 |
| R3 | 类型 / 程序包 / 多实例 | 6、11、18 | 建立 `Connector Type / Package / Instance` 三层模型和程序包分发策略 |
| R4 | Agent 负责 Connector 生命周期管理 | 8、11、15 | 明确 Agent 作为 supervisor，负责创建、停止、监控、资源控制与受控重启 |
| R5 | 数据入库 | 9、13、17 | 明确数据链路与 Gateway Data Plane 到 Storage 的处理链路 |
| R6 | Gateway 统一控制和监控 | 7、10、16、18、23 | 明确 Control Plane / Data Plane、程序包管理、命令分发、监控与审计 |
| R7 | 稳定性与尽量不丢数 | 13、17、19、20、21 | 明确本地持久化队列、checkpoint 原子提交、幂等去重、背压和失败隔离 |
| R8 | 动态分发 | 22 | 明确单主归属、checkpoint 交接和保守迁移策略 |

## 4. 部署边界与总体架构概述

### 4.1 单客户单实例部署边界

本系统不是多租户平台型系统。部署边界定义如下：

- 一套系统实例只服务一个客户
- 客户之间采用物理隔离部署
- 一个客户对应一套独立部署的 `Business System + Data Gateway + Data Storage`
- 该客户下可部署多个 `Client Agent`

因此，本方案不引入平台型 `Tenant` 模型，而是采用“单客户单实例”边界。客户间数据、配置、凭证、监控和审计天然隔离于不同部署实例。

### 4.2 总体架构概述

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

### 6.7 Control Plane

`Control Plane` 表示系统中面向任务、配置、程序包、Agent 管理、命令下发、监控和审计的控制面。

### 6.8 Data Plane

`Data Plane` 表示系统中面向采集消息接入、缓冲、处理、去重、路由和入库的数据面。

### 6.9 Desired State Version

`desiredStateVersion` 表示某个任务或实例目标配置的版本号。Agent 始终以最新版本为准进行状态收敛。

### 6.10 Trigger Mode

`triggerMode` 表示任务的触发模式。第一版重点支持：

- `Manual`
- `Scheduled`
- `Continuous`

### 6.11 Offline Policy

`offlinePolicy` 表示 Agent 与 Gateway 断链后的任务行为策略。第一版支持：

- `ContinueWhenDisconnected`
- `PauseWhenDisconnected`
- `ContinueUntilWatermark`

### 6.12 Backpressure Policy

`backpressurePolicy` 表示任务在背压场景下的处理策略。第一版支持：

- `BestEffort`
- `PreferContinuity`
- `StrictPauseOnPressure`

### 6.13 Collection Partition

`Collection Partition` 表示一个采集任务内部可以独立分配和迁移的最小采集单元。

- 如果源系统天然支持分区，例如消息队列 partition、数据库分片、文件目录分片，则每个分区可作为独立 owner 单元
- 如果源系统不支持天然分区，则整个任务视为单一分区

## 7. Data Gateway 逻辑架构

`Data Gateway` 在逻辑架构上明确拆分为 `Gateway Control Plane` 和 `Gateway Data Plane`。第一版部署可同机或同集群实现，但逻辑边界必须保持清晰。

### 7.1 Gateway Control Plane

负责：

- 任务定义与任务状态管理
- `desiredStateVersion` 维护
- Agent 注册、心跳与状态管理
- 程序包元数据与版本管理
- 配置管理
- 命令生成与分发
- 监控汇总
- 操作审计

### 7.2 Gateway Data Plane

负责：

- Agent 上传数据接入
- 接入鉴权与基础校验
- 接入持久化缓冲
- 标准化与格式转换
- 幂等去重
- 路由与入库
- 失败隔离处理
- 数据链路背压反馈

### 7.3 多实例与单点规避原则

- `Control Plane` 和 `Data Plane` 均应支持多实例部署
- `Control Plane` 多实例共享统一元数据存储
- `Data Plane` 多实例通过负载均衡承接上传流量
- 控制命令必须具备全局唯一 `commandId` 和幂等执行能力
- 同一任务或同一 Agent 的目标状态变更必须串行化提交，避免冲突命令

## 8. Client Agent 逻辑架构

`Client Agent` 逻辑上划分为以下模块：

### 8.1 Command Handler

接收 `Data Gateway Control Plane` 下发的目标配置和控制指令，并触发本地执行动作。

### 8.2 Connector Manager

根据配置维护本机 `Connector Instance` 集合，负责实例创建、启动、停止、重启和销毁。

### 8.3 Local Persistent Queue

作为本地持久化消息队列，负责采集结果落盘、积压、重试、断点续传和上传确认。

### 8.4 Runtime Monitor

负责监控 `Connector` 进程状态、采集吞吐、异常次数、积压量和延迟。

### 8.5 Local State Store

负责保存本地任务状态、实例状态、命令执行记录、`Checkpoint` 和未完成消息索引。

`Client Agent` 的本质不是简单的命令转发器，而是本地“配置驱动的 `Connector` 宿主与管理控制器”。

## 9. 组件职责

### 9.1 Business System

`Business System` 负责：

- 配置采集任务与采集策略
- 查看任务、`Agent`、`Connector` 运行状态
- 查询和消费已入库数据

`Business System` 不负责具体执行调度、程序包管理和现场运行恢复。

### 9.2 Data Gateway

`Data Gateway` 负责：

- 对所有 `Agent` 的统一控制
- 采集任务管理
- `Connector` 程序包下发与版本管理
- 配置管理与命令下发
- `Agent` 和 `Connector` 的运行监控与状态汇聚
- 接收 `Agent` 上传的数据并执行校验、标准化、路由和入库

### 9.3 Data Storage

`Data Storage` 负责：

- 存储原始数据、标准化数据和历史记录
- 为业务查询和后续消费提供支撑

### 9.4 Client Agent

`Client Agent` 是客户现场的独立运行节点，负责：

- 作为本机 `Connector` 的宿主与管理控制器
- 根据配置托管多个 `Connector Instance`
- 负责 `Connector` 的创建、启动、停止、重启、销毁和监控
- 负责本地持久化缓冲和数据上传
- 负责本地运行状态上报

### 9.5 Connector

`Connector` 负责：

- 连接外部数据提供服务器
- 执行源端认证、协议处理和采集逻辑
- 维护源端采集位点语义
- 将采集结果封装为统一采集消息并交给 `Agent`

`Connector` 不负责全局调度、跨网络可靠传输、统一标准化和入库。

### 9.6 External Data Provider Server

`External Data Provider Server` 是外部原始数据源，由 `Connector` 访问和采集。

## 10. 安全设计

系统部署在单客户单实例、专线 / VPN / 内网互通前提下，但仍需具备基础安全能力。

### 10.1 Agent 与 Gateway 通信安全

- `Agent <-> Gateway` 通信采用 `TLS`
- 节点身份认证优先采用 `mTLS`
- 每个 Agent 拥有独立节点身份，不共享统一节点凭证
- Gateway 仅接受已登记 Agent 的请求

### 10.2 Connector 访问源端的凭证管理

- 凭证不得写死在 `Connector Package` 中
- 凭证由 Gateway 配置管理，并由 Agent 本地安全存储后按实例注入
- 凭证与任务或实例绑定
- 第一版要求支持人工轮换，不要求自动轮换

### 10.3 权限边界

- 业务用户权限与 Agent 节点权限分离
- 人工操作通过 `Business System / Gateway Control Plane` 权限体系执行
- 节点操作通过 Agent 身份体系执行

### 10.4 审计要求

- 任务、配置、程序包、命令、迁移、Agent 注册与失效等关键操作必须记录审计日志
- 运行日志中不得明文记录敏感凭证

## 11. 组件关系与职责边界

系统内核心关系如下：

- `Data Gateway` 管 `Client Agent`
- `Client Agent` 管 `Connector`
- `Connector` 负责“采”
- `Client Agent` 负责“管和送”
- `Data Gateway` 负责“控、收、处理、入库”
- `Data Storage` 负责“存”
- `Business System` 负责“配、看、用”

这一定义直接支撑原始需求中的多 `Agent`、多 `Connector`、程序包分发、实例管理和统一监控要求。

## 12. Agent 与 Connector 的关系模型

`Client Agent` 与 `Connector` 不是平级关系，而是“宿主-执行实例”关系。

系统以“目标配置驱动 + Agent 本地收敛执行”为主。`Data Gateway` 下发给 `Agent` 的核心不是一次性启动命令，而是该 `Agent` 的目标运行配置。`Agent` 根据配置决定本机应运行哪些 `Connector`、运行几个实例、使用哪个程序包版本以及采用什么采集参数，并将目标状态收敛为实际运行状态。

该模型天然支撑以下原始需求：

- 一个 `Agent` 上运行多个 `Connector`
- 不同 `Connector Type` 对应不同 `Connector Package`
- 同一个 `Connector Package` 可按配置启动多个实例
- `Agent` 负责 `Connector` 的创建、停止和监控

## 13. 三条主链路

### 13.1 控制链路

`Business System -> Gateway Control Plane -> Client Agent -> Connector`

该链路用于任务编排、程序包版本变更、配置更新、实例启停和运行控制。

### 13.2 数据链路

`External Data Provider Server -> Connector -> Client Agent -> Gateway Data Plane -> Data Storage -> Business System`

该链路用于采集数据的可靠传输、统一处理和最终落库。

### 13.3 状态链路

`Connector -> Client Agent -> Gateway Control Plane -> Business System`

该链路用于心跳、状态、指标和告警信息上报。

## 14. 控制面通信与交互模式

系统采用“控制拉、数据推”的通信模型。

### 14.1 控制面通信模式

控制面以 `Agent` 主动拉取为主，承载以下交互：

- `Agent Register`
- `Heartbeat / Status Report`
- `Desired State Sync / Command Poll`
- `Command Result Report`

控制面语义如下：

- `desiredState` 是事实来源
- `command` 是促使 Agent 尽快收敛的执行信号
- Agent 恢复或重连时，以最新 `desiredStateVersion` 为准，不补执行过期命令

### 14.2 数据面通信模式

数据面以 `Agent` 主动推送为主，承载以下交互：

- 批量上传采集消息
- 接收确认
- 重试
- 背压反馈

第一版要求支持：

- 批量上传
- 压缩传输
- 幂等重试

### 14.3 Agent 注册流程

Agent 首次启动后应执行以下流程：

1. 携带节点身份发起注册
2. Gateway 校验节点身份和所属部署
3. Agent 上报基础信息，例如 `agentId`、版本、能力和主机标识
4. Gateway 建立 Agent 元数据
5. Agent 进入心跳与目标状态同步流程

若 Agent 替换或重装，应重新注册并使旧注册项失效。

## 15. Connector 运行模型与隔离策略

### 15.1 独立进程运行

第一版中，每个 `Connector Instance` 默认以独立进程方式运行，由 `Agent` 统一托管，不采用线程级或协程级混跑。

### 15.2 Agent 的托管职责

Agent 对每个 `Connector Instance` 负责：

- 启动
- 停止
- 重启
- 销毁
- 健康检查
- 资源监控
- 运行目录管理
- 配置注入
- 日志入口管理

### 15.3 资源限制

每个实例应具备独立资源配额，至少包括：

- CPU 配额
- 内存上限
- 磁盘临时目录配额
- 文件句柄或连接数上限
- 并发连接数上限

### 15.4 配置变更生效方式

第一版默认采用“实例级受控重启 / 受控切换”使配置变更生效，不强制要求热更新。

## 16. 配置驱动、版本控制与任务触发

### 16.1 Desired State Version

系统采用 `desiredStateVersion` 作为目标配置的版本控制机制：

- Gateway 对同一 Task 维护单调递增的 `desiredStateVersion`
- Agent 永远以最新目标版本为准进行收敛
- 旧版本命令不再作为执行事实来源

### 16.2 配置冲突处理

- 同一 Task 的目标状态变更必须串行化提交
- Agent 若正在执行旧版本收敛流程，收到新版本后应以最新版本为准
- 旧版本或过期命令应被忽略

### 16.3 Trigger Mode

每个 Task 必须声明 `triggerMode`。第一版重点支持：

- `Manual`
- `Scheduled`
- `Continuous`

其中：

- `Manual` 由人工触发执行
- `Scheduled` 由 Gateway Control Plane 按调度规则触发
- `Continuous` 任务持续保持运行态

### 16.4 Scheduled 任务的时间源

`Scheduled` 任务的触发时间以 `Gateway Control Plane` 时钟为准，Agent 本地时间不作为调度真源。

## 17. 数据模型与消息标识

### 17.1 Message 最小结构

统一采集消息至少包含以下字段：

- `transportMessageId`
- `recordDedupKey`
- `taskId`
- `agentId`
- `connectorType`
- `connectorInstanceId`
- `sourceId`
- `checkpoint`
- `sourceEventTime`
- `collectTime`
- `payloadFormat`
- `payload`
- `metadata`

### 17.2 分层标识模型

系统采用分层标识模型，而不是单一消息 ID 模型。

#### transportMessageId

用于传输层幂等、接收确认和重试控制。其规则如下：

- 由 Agent 在消息写入本地持久化队列时生成
- 同一条本地持久化消息在重试上传时保持不变
- 用于 Agent 本地队列主键和 Gateway Ingress 幂等接收

#### recordDedupKey

用于业务层去重和最终入库幂等。其规则如下：

- 去重语义由 `Connector Type` 定义
- 若源端有稳定主键，优先使用源端主键
- 若无天然主键，则用 `sourceId + checkpoint + subRecordKey` 或关键字段摘要组合生成

### 17.3 去重层次

- `Gateway Ingress` 基于 `transportMessageId` 去重
- `Gateway Processing / Storage` 基于 `recordDedupKey` 去重

## 18. 程序包分发、版本兼容与回滚

### 18.1 分发模式

- `Gateway Control Plane` 负责管理 `Connector Package` 元数据、版本与发布策略
- `Agent` 按目标配置主动拉取所需程序包

### 18.2 程序包元数据

每个程序包至少应具备：

- `connectorType`
- `packageVersion`
- `packageUri`
- `checksum`
- `packageSize`
- `configSchemaVersion`
- `runtimeCompatibility`

### 18.3 下载与校验

- Agent 下载完成后必须先校验完整性，再允许安装和实例启动
- 未校验通过的程序包不得投入运行
- 大包下载应支持可恢复重试，第一版预留断点续传能力

### 18.4 多版本共存

同一 `connectorType` 的多个 `packageVersion` 允许在 Agent 本地短期共存，以支持平滑升级和回滚。

### 18.5 升级与回滚

第一版默认采用“新实例受控切换旧实例”的升级方式：

1. Agent 下载并校验新版本包
2. 在一致性边界上启动新实例或准备新实例
3. 切换到新实例继续运行
4. 停止旧实例
5. 旧版本在无引用后清理

回滚通过切换回上一稳定版本目标配置实现。

### 18.6 配置兼容性

- 每个程序包必须声明 `configSchemaVersion`
- Gateway 下发配置时必须校验程序包与配置版本兼容性
- 若不兼容，不允许直接运行，需执行显式迁移或阻断升级

## 19. 可靠性设计

本系统第一版不追求复杂的 exactly-once，而采用“至少一次传递 + 幂等去重”的可靠性策略。

### 19.1 Agent 侧可靠性

- Agent 采用本地持久化消息队列，不依赖纯内存缓存
- 采集结果只有在写入 `Local Persistent Queue` 后才算被现场可靠保存
- Agent 重启后能够恢复未完成消息并继续上传

### 19.2 Connector 侧可靠性

- Connector 采用 `Checkpoint` 机制维护源端采集进度
- `Checkpoint` 语义归 Connector，持久化由 Agent 负责
- Connector 只产生候选 checkpoint，不直接持久化最终提交进度

### 19.3 Agent 本地原子提交

消息入队和 `Checkpoint` 更新必须属于同一 Agent 本地原子提交单元：

- 要么两者都成功
- 要么两者都不生效

这意味着：

- 允许因恢复导致的少量重复
- 不允许 `Checkpoint` 先行推进导致数据丢失

### 19.4 Gateway 接收确认边界

- Gateway 只有在将上传数据可靠写入自身接入持久化缓冲后，才返回 `Accepted`
- Agent 收到 `Accepted` 后，后续处理责任全部由 `Gateway Data Plane` 接管

## 20. 网络分区、离线策略与背压

### 20.1 Agent 网络状态

建议定义以下网络相关状态：

- `Online`
- `Degraded`
- `Disconnected`
- `Recovering`

### 20.2 Offline Policy

系统支持按任务类型配置离线策略：

- `ContinueWhenDisconnected`
- `PauseWhenDisconnected`
- `ContinueUntilWatermark`

默认建议采用 `ContinueUntilWatermark`。

### 20.3 本地队列水位

Agent 本地持久化队列必须具备：

- `Low Watermark`
- `High Watermark`
- `Critical Watermark`

对应行为：

- 低水位以下：正常运行
- 高水位以上：进入背压模式、降速或暂停可暂停任务
- 临界水位以上：进入强制保护，防止本地磁盘耗尽

### 20.4 分级背压

系统采用分级背压机制：

`Storage Slowdown -> Gateway Data Processing Backlog -> Gateway Ingress Pressure -> Agent Upload Slowdown -> Agent Local Queue Growth -> Connector Throttle / Pause`

其规则如下：

- Gateway Data Plane 先通过内部缓冲和处理队列吸收波动
- 超过阈值后，对 Agent 上传施加流控反馈
- Agent 根据流控反馈和本地水位执行降速、暂停和恢复
- Connector 不直接处理远端背压，由 Agent 统一执行本地流控

### 20.5 长期离线

- Gateway 将长期未恢复的 Agent 标记为 `Unavailable`
- 是否迁移任务取决于任务特性、checkpoint 完整性和业务风险
- 长期离线不等于自动迁移

## 21. Data Gateway 内部处理链路与失败语义

### 21.1 内部处理链路

`Agent Upload -> Gateway Access -> Auth/Validation -> Ingress Persistent Queue -> Ack to Agent -> Normalize/Transform -> Dedup/Idempotency -> Route -> Storage Writer -> Data Storage`

### 21.2 失败分类

Gateway Data Plane 的处理链失败分为三类：

- `Transient Failure`
- `Data Quality Failure`
- `Routing / Storage Failure`

### 21.3 失败处理原则

#### 瞬时失败

- 执行有限重试与退避
- 不立即判死
- 超过阈值后转隔离或告警

#### 数据质量失败

- 不应无限重试
- 转入失败隔离区
- 保存失败原因和消息上下文

#### 路由或存储失败

- 优先保留并重试
- 超过阈值后转入失败隔离区
- 不要求 Agent 重发已确认消息

### 21.4 失败隔离区

Gateway 应具备 `Failed Message Store / Dead Letter Handling`，用于保存无法正常完成处理的消息，并记录：

- `transportMessageId`
- `recordDedupKey`
- `taskId`
- `agentId`
- `connectorInstanceId`
- 失败阶段
- 失败原因
- 首次失败时间
- 最后失败时间
- 重试次数
- 原始 payload 或引用位置

## 22. 动态分发设计

动态分发是选做增强能力，不作为第一版核心必做能力，但架构上应预留。

### 22.1 设计原则

- 同一采集任务或同一采集分区在同一时刻只能有一个有效 owner
- 动态分发采用“单主归属 + Checkpoint 交接”模型
- 迁移过程中优先保证数据完整性，允许少量重复但不允许无界丢失

### 22.2 基本迁移流程

1. `Data Gateway` 根据负载或策略决定迁移任务
2. Gateway 将任务状态标记为 `Rebalancing`
3. 原 Agent 停止继续拉取新数据
4. 原 Agent 刷完本地队列或完成安全封存
5. 原 Agent 上报最后可靠 `Checkpoint`
6. Gateway 将该 `Checkpoint` 下发给目标 Agent
7. 目标 Agent 从交接 `Checkpoint` 启动新的 `Connector Instance`
8. 迁移完成后由 Gateway 切换 owner 并销毁旧实例

### 22.3 第一版实现边界

第一版建议只支持任务级迁移和单实例迁移，不建议直接实现：

- 双活抢占式迁移
- 同一采集流多 Agent 并行消费
- 无停机热迁移

## 23. 时间语义、监控、日志与审计

### 23.1 时间语义

系统要求 Gateway 与 Agent 具备基础时钟同步能力，例如 `NTP`，但不以跨节点时间戳严格有序作为一致性前提。

建议定义以下时间语义：

- `sourceEventTime`
- `collectTime`
- `agentPersistTime`
- `gatewayAcceptTime`
- `processedTime`
- `storedTime`
- `commandIssuedTime`
- `commandExpireTime`

一致性主要依赖：

- `desiredStateVersion`
- `commandId`
- `transportMessageId`
- `recordDedupKey`
- `checkpoint`

### 23.2 监控粒度

监控采用分层粒度：

- 系统级
- Agent 级
- Task 级
- Connector Instance 级

Agent 定期上报实例级摘要，Gateway 聚合展示系统级、Agent 级、任务级和实例级视图。

### 23.3 运行日志

系统区分运行日志与审计日志。

#### Agent / Connector 运行日志

- Agent 本地保存注册、同步、上传、恢复、资源告警等运行日志
- 每个 Connector Instance 保存独立运行日志
- 日志必须支持实例级隔离与轮转，防止日志无限增长

#### Gateway 运行日志

- Control Plane 记录任务、配置、命令、程序包、Agent 生命周期等运行事件
- Data Plane 记录接入失败、标准化失败、路由失败、存储失败、背压触发等事件

第一版以“本地完整保留 + 中心摘要汇聚”为主，不强制全量实时日志回传。

### 23.4 审计日志

Gateway 侧集中保存审计日志，至少覆盖：

- 任务创建、修改、启停、删除
- 目标配置变更
- 程序包发布、升级、回滚
- Agent 注册、失效、替换
- 命令下发
- 动态分发 / 迁移
- 权限与身份相关变更

审计日志至少包含：

- 操作类型
- 操作对象
- 操作发起者
- 发起时间
- 执行结果
- 关联对象标识

## 24. 非功能性要求与容量说明

### 24.1 可扩展性

支持一个客户部署多个 Agent，支持多种 `Connector Type`、多程序包版本和多实例横向扩展。

### 24.2 可用性

`Data Gateway` 和 `Data Storage` 应按逻辑中心、物理集群方式建设，避免单点故障。

### 24.3 可观测性

应监控以下关键指标：

- Agent 心跳
- Connector Instance 状态
- 本地队列积压
- 采集吞吐
- 传输延迟
- 失败率
- Checkpoint 推进进度
- Gateway Ingress Queue 深度
- Data Plane 处理延迟
- Storage Writer 失败率

### 24.4 性能容量说明

本方案当前为逻辑架构方案，不在本文档中定义明确性能数值。后续详细设计阶段应结合目标客户规模补充以下容量基线：

- 目标 Agent 数量
- 单 Agent 目标 Connector 实例数
- 峰值采集吞吐量
- 接入与入库延迟目标
- 本地队列容量与存储保留目标

## 25. 第一版实现边界

第一版建议实现以下能力：

- 多 Agent 统一纳管
- Agent 托管多 Connector Instance
- `Connector Type / Package / Instance` 三层模型
- Gateway `Control Plane / Data Plane` 逻辑分层
- Agent 本地持久化消息队列
- Connector `Checkpoint` 机制
- Agent 本地原子提交
- Gateway 接入持久化与幂等去重
- 任务状态与实例状态管理
- Gateway 程序包下发和运行监控
- 安全通信、基本审计、日志摘要汇聚

第一版不建议实现以下能力：

- 复杂双活分发
- 同一采集流多 Agent 并行消费
- 无停机热迁移
- 进程内热替换
- 过于复杂的本地编排系统
- 强依赖精确时间同步的全局排序机制

## 26. 结论

本方案可以完整支撑原始需求，且职责边界清晰：

- `Business System` 负责配、看、用
- `Gateway Control Plane` 负责控、配、管、审计
- `Gateway Data Plane` 负责收、缓冲、处理、入库
- `Client Agent` 负责宿主、管理、缓冲、上传
- `Connector` 负责源端采集与 `Checkpoint` 语义
- `Data Storage` 负责最终存储与查询支撑

该方案是一版完整的总体逻辑架构方案，已经具备进入下一阶段详细设计的基础。后续若继续深化，应补充接口模型、消息模型、部署拓扑、关键时序图以及性能容量数值。
