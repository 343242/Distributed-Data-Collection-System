# Distributed Data Collection System 数据存储与可靠落地详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System 数据存储与可靠落地详细设计`
- 文档版本：`v0.1`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 修订历史

| 版本 | 日期 | 修订摘要 |
| --- | --- | --- |
| v0.1 | 2026-04-23 | 建立数据存储与可靠落地详细设计，明确中心第一可靠落点、最终查询存储、失败隔离、保留与补偿策略 |

## 1. 目的与范围

本文档专门说明从 `Connector -> Agent -> Gateway -> Storage` 这条链路中的中心侧数据存储设计，重点回答以下问题：

- Agent 上传的数据先落到哪里
- Gateway 在什么边界上返回 `Accepted`
- 中心第一可靠落点如何与最终查询存储解耦
- 原始数据、最终数据和失败隔离数据如何分层持久化
- 如何通过存储设计实现“允许重复，但最大化不丢失”
- 第一版推荐采用什么类型的存储产品组合

本文档不展开：

- Agent 本地持久化队列的内部实现
- Connector 采集逻辑
- 控制面模块拆分
- 精确数据库表结构和索引语句

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

### 2.2 Business Query Storage

用于存放最终可查询、可消费的数据结果。

核心职责：

- 保存标准化、去重、路由后的最终数据
- 提供面向 `Business System` 的查询能力
- 通过唯一约束或 `UPSERT` 实现幂等生效
- 支持历史查询、补偿写入和结果重建

## 3. 数据从采集到入库的落地链路

完整落地链路如下：

`Connector -> Agent Local Persistent Queue -> Gateway Ingress Persistent Queue -> Durable Ingest Storage -> Normalize / Transform -> Dedup / Route -> Business Query Storage`

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
4. Gateway 将消息写入 `Durable Ingest Storage` 或确认其已由接入持久化层可靠承接
5. 只有在这一步成功后，Gateway 才返回 `Accepted`

换句话说：
- `Accepted` 的含义不是“已经写入最终查询库”
- 而是“已经进入中心第一可靠落点”

### 4.3 数据内容

`Durable Ingest Storage` 中保留的数据至少应能恢复或重放：

- `transportMessageId`
- `recordDedupKey`
- `taskId`
- `agentId`
- `connectorInstanceId`
- `checkpoint`
- `collectTime`
- `payloadFormat`
- `payload`
- `metadata`
- `gatewayAcceptTime`

### 4.4 第一版推荐实现类型

第一版推荐采用：

- `Kafka` 或同类具备持久化、顺序写入、可重放能力的日志 / 队列型存储

原因：

- 适合做中心第一可靠落点
- 便于与后续处理链解耦
- 支持消费失败后的重试和重放
- 更符合“允许重复，但最大化不丢失”的目标

当第一版规模较小且必须简化时，可暂时使用 Gateway 本地持久化接入表替代，但该方案不作为长期推荐路径。

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

### 5.3 第一版推荐实现类型

第一版推荐采用：

- `PostgreSQL`

原因：

- 具备事务能力
- 适合控制面和业务查询共用关系型数据模型
- 支持唯一约束和 `UPSERT`
- 适合作为第一版最终结果存储

若后续分析查询成为主要场景，可增加列式分析存储副本，例如 `ClickHouse`，但不替代第一版的最终结果主存储。

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

至少保留：

- `transportMessageId`
- `recordDedupKey`
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

### 8.2 补偿原则

当处理链某一环节失败时：

- 不要求 Agent 重发已确认数据
- 优先在中心侧通过重试、补偿或重放解决

### 8.3 重放原则

重放必须满足：

- 来源于 `Durable Ingest Storage` 中已确认接收的数据
- 重放时仍走标准化、去重、路由和最终入库流程
- 最终依赖 `recordDedupKey` 实现幂等生效

## 9. 第一版推荐落地方案

第一版推荐存储组合如下：

- `Durable Ingest Storage`：`Kafka`
- `Business Query Storage`：`PostgreSQL`
- 可选分析副本：后续按需求增加 `ClickHouse`

职责边界如下：

- `Kafka`
  - 中心第一可靠落点
  - 提供可重放日志能力
- `PostgreSQL`
  - 控制面元数据存储
  - 最终业务查询存储
  - 幂等结果写入
- `ClickHouse`
  - 不作为第一版必交付
  - 后续按分析需求扩展

### 9.1 第一版明确不采用的模式

第一版不采用以下模式作为主方案：

- 仅使用最终业务库直接承接 Gateway 上传并作为第一可靠落点
- 让最终查询库存同时承担接入缓冲、失败补偿和业务查询全部职责
- 以分析型存储替代中心第一可靠落点

## 10. 与其他详细设计文档的关系

- Gateway Control Plane / Data Plane 模块和接口见 `02`
- Agent 本地持久化队列和 checkpoint 原子提交见 `03`
- 消息字段、时间语义、状态机和公共接口结构见 `04`
- 本文档专注于中心侧数据如何可靠落地、如何最终入库以及如何补偿和重放
