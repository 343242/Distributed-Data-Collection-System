# Distributed Data Collection System Gateway 详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System Gateway 详细设计`
- 文档版本：`v0.3`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 修订历史

| 版本 | 日期 | 修订摘要 |
| --- | --- | --- |
| v0.1 | 2026-04-23 | 建立 Gateway 详细设计骨架，明确 Control Plane / Data Plane 边界、核心流程和接口结构 |
| v0.2 | 2026-04-23 | 根据审阅意见补充程序包与配置管理、监控审计、Leader 协调、异步接入解耦、背压传播链路、失败隔离字段与业务侧接口 |
| v0.3 | 2026-04-23 | 交叉一致性复核，统一与公共接口文档的共享字段命名 |

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

关键元数据：

| 字段 | 说明 |
| --- | --- |
| connectorType | Connector 类型 |
| packageVersion | 程序包版本 |
| packageUri | 包下载定位 |
| checksum | 完整性校验值 |
| packageSize | 程序包大小 |
| configSchemaVersion | 配置 schema 版本 |
| runtimeCompatibility | 运行兼容性说明 |

运行要求：

- Agent 按目标配置主动拉取所需包版本
- 包下载完成后必须先校验 `checksum`，再允许安装与实例启动
- Gateway 必须校验 `packageVersion` 与 `configSchemaVersion` 的兼容性
- 同一 `connectorType` 的多个版本允许短期共存，以支持受控切换与回滚
- 升级默认采用“新实例准备完成 -> 切换目标归属 -> 停止旧实例”的受控切换模式
- 回滚本质上是切换回上一稳定版本的目标配置

输入：

- Business System 发布的新包版本
- 配置模板变更
- Agent 上报的包校验或兼容性结果

输出：

- 可发布的 `PackageMetadata`
- 配置兼容性校验结果
- 面向 Task Management 的目标配置约束

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

监控粒度：

- 系统级
- Agent 级
- Task 级
- `Connector Instance` 级

关键指标：

- Agent 在线率与心跳延迟
- Agent 本地队列深度与磁盘水位
- `desiredStateVersion` 收敛延迟
- Data Plane Ingress Queue 深度
- Data Plane 处理延迟与失败率
- Storage Writer 耗时与失败率
- 被限流任务数、被暂停实例数与背压持续时间

聚合方式：

- Agent 周期性通过 `HeartbeatAndDesiredStateSync` 上报轻量摘要
- Data Plane 在接入、处理和失败隔离链路中生成内部运行指标
- Monitoring & Alerting 负责按系统 / Agent / Task / Instance 四层聚合并输出告警

#### 3.1.6 Audit Log

职责：

- 记录控制面关键操作
- 保证责任可追溯

审计范围：

- 任务创建、变更、启停、删除
- 目标状态版本推进
- 程序包发布、升级、回滚
- Agent 注册、失效、替换
- 命令生成与下发
- 动态分发 / 迁移

最小记录结构：

| 字段 | 说明 |
| --- | --- |
| actionType | 操作类型 |
| actionTarget | 操作对象 |
| initiator | 发起者 |
| actionTime | 发起时间 |
| targetVersion | 目标版本 |
| executionResult | 执行结果 |

### 3.2 控制面一致性策略

第一版 `Control Plane` 写路径采用：

- `单 Leader + 共享元数据存储`

规则如下：

- 只有 Leader 负责目标状态变更、命令生成和版本推进
- 非 Leader 可承担读流量和监控展示
- Leader 切换后由新 Leader 接管写路径

细化约束：

- 第一版不在详细设计中展开复杂分布式锁实现，默认以共享元数据存储中的 Leader 身份标识和租约语义作为写路径入口控制
- 非 Leader 接收到写请求时，应转发到 Leader 或返回可重试重定向结果，不得自行推进 `desiredStateVersion`
- Leader 切换后，新 Leader 必须先从共享元数据存储恢复任务状态、命令状态和版本上下文，再继续生成写操作
- Leader 切换期间允许读流量继续服务，但写路径应短暂串行保护，避免并发双写

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

#### 3.3.4 程序包发布与受控升级流程

1. Business System 发布新的 `Connector Package` 版本元数据
2. Package & Config Management 校验元数据完整性与配置兼容性
3. Task Management 生成引用新版本的目标配置
4. `desiredStateVersion` 递增并生成收敛命令
5. Agent 在同步后拉取新目标配置并主动下载程序包
6. Agent 完成下载、校验和本地安装后，按实例级受控切换方式启动新实例
7. 旧实例完成一致性提交边界后停止
8. 若新版本启动失败或运行异常，Gateway 可切换回上一稳定版本目标配置

#### 3.3.5 监控与审计汇聚流程

1. Agent 周期性上报心跳、运行摘要和关键错误摘要
2. Data Plane 内部上报接入、处理、失败和背压指标
3. Monitoring & Alerting 按四层粒度聚合指标并判断告警
4. Audit Log 对关键控制面操作写入审计记录
5. Business System 查询运行视图或审计记录时，统一从 Gateway 侧读取

### 3.4 控制面接口职责

#### 3.4.1 UpsertTaskDefinition

职责：

- 创建或更新任务定义

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| taskId | 任务标识 |
| taskDefinition | 任务定义主体 |
| operatorIdentity | 操作人身份 |
| requestTime | 请求时间 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接受 |
| targetDesiredStateVersion | 生成后的目标版本 |
| taskStatus | 任务状态 |

#### 3.4.2 ChangeTaskLifecycle

职责：

- 触发任务启用、暂停、恢复、停止或删除

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| taskId | 任务标识 |
| lifecycleAction | 生命周期动作 |
| operatorIdentity | 操作人身份 |
| actionReason | 操作原因 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接受 |
| targetDesiredStateVersion | 目标版本 |
| taskStatus | 最新任务状态 |

#### 3.4.3 PublishPackageVersion

职责：

- 发布新的程序包版本元数据并触发兼容性校验

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| connectorType | Connector 类型 |
| packageMetadata | 程序包元数据 |
| operatorIdentity | 操作人身份 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接受 |
| compatibilityStatus | 兼容性校验结果 |
| releaseStatus | 发布状态 |

#### 3.4.4 QueryGatewayRuntimeSummary

职责：

- 提供 Gateway 侧运行态、告警与审计查询入口

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| queryScope | 查询范围 |
| taskId | 任务标识 |
| agentId | Agent 标识 |
| includeAudit | 是否包含审计摘要 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| runtimeSummary | 运行摘要 |
| activeAlerts | 当前告警 |
| auditSummary | 审计摘要 |

#### 3.4.5 AgentRegister

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

#### 3.4.6 HeartbeatAndDesiredStateSync

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
| backpressureHints | Data Plane 反馈的背压建议 |
| serverTime | Gateway 时间 |

`backpressureHints` 建议至少包含：

| 字段 | 说明 |
| --- | --- |
| gatewayPressureLevel | Gateway 当前压力级别 |
| suggestedUploadConcurrency | 建议上传并发 |
| suggestedRetryAfter | 建议重试间隔 |
| lowPriorityThrottleHint | 低优先级任务降速建议 |
| pauseRecommendation | 是否建议暂停可暂停任务 |

#### 3.4.7 ReportCommandResult

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
- 作为 Data Plane 接入确认边界，与后续处理链异步解耦

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

最小记录字段：

| 字段 | 说明 |
| --- | --- |
| transportMessageId | 传输层消息标识 |
| recordDedupKey | 业务去重键 |
| taskId | 任务标识 |
| agentId | Agent 标识 |
| connectorInstanceId | 实例标识 |
| failureStage | 失败阶段 |
| failureReason | 失败原因 |
| firstFailureTime | 首次失败时间 |
| lastFailureTime | 最近失败时间 |
| retryCount | 已重试次数 |
| rawPayloadOrReference | 原始载荷或其引用 |
| failureContext | 补充失败上下文 |

### 4.2 数据面运行流程

#### 4.2.1 上传接入流程

1. Agent 发送批量上传请求
2. Ingress Access 执行身份校验、协议校验和请求合法性检查
3. Ingress Persistent Queue 写入批次
4. 写入成功后返回 `Accepted`
5. Agent 将对应本地消息标记为已确认
6. 后续处理链异步消费该批次，不阻塞接入确认

#### 4.2.2 处理入库流程

1. Data Plane 从 Ingress Persistent Queue 异步消费消息
2. 校验消费批次的完整性和上下文
3. 执行标准化与格式转换
4. 执行基于 `transportMessageId` 的接入层幂等判断
5. 执行基于 `recordDedupKey` 的业务层去重
6. 执行路由决策
7. 调用 Storage Writer 完成入库
8. 记录处理结果和时间语义
9. 对失败消息执行重试、退避或隔离
10. 标记处理完成或转入失败隔离区

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

`Storage Slowdown -> Processing Backlog -> Ingress Pressure -> Agent Upload Slowdown -> Agent Local Queue Growth -> Connector Throttle/Pause`

传播机制：

1. `Storage Writer` 耗时和失败率升高时，首先在 Data Plane 内部形成 `Processing Backlog`
2. 当处理积压超过高水位后，Ingress Queue 增长并进入 `Ingress Pressure`
3. Ingress Pressure 达到阈值后，`UploadBatch` 响应返回 `backpressureState` 和 `retryAfterHint`
4. Control Plane 在后续 `HeartbeatAndDesiredStateSync` 中返回 `backpressureHints`，向 Agent 提供更稳定的控制面降载建议
5. Agent 根据上传反馈和控制面建议降低上传并发、延长重试间隔、暂停低优先级任务
6. 当 Agent 本地队列继续增长至高水位或临界水位时，由 Agent 对 Connector 执行 `Throttle / Pause`

阈值语义：

- `Normal`：Gateway 内部可自行吸收波动，不向 Agent 传播压力
- `High`：Gateway 开始限制上传速率，并给出重试与降速建议
- `Critical`：Gateway 明确建议暂停可暂停任务，优先保护整体稳定性

控制动作包括：

- 降低接入吞吐
- 延长上传重试间隔
- 返回背压反馈
- 对低优先级任务限流
- 通过 `HeartbeatAndDesiredStateSync.backpressureHints` 返回任务级降速或暂停建议

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

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否接收成功 |
| acceptedBatchId | 已接收批次标识 |
| backpressureState | 当前背压状态 |
| retryAfterHint | 建议重试间隔 |
| gatewayAcceptTime | Gateway 接收时间 |

#### 4.5.2 Failed Message Query

职责：

- 提供失败消息查看与排查入口

请求结构示意：

| 字段 | 说明 |
| --- | --- |
| failureScope | 查询范围 |
| taskId | 任务标识 |
| agentId | Agent 标识 |
| transportMessageId | 传输层消息标识 |
| recordDedupKey | 业务去重键 |
| failureStage | 失败阶段 |
| timeRange | 时间范围 |

响应结构示意：

| 字段 | 说明 |
| --- | --- |
| failedMessages | 失败消息集合，元素至少包含 `transportMessageId`、`recordDedupKey`、`taskId`、`agentId`、`connectorInstanceId`、`failureStage`、`failureReason`、`firstFailureTime`、`lastFailureTime`、`retryCount`、`rawPayloadOrReference`、`failureContext` |
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
