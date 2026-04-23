# Distributed Data Collection System 详细设计总览

## 文档信息

- 文档名称：`Distributed Data Collection System 详细设计总览`
- 文档版本：`v0.6`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 修订历史

| 版本 | 日期 | 修订摘要 |
| --- | --- | --- |
| v0.1 | 2026-04-23 | 建立详细设计总览，明确文档拆分、总体模块映射、关键链路和落地顺序 |
| v0.2 | 2026-04-23 | 根据审阅意见补充安全约束传递、核心概念索引、需求追溯、多实例策略、监控审计粒度、动态分发预留和阶段映射 |
| v0.3 | 2026-04-23 | 将数据存储与可靠落地拆分为独立第 5 份详细设计文档，并补充文档边界与阅读路径 |
| v0.4 | 2026-04-23 | 对齐逻辑架构 `v0.6` 的首版边界和存储决策，修正文档范围与数量描述中的旧表述 |
| v0.5 | 2026-04-23 | 将总前提修订为多租户共享中心平台，并同步需求解释、核心概念和统一约束 |
| v0.6 | 2026-04-23 | 在 `02/03/04/05` 中补齐租户字段、租户范围校验和租户隔离落地后，回写总览文档的当前状态说明 |

## 1. 目的与范围

本文档是 `Distributed Data Collection System` 详细设计文档集的总览文档，用于说明详细设计的拆分方式、文档边界、统一约束、关键链路和建议实现顺序。

本文档不重复展开逻辑架构方案中的全部内容，而是承接以下上游文档：

- [Distributed Data Collection System 总体逻辑架构方案](../2026-04-22-distributed-data-collection-system-logical-architecture-design.md)

本轮详细设计文档重点覆盖：

- 功能设计
- 运行流程
- 控制流程
- 接口职责、请求/响应结构和关键字段说明

说明：

- 本轮详细设计文档已统一切换到多租户共享中心平台前提
- `02/03/04/05` 已补入租户字段、租户范围校验和存储隔离的基础约束

本轮详细设计文档暂不覆盖：

- 精确 API 契约
- 数据库表结构
- Kafka / PostgreSQL 的低层部署参数、Topic / 表级实现细节
- 部署脚本
- 运维 SOP

## 2. 文档清单与边界

本次详细设计按系统角色、公共模型和存储专题拆分为五份文档：

### 2.1 01-overview-detailed-design.md

作用：

- 说明详细设计整体范围
- 串联后续四份文档
- 定义统一设计约束和阅读路径

### 2.2 02-gateway-detailed-design.md

作用：

- 展开 `Gateway Control Plane` 和 `Gateway Data Plane` 的详细设计
- 说明 Gateway 侧功能分解、运行流程、控制流程、失败处理和对外接口职责

边界：

- 只写 Gateway 视角
- 不重复 Agent 内部实现细节

### 2.3 03-agent-connector-detailed-design.md

作用：

- 展开 `Client Agent` 和 `Connector` 的详细设计
- 说明宿主管理、程序包管理、实例生命周期、本地队列、checkpoint 提交、资源隔离、恢复流程

边界：

- 只写 Agent / Connector 视角
- 不重复 Gateway 内部处理实现

### 2.4 04-message-state-interface-detailed-design.md

作用：

- 定义公共消息模型、时间语义、状态机、控制面接口和数据面接口结构

边界：

- 作为公共模型与接口基线
- 避免 Gateway 文档与 Agent 文档各自维护一套消息与状态定义

### 2.5 05-data-storage-and-durable-ingest-detailed-design.md

作用：

- 展开中心侧可靠落地、最终查询存储、失败隔离与数据保留策略
- 说明从 Agent 上传进入 Gateway 之后，数据如何进入第一可靠落点并最终入库

边界：

- 不重复 Gateway 模块拆分
- 不重复 Agent 本地队列实现
- 不重复公共消息字段定义
- 专门负责存储分层、保留、补偿、重放和持久化约束

## 3. 详细设计与逻辑架构的映射

### 3.1 核心概念速览

下列概念定义以上游逻辑架构方案为准，本轮详细设计仅在此做最小索引，避免五份详细设计文档出现术语漂移。

| 概念 | 简述 | 主要展开文档 |
| --- | --- | --- |
| `Connector Type` | 采集能力的逻辑类型定义 | 03 |
| `Connector Package` | 某类 Connector 的可部署程序包 | 03 |
| `Connector Instance` | 某个 Agent 上实际运行的实例 | 03、04 |
| `Tenant` | 平台中的客户隔离边界 | 01、02、04、05 |
| `tenantId` | 租户归属标识，控制、数据、审计与存储的统一隔离维度 | 01、02、03、04、05 |
| `Task` | 一条采集任务及其运行约束 | 02、04 |
| `Checkpoint` | 源端采集进度及恢复上下文 | 03、04 |
| `Message` | 统一采集消息载体 | 04 |
| `Control Plane` | 任务、配置、命令、监控、审计控制面 | 02 |
| `Data Plane` | 数据接入、缓冲、处理、入库数据面 | 02 |
| `desiredStateVersion` | 目标状态版本号，Agent 以最新版本收敛 | 02、03、04 |
| `triggerMode` | 任务触发模式 | 02、04 |
| `offlinePolicy` | 断链时任务行为策略 | 03、04 |
| `backpressurePolicy` | 背压场景下的任务处理策略 | 02、03、04 |
| `Collection Partition` | 可独立分配和迁移的最小采集单元 | 02、03、04 |

### 3.2 逻辑架构主题映射

| 逻辑架构主题 | 详细设计文档 |
| --- | --- |
| Control Plane / Data Plane 边界 | 02 |
| Agent 宿主管理与 Connector 执行边界 | 03 |
| 消息模型、Checkpoint、状态机、接口 | 04 |
| 存储分层、可靠落地与入库持久化 | 05 |
| 统一约束、落地顺序、文档关系 | 01 |

### 3.3 原始需求到详细设计文档的追溯

| 需求 | 摘要 | 详细设计文档 | 设计落实说明 |
| --- | --- | --- | --- |
| `R1` | 多客户平台下多 Agent 部署 | 01、02、03 | 在总览中明确多租户共享中心平台边界，并在后续文档中补充租户绑定的 Agent 注册、管理与运行关系 |
| `R2` | Agent 按配置运行多个 Connector | 01、03、04 | 在总览中固定宿主边界，在 Agent 文档中展开实例托管，在公共模型文档中定义状态与接口 |
| `R3` | 类型 / 程序包 / 多实例 | 01、03、04 | 在总览中汇总概念，在 Agent 文档中展开程序包与实例模型，在公共模型文档中统一字段与状态 |
| `R4` | Agent 负责生命周期管理 | 01、03 | 在总览中明确 supervisor 约束，在 Agent 文档中展开创建、停止、重启、监控与恢复 |
| `R5` | 数据必须入库 | 01、02、04、05 | 在总览中固定数据链路，在 Gateway 文档中展开处理链路，在公共模型文档中定义消息与响应语义，在存储文档中展开可靠落地与最终入库 |
| `R6` | Gateway 统一控制、下发与监控 | 01、02、04 | 在总览中明确 Control Plane / Data Plane 约束，在 Gateway 文档中展开控制流程与接口 |
| `R7` | 异常场景下尽量不丢数 | 01、02、03、04 | 在总览中固化可靠性约束，在 Gateway/Agent 文档中展开失败处理、背压、恢复与原子提交 |
| `R8` | 动态分发与数据完整性 | 01、02、03、04 | 在总览中标记为增强能力预留，在后续文档中保持单主归属、checkpoint 交接与幂等语义一致 |

## 4. 总体模块映射

### 4.1 Gateway 侧

- `Task Management`
- `Agent Registry`
- `Package & Config Management`
- `Command Coordination`
- `Monitoring & Alerting`
- `Audit Log`
- `Ingress Access`
- `Ingress Persistent Queue`
- `Normalize / Transform`
- `Dedup / Idempotency`
- `Route`
- `Storage Writer`
- `Failed Message Handling`

### 4.2 Agent 侧

- `Command Handler`
- `Desired State Reconciler`
- `Package Manager`
- `Connector Manager`
- `Local Persistent Queue`
- `Local State Store`
- `Runtime Monitor`
- `Recovery Manager`

### 4.3 Connector 侧

- `Source Adapter`
- `Collector Runtime`
- `Checkpoint Producer`
- `Health Reporter`

## 5. 关键设计约束

本轮详细设计必须遵守以下统一约束：

### 5.1 部署边界

- 统一中心平台服务多个租户
- 每个租户在现场可部署多个 Agent
- 每个 Agent 只能归属一个租户
- 控制对象、数据对象、审计对象和存储对象都必须在租户维度下隔离

### 5.2 安全边界

- `Agent <-> Gateway` 使用 `TLS`
- 节点身份认证优先采用 `mTLS`
- Connector 源端凭证不得写死在程序包中
- 源端凭证由 `Gateway` 配置管理并由 `Agent` 按实例安全注入
- 业务用户权限与 Agent 节点身份权限必须分离
- 任何控制面查询、写操作和数据查询都必须受租户范围约束
- 运行日志不得明文记录敏感凭证
- 任务、配置、程序包、命令、迁移、身份变更等关键动作必须纳入审计范围，并能够追溯到租户

### 5.3 控制与数据分离

- Gateway 逻辑上拆分为 `Control Plane` 和 `Data Plane`
- 控制流与数据流接口分离
- Control Plane 不处理采集正文数据

### 5.4 多实例与协调约束

- 第一版 `Gateway Control Plane` 写路径采用“单 Leader + 共享元数据存储”
- `Gateway Data Plane` 应支持多实例部署，并通过负载均衡承接上传流量
- 同一租户内任务对象的目标状态变更必须串行化提交，避免多写冲突

### 5.5 通信模式

- 控制面采用“控制拉”
- 数据面采用“数据推”
- `desiredState` 是事实来源，命令是收敛信号
- 重连恢复时以最新 `desiredStateVersion` 为准，不补执行过期命令

### 5.6 可靠性约束

- Agent 本地持久化消息队列是第一可靠落点
- `Checkpoint` 与消息入队必须在 Agent 本地原子提交
- Gateway 仅在接入持久化成功后返回 `Accepted`
- 系统采用“至少一次 + 幂等去重”

### 5.7 监控与审计粒度约束

- 监控至少覆盖系统级、租户级、Agent 级、Task 级和 `Connector Instance` 级五层粒度
- 审计日志至少记录租户标识、操作类型、操作对象、发起者、发起时间、目标版本和执行结果
- 运行监控与审计日志在设计上分离，但都必须可按 Agent、Task、Instance 追溯

### 5.8 运行隔离约束

- 每个 Connector Instance 默认独立进程运行
- Agent 是本地宿主和 supervisor
- 配置变更默认通过实例级受控重启或受控切换生效

## 6. 统一设计原则

### 6.1 先目标状态，后执行命令

所有控制行为最终都收敛到 `desiredStateVersion`，而不是依赖历史命令重放。`desiredState` 是事实来源，`command` 仅是促使 Agent 尽快收敛的执行信号；在恢复或重连场景下，Agent 以最新目标版本为准，不补执行过期命令。

### 6.2 先局部吸收波动，后逐级传播压力

背压遵循“先本层消化，超阈值再向上游传播”的原则。

### 6.3 先保存数据，再推进进度

任何 checkpoint 推进都必须以本地可靠持久化完成为前提。

### 6.4 允许少量重复，不允许无界丢失

通过 `transportMessageId` 和 `recordDedupKey` 两层标识实现可靠传输与业务去重。

### 6.5 单主控制，避免多写冲突

第一版 Gateway Control Plane 写路径采用单 Leader 协调，避免控制冲突。

### 6.6 动态分发作为增强能力预留

动态分发不作为第一版基础闭环的前置条件，但详细设计必须为其预留一致的状态、checkpoint 和幂等语义。后续扩展时继续遵循“单主归属 + checkpoint 交接”的原则。

## 7. 关键链路总览

### 7.1 控制链路

`Business System -> Gateway Control Plane -> Agent -> Connector`

### 7.2 数据链路

`Source -> Connector -> Agent Local Persistent Queue -> Gateway Data Plane -> Storage`

### 7.3 状态链路

`Connector -> Agent -> Gateway Control Plane -> Business System`

### 7.4 恢复链路

`Agent Reconnect -> State Report -> Desired State Sync -> Backlog Replay -> Resume`

## 8. 实现优先级与建议落地顺序

建议实现顺序如下。该顺序与逻辑架构 `v0.6` 的“第一版实现边界”保持一致：Phase 1-3 对应首版建议实现范围，Phase 4 对应增强能力预留。

### Phase 1：控制与运行骨架

- Agent 注册
- 心跳与目标状态同步
- Task / Agent / Package 元数据管理
- Connector 独立进程托管

对应逻辑架构首版边界：

- Gateway `Control Plane / Data Plane` 逻辑分层
- 多 Agent 统一纳管
- Agent 托管多 Connector 实例
- 控制拉 / 数据推的基础通信模式

### Phase 2：可靠采集闭环

- Agent 本地持久化队列
- 本地原子提交
- 数据面上传与接收确认
- Gateway Ingress 幂等

对应逻辑架构首版边界：

- Agent 本地持久化消息队列
- Connector checkpoint 机制
- Gateway 接入持久化与幂等去重
- 任务状态与实例状态的基础闭环

### Phase 3：处理与稳定性

- 标准化、去重、路由、入库
- 失败隔离
- 背压反馈
- 离线恢复

对应逻辑架构首版边界：

- Data Plane 处理链路
- 背压与失败隔离
- 安全通信、基本审计、日志摘要汇聚
- 运行监控与状态汇聚

### Phase 4：增强能力

- 程序包升级与回滚
- 监控与审计完善
- 动态分发

对应逻辑架构增强项：

- 动态分发
- 更完整的审计和运维支撑
- 更细粒度的升级、回滚和灰度策略

## 9. 阅读路径

建议按以下顺序阅读本轮详细设计：

1. `01-overview-detailed-design.md`
2. `04-message-state-interface-detailed-design.md`
3. `02-gateway-detailed-design.md`
4. `03-agent-connector-detailed-design.md`
5. `05-data-storage-and-durable-ingest-detailed-design.md`

推荐原因：

- 先建立总览和约束
- 再掌握公共模型、状态和接口，因为 `02` 和 `03` 都依赖 `04` 中的消息结构、状态机和统一接口语义
- 再分别进入 Gateway 和 Agent / Connector 设计，避免在阅读局部实现前缺少公共语义基础
- 最后阅读存储专题文档，将前面已经确定的消息、状态和处理链路收口到可靠落地和最终入库设计

## 10. 后续深化方向

本轮详细设计完成后，下一步建议按以下优先级继续输出：

1. `API` 契约草案
2. 关键时序图
3. 数据库存储设计
4. 部署拓扑设计
5. 运维与故障处理手册
