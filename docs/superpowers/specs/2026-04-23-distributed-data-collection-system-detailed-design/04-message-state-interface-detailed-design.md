# Distributed Data Collection System 消息、状态机与接口详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System 消息、状态机与接口详细设计`
- 文档版本：`v0.5`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 修订历史

| 版本 | 日期 | 修订摘要 |
| --- | --- | --- |
| v0.1 | 2026-04-23 | 建立公共模型、状态机、接口和时间语义的基础骨架 |
| v0.2 | 2026-04-23 | 根据审阅意见补充双层标识生成规则、去重层次、时间字段约束、Rebalancing 状态、backpressureHints、失败查询结构与统一错误结构 |
| v0.3 | 2026-04-23 | 交叉一致性复核，统一与 Gateway 详细设计的共享接口字段定义 |
| v0.4 | 2026-04-23 | 为动态分发补充迁移相关接口与共享字段语义 |
| v0.5 | 2026-04-23 | 将动态分发相关接口标记为增强能力预留，并与逻辑架构首版边界保持一致 |

## 1. 目的与范围

本文档定义系统详细设计中共享的公共基础：

- 核心数据模型
- 时间语义
- 状态机
- 控制面接口结构
- 数据面接口结构
- 错误与响应语义

本文档是 `Gateway` 与 `Agent / Connector` 设计的公共基线。

## 2. 核心数据模型

### 2.1 TaskDefinition

用于表达业务定义的采集任务。

关键字段：

| 字段 | 说明 |
| --- | --- |
| taskId | 任务标识 |
| taskName | 任务名称 |
| connectorType | 所需 Connector 类型 |
| triggerMode | 触发模式 |
| offlinePolicy | 离线策略 |
| backpressurePolicy | 背压策略 |
| sourceConfigRef | 源配置引用 |
| targetAgentSelector | Agent 选择条件 |
| packageVersionPolicy | 包版本策略 |
| priority | 优先级 |
| enabled | 是否启用 |

### 2.2 DesiredState

用于表达 Gateway 下发给 Agent 的目标运行状态。

关键字段：

| 字段 | 说明 |
| --- | --- |
| desiredStateVersion | 目标状态版本 |
| taskBindings | 任务到实例绑定 |
| targetInstances | 目标实例定义集合 |
| ownershipAssignments | 当前 owner 分配信息 |
| packageRequirements | 程序包要求 |
| configRequirements | 配置要求 |
| effectiveTime | 生效时间 |

### 2.3 Command

用于表达促使 Agent 尽快收敛的控制信号。

关键字段：

| 字段 | 说明 |
| --- | --- |
| commandId | 命令标识 |
| commandType | 命令类型 |
| desiredStateVersion | 关联目标版本 |
| targetAgentId | 目标 Agent |
| targetInstanceId | 目标实例 |
| collectionPartitionId | 目标分区，可选 |
| issuedAt | 生成时间 |
| expireAt | 失效时间 |
| commandPayload | 命令参数 |

### 2.4 PackageMetadata

关键字段：

| 字段 | 说明 |
| --- | --- |
| connectorType | 类型 |
| packageVersion | 版本 |
| packageUri | 下载定位 |
| checksum | 完整性校验 |
| packageSize | 包大小 |
| configSchemaVersion | 配置 schema 版本 |
| runtimeCompatibility | 运行兼容性说明 |

使用说明：

- `PackageMetadata` 由 Gateway Control Plane 管理和发布
- Agent 通过 `desiredState.packageRequirements` 识别所需包版本，再按 `packageUri` 主动拉取
- 具体发布与获取流程见 `02` 和 `03`

### 2.5 CheckpointModel

关键字段：

| 字段 | 说明 |
| --- | --- |
| checkpointValue | 位点值 |
| checkpointType | 位点类型 |
| sourceContext | 恢复上下文 |
| capturedAt | 采集时间 |
| committedAt | Agent 提交时间 |

说明：

- `Checkpoint` 不仅是位点值
- 还必须包含恢复时需要的源端上下文

`sourceContext` 至少应覆盖：

| 字段 | 说明 |
| --- | --- |
| sourcePartitionIdentity | 源分区、文件或分片标识 |
| sourceCursorContext | cursor / offset / 文件位置等上下文 |
| sourceSchemaOrSnapshotHint | 恢复时所需的源端 schema 或快照提示 |
| sourceSessionHint | 必要时用于恢复连接上下文的会话提示 |

### 2.6 CollectionMessage

关键字段：

| 字段 | 说明 |
| --- | --- |
| transportMessageId | 传输层消息标识 |
| recordDedupKey | 业务层去重键 |
| taskId | 任务标识 |
| agentId | Agent 标识 |
| connectorType | Connector 类型 |
| connectorInstanceId | 实例标识 |
| sourceId | 源标识 |
| checkpoint | 对应 checkpoint |
| sourceEventTime | 源事件时间 |
| collectTime | 采集时间 |
| payloadFormat | 数据格式 |
| payload | 数据载荷 |
| metadata | 补充上下文 |

约束：

- `metadata` 不参与路由、去重、状态管理或控制决策
- `transportMessageId` 由 Agent 在消息写入本地持久化队列时生成
- 同一条本地消息在重试上传时必须保持相同的 `transportMessageId`
- `transportMessageId` 推荐包含 `agentId` 前缀，并结合本地唯一序列或高熵标识生成
- `recordDedupKey` 的去重语义由 `Connector Type` 定义，并由 Agent 随消息一并持久化和上传

`transportMessageId` 生成规则：

- 生成时机：Agent 将消息成功写入本地持久化队列时
- 唯一性范围：单系统实例内全局唯一
- 稳定性要求：消息重试上传时保持不变
- 推荐格式：`<agentId>-<localSequenceOrTime>-<highEntropySuffix>`

`recordDedupKey` 生成规则：

- 若源端存在稳定唯一主键，则优先使用 `sourceId + sourceRecordId`
- 若源端无天然主键但存在稳定 checkpoint 与局部记录标识，则使用 `sourceId + checkpoint + subRecordKey`
- 若两者都不存在，则使用关键业务字段归一化后生成摘要键
- 同一条源记录在重复采集时应尽量生成相同的 `recordDedupKey`

去重层次：

- Gateway Ingress 基于 `transportMessageId` 执行接入层幂等去重
- Gateway Processing / Storage 基于 `recordDedupKey` 执行业务层去重与最终入库幂等

## 3. 时间语义

系统定义以下时间语义：

| 字段 | 说明 | 产生方 | 要求 |
| --- | --- | --- | --- |
| sourceEventTime | 源数据产生时间 | 源端或 Connector | 可选，源端缺失时可为空 |
| collectTime | Connector 采到数据时间 | Connector | 必填 |
| agentPersistTime | Agent 本地持久化成功时间 | Agent | 必填 |
| gatewayAcceptTime | Gateway 接收成功时间 | Gateway Data Plane | 对已接收批次必填 |
| processedTime | Gateway 完成处理时间 | Gateway Data Plane | 对已处理消息必填 |
| storedTime | 数据入库存储时间 | Storage Writer | 对成功入库消息必填 |
| commandIssuedTime | 命令生成时间 | Gateway Control Plane | 命令类对象必填 |
| commandExpireTime | 命令失效时间 | Gateway Control Plane | 对具备失效语义的命令必填 |

原则：

- 各节点需具备基础时间同步能力，例如 `NTP`
- 不把全局时间戳排序作为唯一一致性依据

完整性约束：

- 采集链路至少应保证 `collectTime -> agentPersistTime -> gatewayAcceptTime` 的语义链条可追踪
- 若消息完成处理和入库，则 `processedTime`、`storedTime` 不应早于前置时间
- 时间字段的消费方只能按语义使用，不应把跨节点时间先后当作唯一一致性依据

## 4. 状态机设计

### 4.1 Task 状态机

说明：

- `Rebalancing` 是为动态分发增强能力预留的任务状态
- 第一版基础闭环默认不进入该状态，除非显式启用动态分发

状态定义：

- `Draft`
- `Scheduled`
- `Deploying`
- `Rebalancing`
- `Running`
- `Paused`
- `Failed`
- `Stopped`

触发条件示意：

| 当前状态 | 触发 | 下一状态 |
| --- | --- | --- |
| Draft | 启用任务 | Scheduled |
| Scheduled | 开始收敛 | Deploying |
| Deploying | 实例就绪 | Running |
| Running | 触发迁移或重分配 | Rebalancing |
| Rebalancing | 迁移完成 | Running |
| Rebalancing | 迁移失败 | Failed |
| Running | 人工暂停 | Paused |
| Paused | 恢复 | Running |
| Running | 自动恢复失败 | Failed |
| Failed | 重新部署 | Deploying |
| Running / Paused / Failed / Rebalancing | 停止 | Stopped |

### 4.2 Connector Instance 状态机

状态定义：

- `Created`
- `Starting`
- `Running`
- `Degraded`
- `Retrying`
- `Stopped`
- `Failed`
- `Destroyed`

### 4.3 Agent 网络状态机

状态定义：

- `Online`
- `Degraded`
- `Disconnected`
- `Recovering`
- `Unavailable`

## 5. 控制面接口结构

### 5.1 AgentRegister

职责：

- 建立 Agent 节点身份和元数据

请求结构：

| 字段 | 说明 |
| --- | --- |
| agentId | Agent 标识 |
| agentInstanceId | 当前实例标识 |
| nodeIdentity | 节点身份 |
| agentVersion | 版本 |
| hostMetadata | 主机信息 |
| capabilitySummary | 能力摘要 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| registrationStatus | 注册结果 |
| acceptedAgentId | Gateway 接受的 Agent 标识 |
| currentDesiredStateVersion | 当前版本 |
| controlSyncToken | 同步上下文 |

### 5.2 HeartbeatAndDesiredStateSync

职责：

- 接收心跳与状态摘要
- 返回最新目标状态和待执行命令

请求结构：

| 字段 | 说明 |
| --- | --- |
| agentId | Agent 标识 |
| currentDesiredStateVersion | 当前已应用版本 |
| networkState | 网络状态 |
| queueSummary | 本地队列摘要 |
| instanceSummary | 实例摘要 |
| resourceSummary | 资源摘要 |
| recentErrors | 最近错误 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| targetDesiredStateVersion | 目标版本 |
| desiredState | 最新目标状态 |
| pendingCommands | 命令集合 |
| backpressureHints | 背压建议 |
| serverTime | 服务器时间 |

说明：

- `DesiredStateSync` 与 `CommandPoll` 在第一版属于同一交互中的两个语义部分

`backpressureHints` 结构建议至少包含：

| 字段 | 说明 |
| --- | --- |
| gatewayPressureLevel | Gateway 当前压力级别 |
| suggestedUploadConcurrency | 建议上传并发 |
| suggestedRetryAfter | 建议重试间隔 |
| lowPriorityThrottleHint | 低优先级任务降速建议 |
| pauseRecommendation | 是否建议暂停可暂停任务 |

### 5.3 ReportCommandResult

职责：

- 上报命令执行结果

请求结构：

| 字段 | 说明 |
| --- | --- |
| commandId | 命令标识 |
| agentId | Agent 标识 |
| desiredStateVersion | 对应版本 |
| executionStatus | 执行结果 |
| resultDetail | 详情 |
| eventTime | 事件时间 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接收 |

### 5.4 TriggerRebalance

说明：

- 该接口仅在启用动态分发增强能力时开放
- 第一版基础闭环默认不对外开放该接口

职责：

- 触发任务级或分区级迁移编排

请求结构：

| 字段 | 说明 |
| --- | --- |
| taskId | 任务标识 |
| collectionPartitionId | 分区标识，可选 |
| sourceAgentId | 当前 owner Agent |
| targetAgentSelector | 目标 Agent 选择条件 |
| rebalanceReason | 迁移原因 |
| operatorIdentity | 操作人身份 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接受 |
| rebalanceStatus | 迁移状态 |
| targetDesiredStateVersion | 迁移目标版本 |

## 6. 数据面接口结构

### 6.1 UploadBatch

职责：

- 批量上传采集消息

请求结构：

| 字段 | 说明 |
| --- | --- |
| agentId | Agent 标识 |
| batchId | 批次标识 |
| compressionType | 压缩方式 |
| messageCount | 消息数量 |
| messages | 消息集合 |
| uploadContext | 上传上下文 |

`uploadContext` 结构建议至少包含：

| 字段 | 说明 |
| --- | --- |
| currentQueueDepth | Agent 当前本地队列深度 |
| localWatermarkState | Agent 当前水位状态 |
| uploadAttempt | 当前上传尝试次数 |
| networkState | Agent 当前网络状态 |
| desiredStateVersion | Agent 当前已应用目标版本 |
| firstMessageCollectTime | 本批次首条消息采集时间 |
| lastMessageCollectTime | 本批次末条消息采集时间 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否成功接收 |
| acceptedBatchId | 已确认批次 |
| backpressureState | 背压状态 |
| retryAfterHint | 建议重试时间 |
| gatewayAcceptTime | 接收时间 |

### 6.2 FailedMessageQuery

职责：

- 查询失败隔离区中的消息

请求结构：

| 字段 | 说明 |
| --- | --- |
| taskId | 任务标识 |
| agentId | Agent 标识 |
| failureScope | 查询范围 |
| transportMessageId | 传输层消息标识 |
| recordDedupKey | 业务去重键 |
| failureStage | 失败阶段 |
| timeRange | 时间范围 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| failedMessages | 失败消息集合，元素至少包含 `transportMessageId`、`recordDedupKey`、`taskId`、`agentId`、`connectorInstanceId`、`failureStage`、`failureReason`、`firstFailureTime`、`lastFailureTime`、`retryCount`、`rawPayloadOrReference`、`failureContext` |
| totalCount | 数量 |

### 6.3 ReportDrainStatus

说明：

- 该接口用于动态分发迁移中的 `drain` 进度上报
- 第一版基础闭环默认不启用该接口

职责：

- 上报迁移过程中旧 owner 的 `drain` 进度

请求结构：

| 字段 | 说明 |
| --- | --- |
| instanceId | 实例标识 |
| drainStatus | 当前 drain 状态 |
| lastCommittedCheckpoint | 最近可靠 checkpoint |
| pendingQueueDepth | 剩余本地积压 |
| drainEventTime | 状态时间 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否已记录 |
| nextActionHint | 后续动作建议 |

## 7. 错误与响应语义

### 7.0 通用错误结构

建议控制面与数据面错误统一使用如下结构：

| 字段 | 说明 |
| --- | --- |
| errorCode | 错误码 |
| errorCategory | 错误类别 |
| message | 错误摘要 |
| retryable | 是否可重试 |
| retryAfterHint | 建议重试时间 |
| correlationId | 关联请求或链路标识 |
| detail | 补充错误详情 |

### 7.1 控制面错误语义

建议错误类别：

- `InvalidIdentity`
- `ExpiredCommand`
- `StaleDesiredStateVersion`
- `UnsupportedCapability`
- `RejectedByPolicy`

建议错误码示例：

- `CP_INVALID_IDENTITY`
- `CP_EXPIRED_COMMAND`
- `CP_STALE_DESIRED_STATE`
- `CP_UNSUPPORTED_CAPABILITY`
- `CP_REJECTED_BY_POLICY`

### 7.2 数据面错误语义

建议错误类别：

- `IngressRejected`
- `BackpressureApplied`
- `TemporaryUnavailable`
- `InvalidBatch`
- `AuthenticationFailed`

建议错误码示例：

- `DP_INGRESS_REJECTED`
- `DP_BACKPRESSURE_APPLIED`
- `DP_TEMPORARY_UNAVAILABLE`
- `DP_INVALID_BATCH`
- `DP_AUTHENTICATION_FAILED`

### 7.3 Gateway 处理链失败分类

- `Transient Failure`
- `Data Quality Failure`
- `Routing / Storage Failure`

### 7.4 旧身份处理

- 旧 Agent 身份被标记失效后，Gateway 必须拒绝其后续注册、心跳、同步和上传请求

## 8. 与其他文档的关系

- Gateway 模块与流程设计：见 `02`
- Agent / Connector 本地运行与恢复设计：见 `03`
