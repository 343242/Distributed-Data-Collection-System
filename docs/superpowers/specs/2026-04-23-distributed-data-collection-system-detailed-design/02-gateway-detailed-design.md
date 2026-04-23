# Distributed Data Collection System Gateway 详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System Gateway 详细设计`
- 文档版本：`v0.1`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 1. 目的与范围

本文档详细说明 `Data Gateway` 的设计，覆盖以下内容：

- `Gateway Control Plane`
- `Gateway Data Plane`
- 功能职责
- 运行流程
- 控制流程
- 数据处理流程
- 失败处理与背压
- 接口职责、请求/响应结构和关键字段说明

本文档不展开：

- Agent 本地实现细节
- Connector 内部采集实现
- 完整 API 协议细节

## 2. Gateway 总体边界

Gateway 是系统的中心枢纽，但在详细设计中必须严格区分两类职责：

### 2.1 Gateway Control Plane

负责：

- 任务定义与状态管理
- 目标配置与版本推进
- Agent 注册与生命周期管理
- 程序包与配置管理
- 命令协调与下发
- 监控汇总与审计

### 2.2 Gateway Data Plane

负责：

- Agent 上传数据接入
- 接入鉴权与基础校验
- 接入持久化缓冲
- 标准化
- 幂等去重
- 路由
- 入库
- 失败隔离
- 背压反馈

## 3. Gateway Control Plane 详细设计

### 3.1 功能模块

#### 3.1.1 Task Management

职责：

- 维护任务定义
- 维护任务状态
- 维护 `desiredStateVersion`
- 根据 `triggerMode` 和配置生成目标运行状态

输入：

- 业务系统任务请求
- Agent 状态变化
- 程序包与配置变更

输出：

- 最新目标配置
- 任务状态变更
- 命令协调请求

#### 3.1.2 Agent Registry

职责：

- 管理 Agent 注册与失效
- 管理 Agent 当前状态、心跳、能力和负载
- 维护 Agent 与实例映射

#### 3.1.3 Package & Config Management

职责：

- 管理 `Connector Package` 元数据
- 管理包版本、配置模板、兼容关系
- 生成与发布目标配置

#### 3.1.4 Command Coordination

职责：

- 生成控制命令
- 与 `desiredStateVersion` 协同
- 维护命令状态与有效期
- 避免冲突命令

#### 3.1.5 Monitoring & Alerting

职责：

- 汇总 Agent / Task / Connector 运行指标
- 输出告警
- 支撑业务系统查看运行状态

#### 3.1.6 Audit Log

职责：

- 记录控制面关键操作
- 保证责任可追溯

### 3.2 控制面一致性策略

第一版 `Control Plane` 写路径采用：

- `单 Leader + 共享元数据存储`

规则如下：

- 只有 Leader 负责目标状态变更、命令生成和版本推进
- 非 Leader 可承担读流量和监控展示
- Leader 切换后由新 Leader 接管写路径

该策略用于降低多实例控制冲突风险。

### 3.3 控制面运行流程

#### 3.3.1 Agent 注册流程

1. Agent 携带节点身份发起注册
2. Gateway 校验身份和部署边界
3. 保存 Agent 元数据
4. 返回初始控制面同步信息

#### 3.3.2 心跳与状态同步流程

1. Agent 周期性发送心跳和摘要状态
2. Gateway 更新 Agent 状态
3. Gateway 返回最新 `desiredStateVersion` 与待处理命令

#### 3.3.3 任务配置变更流程

1. Business System 提交任务变更
2. Task Management 生成新的目标状态
3. `desiredStateVersion` 递增
4. Command Coordination 生成收敛命令
5. Agent 在下次同步中拉取到最新版本

### 3.4 控制面接口职责

#### 3.4.1 AgentRegister

职责：

- 建立 Agent 身份与元数据

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| agentId | Agent 逻辑标识 |
| agentInstanceId | 当前运行实例标识 |
| nodeIdentity | 节点身份凭据 |
| agentVersion | Agent 版本 |
| capabilitySummary | 能力摘要 |
| hostMetadata | 主机元数据 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| registrationStatus | 注册结果 |
| acceptedAgentId | 接受的 Agent 标识 |
| currentDesiredStateVersion | 当前目标状态版本 |
| controlSyncToken | 控制同步上下文 |

#### 3.4.2 HeartbeatAndDesiredStateSync

职责：

- 接收心跳和状态摘要
- 返回最新目标状态和待执行命令

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| agentId | Agent 标识 |
| currentDesiredStateVersion | Agent 当前已应用版本 |
| networkState | 网络状态 |
| queueSummary | 本地队列摘要 |
| instanceSummary | 实例状态摘要 |
| resourceSummary | 资源使用摘要 |
| recentErrors | 最近错误摘要 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| targetDesiredStateVersion | 最新目标状态版本 |
| desiredState | 最新目标配置 |
| pendingCommands | 待执行命令集合 |
| backpressureHints | 控制面补充建议 |
| serverTime | Gateway 时间 |

#### 3.4.3 ReportCommandResult

职责：

- 接收 Agent 对控制命令的执行结果

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| commandId | 命令标识 |
| agentId | Agent 标识 |
| desiredStateVersion | 命令关联版本 |
| executionStatus | 执行状态 |
| resultDetail | 结果详情 |
| eventTime | 执行时间 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否已记录 |
| nextActionHint | 后续动作提示 |

## 4. Gateway Data Plane 详细设计

### 4.1 功能模块

#### 4.1.1 Ingress Access

职责：

- 接收 Agent 上传批次
- 执行基础鉴权、协议校验和请求合法性检查

#### 4.1.2 Ingress Persistent Queue

职责：

- 将已接收批次可靠写入接入持久化缓冲
- 仅在写入成功后返回 `Accepted`

#### 4.1.3 Normalize / Transform

职责：

- 将不同 Connector 输出转换为统一内部处理模型

#### 4.1.4 Dedup / Idempotency

职责：

- 基于 `transportMessageId` 和 `recordDedupKey` 执行幂等去重

#### 4.1.5 Route

职责：

- 根据任务配置和数据类型选择目标存储路径

#### 4.1.6 Storage Writer

职责：

- 执行最终入库

#### 4.1.7 Failed Message Handling

职责：

- 记录无法正常完成处理的消息
- 支持后续排查与补偿

### 4.2 数据面运行流程

#### 4.2.1 上传接入流程

1. Agent 发送批量上传请求
2. Ingress Access 校验请求合法性
3. Ingress Persistent Queue 写入批次
4. 写入成功后返回 `Accepted`
5. Agent 将对应本地消息标记为已确认

#### 4.2.2 处理入库流程

1. Data Plane 从接入队列消费消息
2. 执行标准化
3. 执行去重
4. 执行路由
5. 执行入库
6. 标记处理完成或转入失败隔离区

### 4.3 数据面失败语义

失败分三类：

- `Transient Failure`
- `Data Quality Failure`
- `Routing / Storage Failure`

对应策略：

- 瞬时失败：有限重试与退避
- 数据质量失败：进入失败隔离区
- 路由或存储失败：保留重试，超阈值隔离

### 4.4 数据面背压

背压传播遵循：

`Storage Slowdown -> Processing Backlog -> Ingress Pressure -> Agent Upload Slowdown`

控制动作包括：

- 降低接入吞吐
- 延长上传重试间隔
- 返回背压反馈
- 对低优先级任务限流

### 4.5 数据面接口职责

#### 4.5.1 UploadBatch

职责：

- 接收 Agent 上传的消息批次

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| agentId | Agent 标识 |
| batchId | 批次标识 |
| compressionType | 压缩方式 |
| messageCount | 消息数量 |
| messages | 消息集合 |
| uploadContext | 上传上下文 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接收成功 |
| acceptedBatchId | 已接收批次标识 |
| backpressureState | 当前背压状态 |
| retryAfterHint | 建议重试间隔 |
| serverAcceptTime | Gateway 接收时间 |

#### 4.5.2 Failed Message Query

职责：

- 提供失败消息查看与排查入口

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| failureScope | 查询范围 |
| taskId | 任务标识 |
| agentId | Agent 标识 |
| failureStage | 失败阶段 |
| timeRange | 时间范围 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| failedMessages | 失败消息摘要集合 |
| totalCount | 总数 |

## 5. Gateway 运行与控制重点

### 5.1 Control Plane 优先保证一致性

- 目标状态变更串行化
- 旧命令可丢，目标状态不可乱

### 5.2 Data Plane 优先保证接入稳定性

- 先接住，再处理
- 接住后由 Gateway 承担后续责任

### 5.3 Gateway 与其他文档的关系

以下内容不在本文档重复展开：

- 具体消息字段定义：见 `04`
- Agent 本地原子提交：见 `03`
- Task / Instance 状态机：见 `04`
