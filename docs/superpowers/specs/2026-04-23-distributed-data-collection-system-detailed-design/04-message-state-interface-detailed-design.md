# Distributed Data Collection System 消息、状态机与接口详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System 消息、状态机与接口详细设计`
- 文档版本：`v0.1`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

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
- `transportMessageId` 推荐包含 `agentId` 前缀

## 3. 时间语义

系统定义以下时间语义：

| 字段 | 说明 | 产生方 |
| --- | --- | --- |
| sourceEventTime | 源数据产生时间 | 源端或 Connector |
| collectTime | Connector 采到数据时间 | Connector |
| agentPersistTime | Agent 本地持久化成功时间 | Agent |
| gatewayAcceptTime | Gateway 接收成功时间 | Gateway Data Plane |
| processedTime | Gateway 完成处理时间 | Gateway Data Plane |
| storedTime | 数据入库存储时间 | Storage Writer |
| commandIssuedTime | 命令生成时间 | Gateway Control Plane |
| commandExpireTime | 命令失效时间 | Gateway Control Plane |

原则：

- 各节点需具备基础时间同步能力，例如 `NTP`
- 不把全局时间戳排序作为唯一一致性依据

## 4. 状态机设计

### 4.1 Task 状态机

建议状态：

- `Draft`
- `Scheduled`
- `Deploying`
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
| Running | 人工暂停 | Paused |
| Paused | 恢复 | Running |
| Running | 自动恢复失败 | Failed |
| Failed | 重新部署 | Deploying |
| Running / Paused / Failed | 停止 | Stopped |

### 4.2 Connector Instance 状态机

建议状态：

- `Created`
- `Starting`
- `Running`
- `Degraded`
- `Retrying`
- `Stopped`
- `Failed`
- `Destroyed`

### 4.3 Agent 网络状态机

建议状态：

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
| serverTime | 服务器时间 |

说明：

- `DesiredStateSync` 与 `CommandPoll` 在第一版属于同一交互中的两个语义部分

### 5.3 ReportCommandResult

职责：

- 上报命令执行结果

请求结构：

| 字段 | 说明 |
| --- | --- |
| commandId | 命令标识 |
| desiredStateVersion | 对应版本 |
| executionStatus | 执行结果 |
| resultDetail | 详情 |
| eventTime | 事件时间 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接收 |

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
| failureStage | 失败阶段 |
| timeRange | 时间范围 |

响应结构：

| 字段 | 说明 |
| --- | --- |
| failedMessages | 失败消息摘要 |
| totalCount | 数量 |

## 7. 错误与响应语义

### 7.1 控制面错误语义

建议错误类别：

- `InvalidIdentity`
- `ExpiredCommand`
- `StaleDesiredStateVersion`
- `UnsupportedCapability`
- `RejectedByPolicy`

### 7.2 数据面错误语义

建议错误类别：

- `IngressRejected`
- `BackpressureApplied`
- `TemporaryUnavailable`
- `InvalidBatch`
- `AuthenticationFailed`

### 7.3 Gateway 处理链失败分类

- `Transient Failure`
- `Data Quality Failure`
- `Routing / Storage Failure`

### 7.4 旧身份处理

- 旧 Agent 身份被标记失效后，Gateway 必须拒绝其后续注册、心跳、同步和上传请求

## 8. 与其他文档的关系

- Gateway 模块与流程设计：见 `02`
- Agent / Connector 本地运行与恢复设计：见 `03`
