# Distributed Data Collection System 数据存储与可靠落地详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System 数据存储与可靠落地详细设计`
- 文档版本：`v0.5`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 修订历史

| 版本 | 日期 | 修订摘要 |
| --- | --- | --- |
| v0.1 | 2026-04-23 | 建立数据存储与可靠落地详细设计，明确中心第一可靠落点、最终查询存储、失败隔离、保留与补偿策略 |
| v0.2 | 2026-04-23 | 根据审阅意见澄清 Ingress Queue 与 Durable Ingest Storage 关系、Accepted 边界、失败隔离落点、Kafka 消费模型、保留周期与实例隔离策略 |
| v0.3 | 2026-04-23 | 统一首版中间件决策表达，补全 Durable Ingest Storage 的公共消息字段，并对齐动态分发的首版边界说明 |
| v0.4 | 2026-04-23 | 将存储专项设计修订为多租户共享中心平台模型，补充租户隔离、租户字段和租户范围下的重放与查询约束 |
| v0.5 | 2026-04-23 | 澄清 PostgreSQL 是统一产品选型而非控制面与结果库共用同一逻辑实例 |

## 1. 目的与范围

本文档专门说明从 `Connector -> Agent -> Gateway -> Storage` 这条链路中的中心侧数据存储设计，重点回答以下问题：

- Agent 上传的数据先落到哪里
- Gateway 在什么边界上返回 `Accepted`
- 中心第一可靠落点如何与最终查询存储解耦
- 原始数据、最终数据和失败隔离数据如何分层持久化
- 如何通过存储设计实现“允许重复，但最大化不丢失”
- 第一版采用什么存储产品组合

本文档不展开：

- Agent 本地持久化队列的内部实现
- Connector 采集逻辑
- 控制面模块拆分
- 精确数据库表结构和索引语句

说明：

- 本文档基于多租户共享中心平台前提展开
- `Durable Ingest Storage`、`Business Query Storage` 和失败隔离数据层都必须在租户维度下隔离

## 2. 存储分层总览

中心侧存储采用两层模型：

1. `Durable Ingest Storage`
2. `Business Query Storage`

两层职责不同，不应混为一个单一数据库。

### 2.1 Durable Ingest Storage

用于承接 Gateway Data Plane 已确认接收的数据，是中心侧第一可靠落点。

核心职责：

- 在 Gateway 返回 `Accepted` 前可靠持久化数据
- 提供可重放、可补偿、可追溯的数据保留能力
- 与后续标准化、去重、路由、最终查询入库解耦
- 在最终查询库存储变慢或不可用时继续保留待处理数据
- 在共享中心平台中按租户维度隔离原始接入数据

与 `Ingress Persistent Queue` 的关系：

- `Ingress Persistent Queue` 是 Gateway Data Plane 的逻辑接入队列抽象
- `Durable Ingest Storage` 是该逻辑接入队列的持久化后端
- 第一版中，两者在职责上不是两层串行写入，而是同一接入确认边界的“逻辑抽象 + 持久化实现”
- 在第一版采用方案下，`Ingress Persistent Queue` 的持久化后端就是 `Kafka`

### 2.2 Business Query Storage

用于存放最终可查询、可消费的数据结果。

核心职责：

- 保存标准化、去重、路由后的最终数据
- 提供面向 `Business System` 的查询能力
- 通过唯一约束或 `UPSERT` 实现幂等生效
- 支持历史查询、补偿写入和结果重建
- 在共享中心平台中按租户维度隔离结果数据和失败隔离数据

## 3. 数据从采集到入库的落地链路

完整落地链路如下：

`Connector -> Agent Local Persistent Queue -> Gateway Ingress Persistent Queue / Durable Ingest Storage -> Normalize / Transform -> Dedup / Route -> Business Query Storage`

这条链路中的关键确认边界如下：

1. `Agent Local Persistent Queue`
- 现场第一可靠落点
- `checkpoint` 在本地原子提交后推进

2. `Durable Ingest Storage`
- 中心第一可靠落点
- Gateway 只有在数据可靠进入该层后才返回 `Accepted`

3. `Business Query Storage`
- 最终可消费结果层
- 允许重试写入和幂等更新

## 4. Durable Ingest Storage 详细设计

### 4.1 职责定位

`Durable Ingest Storage` 不等同于最终业务库，它承担以下职责：

- 中心侧第一可靠持久化
- 批次保留与重放
- 后续处理失败时的缓冲
- 原始接入消息追溯

### 4.2 数据写入边界

Gateway Data Plane 的写入流程应满足：

1. Agent 发送上传批次
2. Gateway 完成接入鉴权和基础校验
3. Gateway 将批次写入 `Ingress Persistent Queue`
4. `Ingress Persistent Queue` 将批次可靠追加到其持久化后端 `Durable Ingest Storage`
5. 只有在这一步成功后，Gateway 才返回 `Accepted`

换句话说：
- `Accepted` 的含义不是“已经写入最终查询库”
- 而是“已经进入中心第一可靠落点”

第一版明确边界：

- 写入 `Kafka` 成功即视为 `Ingress Persistent Queue` 写入成功
- 不要求再额外同步写第二份中心接入存储后才返回 `Accepted`

### 4.3 数据内容

`Durable Ingest Storage` 中保留的数据至少应能恢复或重放：

- `transportMessageId`
- `recordDedupKey`
- `tenantId`
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
- `gatewayAcceptTime`

### 4.4 第一版采用实现类型

第一版明确采用：

- `Kafka`

原因：

- 适合做中心第一可靠落点
- 便于与后续处理链解耦
- 支持消费失败后的重试和重放
- 更符合“允许重复，但最大化不丢失”的目标

### 4.5 Kafka 消费模型

第一版采用 `Kafka` 时，消费模型定义如下：

- Producer：
  - `Gateway Ingress Access` / `Ingress Persistent Queue` 写入 Kafka
- Consumer：
  - `Gateway Data Plane Processing Worker` 作为消费组读取 Kafka

分区键建议：

- 默认使用 `tenantId + taskId`
- 若任务启用了 `Collection Partition`，则使用 `tenantId + taskId + collectionPartitionId` 作为分区键

offset 管理原则：

- 只有当消息完成以下任一结果后，消费者才提交 offset：
  - 成功写入 `Business Query Storage`
  - 成功写入失败隔离存储
- 若发生瞬时失败，则不提交 offset，交由重试和退避机制处理
- 若发生不可恢复的数据质量失败，在失败隔离写入成功后提交 offset，避免阻塞分区

接入层幂等原则：

- Kafka 本身不是按 `transportMessageId` 做业务去重的组件
- Gateway Ingress 必须维护一份短窗口的 `transportMessageId` 接入确认账本
- 当 Agent 重试上传时：
  - 若确认账本中已存在同一 `transportMessageId` 的已接收记录，则 Gateway 直接返回已接受结果
  - 若不存在，则继续执行写入

该账本的实现可与 Gateway 接入元数据一并持久化，但不改变“Kafka 是中心第一可靠落点”的架构决策

## 5. Business Query Storage 详细设计

### 5.1 职责定位

`Business Query Storage` 存放的是最终结果数据，而不是接入临时缓冲。

它负责：

- 保存标准化和去重后的结果数据
- 支撑业务系统查询
- 支撑历史补偿和幂等更新

### 5.2 写入原则

写入 `Business Query Storage` 时应满足：

- 处理链已完成标准化、去重和路由
- 写入动作允许重试
- 最终生效由唯一键 / 幂等约束保证

### 5.3 第一版采用实现类型

第一版明确采用：

- `PostgreSQL`

原因：

- 具备事务能力
- 适合作为控制面元数据和业务查询存储的统一关系型产品选型
- 支持唯一约束和 `UPSERT`
- 适合作为第一版最终结果存储

若后续分析查询成为主要场景，可增加列式分析存储副本，例如 `ClickHouse`，但不替代第一版的最终结果主存储。

隔离策略：

- 第一版 `PostgreSQL` 虽然同时用于控制面元数据和最终结果存储，但必须采用隔离部署
- 不允许控制面元数据和高吞吐业务结果写入共用同一个逻辑实例
- 至少应满足：
  - Control Plane 元数据独立数据库 / 实例
  - Business Query Storage 独立数据库 / 实例
- 在 `Business Query Storage` 内部，结果数据表和失败隔离表必须包含 `tenantId`
- 唯一约束、查询过滤、重放过滤和补偿操作都必须以 `tenantId` 为前置条件

## 6. 原始数据、结果数据、失败隔离数据分层

### 6.1 原始接入数据层

存于 `Durable Ingest Storage`，保留接入后的原始消息。

用途：

- 重放
- 排障
- 审计
- 补偿处理

### 6.2 最终结果数据层

存于 `Business Query Storage`，保留最终可查询的数据。

用途：

- 业务系统查询
- 历史统计
- 对外消费

### 6.3 失败隔离数据层

对无法正常完成处理的数据，应单独保留失败隔离记录，而不直接丢弃。

与 `Failed Message Handling` 的关系：

- `Failed Message Handling` 是 Gateway Data Plane 中负责写入失败隔离记录的处理模块
- 失败隔离数据层是该模块对应的持久化落点

第一版落点策略：

- 失败隔离数据默认存放在 `Business Query Storage` 的独立隔离 schema / 表中
- `rawPayloadOrReference` 可直接保存原始载荷，也可保存回指 `Durable Ingest Storage` 的引用
- 因此失败隔离是一个独立逻辑数据层，但第一版物理上不单独引入第三种数据库产品

至少保留：

- `transportMessageId`
- `recordDedupKey`
- `tenantId`
- `taskId`
- `agentId`
- `connectorInstanceId`
- `failureStage`
- `failureReason`
- `firstFailureTime`
- `lastFailureTime`
- `retryCount`
- `rawPayloadOrReference`
- `failureContext`

## 7. 幂等去重与存储约束

### 7.1 接入层幂等

`Durable Ingest Storage` 应支持基于 `transportMessageId` 的接入层幂等语义。

目标：

- Agent 重试上传不会导致同一条接入消息被无限重复接收

第一版实现说明：

- 不依赖 Kafka 的 EOS 特性来完成按消息 ID 的业务去重
- 由 Gateway Ingress 维护短窗口接入确认账本，按 `transportMessageId` 判断是否为已接受消息
- 接入确认账本中的幂等判断必须在租户范围内执行
- Kafka 负责可靠落地和可重放，不直接承担消息 ID 级别的接入幂等判断

### 7.2 结果层幂等

`Business Query Storage` 应支持基于 `recordDedupKey` 的结果层幂等语义。

目标：

- 同一条源记录即使因重采、迁移或恢复重复进入处理链，最终也只生效一次

### 7.3 第一版约束

第一版系统不追求端到端 `exactly-once`，而是明确采用：

- 至少一次传递
- 存储侧幂等去重

这意味着：

- 可以重复投递
- 可以重复消费
- 但不允许因为最终库存储抖动或处理失败而轻易丢数

## 8. 数据保留、补偿与重放

### 8.1 保留原则

`Durable Ingest Storage` 不应只保留“当前正在处理”的短暂数据，而应具备明确的保留期。

其保留周期至少应覆盖：

- Gateway 短时处理失败
- 结果层写入重试
- 人工补偿与重放窗口

第一版建议：

- `Durable Ingest Storage` 默认保留期建议为 `7 天`
- 在存储资源受限场景下，不应低于 `72 小时`
- 失败隔离数据保留期建议为 `30 天`

保留周期决策因素：

- 处理链最大重试时间
- 人工补偿窗口
- 业务历史追溯要求
- 存储成本与容量规划

### 8.2 补偿原则

当处理链某一环节失败时：

- 不要求 Agent 重发已确认数据
- 优先在中心侧通过重试、补偿或重放解决

### 8.3 重放原则

重放必须满足：

- 来源于 `Durable Ingest Storage` 中已确认接收的数据
- 重放时仍走标准化、去重、路由和最终入库流程
- 最终依赖 `recordDedupKey` 实现幂等生效

重放范围建议至少支持按以下维度过滤：

- `tenantId`
- `taskId`
- `agentId`
- 时间范围
- 失败阶段

与动态分发的关系：

- 本节仅约束未来增强能力的存储语义，不代表第一版默认启用动态分发
- 存储重放与 `checkpoint` 交接是两套不同语义
- 动态分发中的 `checkpoint` 交接决定“新 owner 从源端哪里继续采”
- 存储重放决定“中心已接收但未完成处理的数据如何重新进入处理链”
- 迁移后的重放仍按中心侧已接收数据执行，不要求源端重新读取

## 9. 第一版落地方案

第一版采用的存储组合如下：

- `Durable Ingest Storage`：`Kafka`
- `Business Query Storage`：`PostgreSQL`
- 可选分析副本：后续按需求增加 `ClickHouse`

职责边界如下：

- `Kafka`
  - 中心第一可靠落点
  - 提供可重放日志能力
  - 消息必须携带 `tenantId`，供后续处理、查询和重放隔离使用
- `PostgreSQL`
  - 作为控制面元数据存储和最终业务查询存储的统一产品类型
  - 控制面元数据与最终业务查询必须使用独立数据库 / 实例隔离部署
  - 幂等结果写入
  - 结果数据、失败隔离和查询过滤都必须按 `tenantId` 隔离
- `ClickHouse`
  - 不作为第一版必交付
  - 后续按分析需求扩展

### 9.1 第一版明确不采用的模式

第一版不采用以下模式作为主方案：

- 仅使用最终业务库直接承接 Gateway 上传并作为第一可靠落点
- 让最终查询库存同时承担接入缓冲、失败补偿和业务查询全部职责
- 以分析型存储替代中心第一可靠落点

### 9.2 第一版容量规划建议

第一版容量规划至少应覆盖以下维度：

- Kafka topic 保留期与保留容量
- Kafka 分区数与副本数
- PostgreSQL 结果表空间增长速度
- 失败隔离表增长速度

第一版建议范围：

- Kafka topic 分区数：按 `taskId` 并发度和消费并行度预估，首版不应少于 `3`
- Kafka 副本数：生产环境建议 `3`
- Kafka `min.insync.replicas`：建议不低于 `2`
- PostgreSQL 应为结果数据和失败隔离数据预留独立表空间或独立存储配额

## 10. 与其他详细设计文档的关系

- Gateway Control Plane / Data Plane 模块和接口见 `02`
- Agent 本地持久化队列和 checkpoint 原子提交见 `03`
- 消息字段、时间语义、状态机和公共接口结构见 `04`
- 本文档专注于中心侧数据如何可靠落地、如何最终入库以及如何补偿和重放
