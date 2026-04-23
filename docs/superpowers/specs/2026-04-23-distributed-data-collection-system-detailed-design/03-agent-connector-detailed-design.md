# Distributed Data Collection System Agent / Connector 详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System Agent / Connector 详细设计`
- 文档版本：`v0.4`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 修订历史

| 版本 | 日期 | 修订摘要 |
| --- | --- | --- |
| v0.1 | 2026-04-23 | 建立 Agent / Connector 详细设计骨架，明确模块划分、生命周期和基本运行流程 |
| v0.2 | 2026-04-23 | 根据审阅意见补充原子提交实现约束、Checkpoint 恢复上下文、离线与水位联动、注册失效处理、资源限制来源、升级回滚与恢复失败路径 |
| v0.3 | 2026-04-23 | 补充动态分发场景下的 Agent drain、checkpoint 交接和实例迁移执行语义 |
| v0.4 | 2026-04-23 | 将动态分发执行流程标记为增强能力预留，并修正文档内交互小节的编号顺序 |

## 1. 目的与范围

本文档详细说明 `Client Agent` 和 `Connector` 的设计，覆盖：

- Agent 内部模块
- Connector 运行模型
- 程序包与实例管理
- 本地可靠队列与 checkpoint 原子提交
- 离线与恢复
- 关键接口职责、请求/响应结构和关键字段说明

## 2. Agent 总体定位

Agent 是客户现场的本地宿主与管理控制器，不是简单命令转发器。Agent 的责任包括：

- 按目标配置托管 Connector 实例
- 负责包下载、校验、安装和版本切换
- 负责本地可靠落盘与上传
- 负责实例生命周期管理
- 负责监控、恢复和本地资源保护

## 3. Agent 详细设计

### 3.1 内部模块

#### 3.1.1 Command Handler

职责：

- 接收 Control Plane 返回的 `desiredState`
- 解析 `pendingCommands`
- 触发本地收敛流程

#### 3.1.2 Desired State Reconciler

职责：

- 对比目标状态与当前状态
- 决定需要新增、停止、重启、升级哪些实例
- 忽略过期命令

#### 3.1.3 Package Manager

职责：

- 拉取程序包
- 校验完整性
- 管理本地多版本共存
- 支持受控清理旧包

包清理约束：

- 仅当旧包不再被任何运行实例引用时才允许清理
- 当前目标版本和上一稳定回滚版本默认保留
- 磁盘压力触发清理时，优先清理无引用且不在升级/回滚窗口内的旧版本
- 不允许在实例受控切换尚未完成时清理对应程序包

#### 3.1.4 Connector Manager

职责：

- 创建实例运行目录
- 注入配置和凭证
- 启动、停止、重启、销毁 Connector 实例

#### 3.1.5 Local Persistent Queue

职责：

- 持久化采集消息
- 支持积压
- 支持重试上传
- 管理本地消息状态

#### 3.1.6 Local State Store

职责：

- 保存任务状态
- 保存实例状态
- 保存已提交 checkpoint
- 保存命令执行记录

#### 3.1.7 Runtime Monitor

职责：

- 监控实例存活、资源、水位和错误
- 产生状态摘要和告警

#### 3.1.8 Recovery Manager

职责：

- Agent 重启后恢复本地状态
- 网络恢复后补传积压数据
- 重新收敛到最新目标状态
- 恢复失败时进入保守恢复路径并上报 Gateway

### 3.2 Agent 运行流程

#### 3.2.1 Agent 启动流程

1. 加载本地状态与上次已提交 checkpoint
2. 初始化本地队列、状态存储和运行监控
3. 携带节点身份与主机元数据向 Gateway 发起注册
4. Gateway 校验身份、部署边界以及旧身份失效状态
5. Agent 上报基础信息、能力摘要与运行实例标识
6. Gateway 建立或更新 Agent 元数据
7. Agent 进入 `HeartbeatAndDesiredStateSync`
8. 拉取目标状态并启动本地收敛流程

约束：

- 若旧身份已被 Gateway 标记失效，则旧身份的注册、心跳、同步和上传请求必须被拒绝
- Agent 替换后应以新的 `agentInstanceId` 重新加入，不复用已失效运行身份

#### 3.2.2 目标状态收敛流程

1. Agent 获取最新 `desiredStateVersion`
2. 对比本地已应用版本
3. 生成本地动作集合
4. 顺序执行程序包、配置、实例管理动作
5. 回报结果

#### 3.2.3 采集与上传流程

1. Connector 采到数据
2. Agent 执行本地原子提交
3. 消息进入本地持久化队列
4. Upload Worker 批量发送到 Gateway
5. 收到 `Accepted` 后将消息标记完成

#### 3.2.4 离线与恢复流程

1. 与 Gateway 断链
2. 根据 `offlinePolicy` 决定继续采、暂停或按水位运行
3. 本地队列持续积压并按水位执行保护动作
4. 恢复连接后先同步状态
5. 继续补传积压消息

联动规则：

- `ContinueWhenDisconnected`
  - 在 `Low Watermark` 和 `High Watermark` 下持续采集
  - 到达 `Critical Watermark` 时，必须触发保护动作，避免磁盘耗尽
- `PauseWhenDisconnected`
  - 一旦确认进入断链状态，暂停可暂停任务，不再继续拉取新数据
- `ContinueUntilWatermark`
  - 默认继续采集直到达到 `High Watermark`
  - 达到 `High Watermark` 后执行降速
  - 达到 `Critical Watermark` 后暂停可暂停任务

与 `backpressurePolicy` 的交互：

- `BestEffort`：在 `High Watermark` 即优先降速或暂停
- `PreferContinuity`：尽量维持采集直到接近 `Critical Watermark`
- `StrictPauseOnPressure`：一旦进入高水位或收到强背压建议即暂停

水位配置原则：

- 水位阈值由 Gateway 目标配置下发为主
- Agent 可提供保守默认值作为兜底
- 实际保护动作由 Agent 基于任务类型和本地磁盘状态执行

#### 3.2.5 动态分发 / 迁移执行流程

说明：

- 本节描述的是增强能力预留，不属于第一版核心交付范围
- 第一版若未启用动态分发，则 Agent 不进入该执行路径

1. Agent 从 Gateway 拉取到 `Rebalancing` 状态的目标配置
2. 若当前 Agent 是旧 owner，则进入 `drain` 模式
3. 旧 owner 停止继续拉取新数据，但继续完成本地已采集数据的原子提交与上传
4. 旧 owner 到达可交接边界后，上报最后可靠 `checkpoint` 与 `drainStatus`
5. 若当前 Agent 是新 owner，则根据 `startCheckpoint` 准备启动上下文
6. 新 owner 启动 Connector 实例并等待 `readySignal`
7. Gateway 确认 owner 切换后，旧 owner 停止旧实例并清理运行上下文
8. 新 owner 进入正常采集状态

执行约束：

- 旧 owner 在 `drain` 阶段不得继续拉取新数据
- 新 owner 不得在收到 `startCheckpoint` 之前开始采集
- 若切换失败，旧 owner 优先保持可恢复运行，避免任务悬空

### 3.3 Agent 本地可靠性设计

#### 3.3.1 本地原子提交

Agent 必须保证：

- 消息入队
- checkpoint 更新

属于同一原子提交单元。

处理原则：

- 成功则两者同时可见
- 失败则两者都不生效

最低实现要求：

- Agent 本地存储引擎必须具备事务能力，或提供等价的崩溃一致性原子提交语义
- 第一版不接受无法保证该语义的纯无事务持久化方案

第一版可接受实现方式示例：

- 基于同一本地存储引擎的单事务提交
- 基于 `WAL / journaling` 的崩溃恢复一致提交
- 基于影子写入与原子切换的等价提交语义

恢复语义：

- 若提交前崩溃，则消息与 checkpoint 都视为未生效，恢复后允许重采
- 若提交后崩溃，则消息与 checkpoint 都必须可恢复，恢复后继续补传

#### 3.3.2 本地消息状态

建议本地队列中消息至少具备以下状态：

- `Pending`
- `Sending`
- `Acked`
- `FailedLocal`

#### 3.3.3 本地水位与保护

本地队列应具备：

- `Low Watermark`
- `High Watermark`
- `Critical Watermark`

并触发：

- 告警
- 降速
- 暂停
- 磁盘保护

### 3.4 Agent 接口职责

#### 3.4.1 Control Sync Handler

职责：

- 处理来自 Gateway 的目标状态同步结果

输入结构示意：

| 字段 | 说明 |
| --- | --- |
| targetDesiredStateVersion | 目标版本 |
| desiredState | 目标配置 |
| pendingCommands | 待执行命令 |

输出结构示意：

| 字段 | 说明 |
| --- | --- |
| reconcilePlan | 本地收敛计划 |
| skippedCommands | 被忽略命令 |

#### 3.4.2 Upload Worker

职责：

- 从本地队列取消息并批量上传

输入结构示意：

| 字段 | 说明 |
| --- | --- |
| pendingMessages | 待上传消息 |
| backpressureState | 当前背压状态 |

输出结构示意：

| 字段 | 说明 |
| --- | --- |
| acceptedMessages | 已确认消息 |
| retryMessages | 需要重试消息 |
| throttleAction | 节流动作 |

## 4. Connector 详细设计

### 4.1 Connector 总体定位

Connector 是特定源类型的采集执行插件，只负责：

- 连接源端
- 执行采集
- 生成候选 checkpoint
- 输出统一采集消息
- 上报实例运行状态

### 4.2 Connector 内部职责

#### 4.2.1 Source Adapter

- 建立源端连接
- 处理认证
- 处理协议细节

#### 4.2.2 Collector Runtime

- 执行轮询、订阅或增量读取

#### 4.2.3 Checkpoint Producer

- 生成候选 checkpoint
- 不直接提交最终 checkpoint

候选 checkpoint 至少应包含：

| 字段 | 说明 |
| --- | --- |
| checkpointValue | 位点值 |
| checkpointType | 位点类型 |
| sourceContext | 源端恢复上下文 |
| sourcePartitionIdentity | 源分区、文件或分片标识 |
| sourceCursorContext | cursor / offset / 文件位置等上下文 |
| candidateCapturedAt | 候选位点产生时间 |

说明：

- `sourceContext` 必须足以支撑重启、恢复或迁移后的继续采集
- 对文件尾随、CDC、游标式接口等场景，不允许只保存单一偏移值

#### 4.2.4 Health Reporter

- 向 Agent 上报存活、吞吐、错误、重连状态

### 4.3 Connector 运行模型

- 每个实例默认独立进程运行
- 每个实例拥有独立运行目录
- 每个实例拥有独立配置与日志上下文
- 同一程序包可同时运行多个实例

资源限制来源：

- `runtimeLimits` 以 Gateway 目标配置下发为主
- Agent 在缺少显式配置时可应用保守默认值
- 资源限制至少应覆盖 CPU、内存、临时目录、文件句柄或连接数等维度

超限处理原则：

- 首次超限记录告警并上报 Gateway
- 持续超限可触发实例降级、受控重启或暂停
- 不允许单实例长期无界消耗资源并影响同机其他实例

### 4.4 Connector 生命周期

主要阶段：

- `Created`
- `Starting`
- `Running`
- `Degraded`
- `Retrying`
- `Stopped`
- `Failed`
- `Destroyed`

Agent 负责生命周期驱动，Connector 负责状态上报。

常见触发事件示意：

- `Created -> Starting`：Agent 完成运行上下文准备并发起启动
- `Starting -> Running`：实例返回 `readySignal`
- `Running -> Degraded`：连续错误、源端限流或本地积压升高
- `Degraded -> Retrying`：实例进入短时自恢复
- `Retrying -> Running`：恢复成功
- `Retrying -> Failed`：恢复超过阈值仍失败
- `Running / Degraded -> Stopped`：Agent 触发受控停止
- `Stopped / Failed -> Destroyed`：实例被清理

### 4.5 Connector 与 Agent 的交互

#### 4.5.1 StartConnector

职责：

- Agent 根据本地准备好的运行上下文启动实例

输入结构示意：

| 字段 | 说明 |
| --- | --- |
| instanceId | 实例标识 |
| packageVersion | 程序包版本 |
| configPath | 配置路径 |
| credentialRef | 凭证引用 |
| runtimeLimits | 运行限制 |
| startCheckpoint | 启动时使用的 checkpoint，可用于恢复或迁移接管 |
| ownershipMode | 当前实例是正常启动、恢复启动还是迁移接管启动 |

输出结构示意：

| 字段 | 说明 |
| --- | --- |
| startStatus | 启动结果 |
| readySignal | 就绪状态 |
| failureReason | 失败原因 |

#### 4.5.2 SubmitCollectedRecord

职责：

- Connector 将采集结果与候选 checkpoint 交给 Agent

输入结构示意：

| 字段 | 说明 |
| --- | --- |
| sourceRecord | 源记录或批次 |
| candidateCheckpoint | 候选 checkpoint |
| collectContext | 采集上下文 |

输出结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否被 Agent 接收 |
| committedCheckpoint | 已提交 checkpoint |

迁移 / 恢复约束：

- 在动态分发或异常恢复场景下，Connector 的首次采集必须基于 `StartConnector.startCheckpoint` 初始化
- `SubmitCollectedRecord` 只提交当前采集产生的候选 checkpoint，不负责跨实例交接语义

#### 4.5.3 ReportDrainStatus

职责：

- 旧 owner 在迁移过程中向 Agent / Gateway 上报实例 `drain` 进展

输入结构示意：

| 字段 | 说明 |
| --- | --- |
| instanceId | 实例标识 |
| drainStatus | 当前 drain 状态 |
| lastCommittedCheckpoint | 最近可靠 checkpoint |
| pendingQueueDepth | 剩余本地积压 |
| drainEventTime | 状态时间 |

输出结构示意：

| 字段 | 说明 |
| --- | --- |
| accepted | 是否已记录 |
| nextActionHint | 后续动作建议 |

#### 4.5.4 ReportRuntimeStatus

职责：

- 上报实例状态与健康信息

输入结构示意：

| 字段 | 说明 |
| --- | --- |
| instanceId | 实例标识 |
| runtimeState | 运行状态 |
| throughput | 吞吐摘要 |
| errorSummary | 错误摘要 |
| resourceUsage | 资源使用摘要 |

## 5. 程序包与实例管理

### 5.1 程序包获取

1. Agent 拉取目标版本元数据
2. 若本地缺失，则主动下载
3. 执行 checksum 校验
4. 校验通过后安装并登记

### 5.2 实例创建

1. 创建运行目录
2. 写入配置
3. 注入凭证引用
4. 写入资源限制
5. 启动实例

### 5.3 升级与回滚

默认采用受控切换。

升级流程：

1. Agent 拉取新版本元数据并完成下载校验
2. 保留旧版本实例继续运行，准备新实例运行上下文
3. 以最近已提交 checkpoint 作为新实例启动基线
4. 启动新实例并等待 `readySignal`
5. 新实例就绪后，在一致性提交边界上切换任务执行归属
6. 旧实例完成当前提交边界后停止

回滚流程：

1. 若新实例启动失败或进入持续异常态，则保持旧实例继续运行或重新拉起旧版本实例
2. Gateway 切换回上一稳定版本目标配置
3. Agent 以最后稳定 checkpoint 启动旧版本实例
4. 新版本实例退出并保留诊断信息

一致性边界定义：

- 至少要求当前批次消息已完成本地原子提交
- 不要求等待全部消息上传完成后才切换
- 切换过程允许少量重复，不允许因 checkpoint 前推导致丢数

## 6. 恢复与异常处理

### 6.1 Agent 重启恢复

1. 恢复本地状态
2. 恢复未完成消息
3. 重新同步目标状态
4. 重建必要实例

若恢复过程中出现本地状态损坏或 checkpoint 不一致：

- Agent 进入保守恢复模式
- 优先保留本地队列中的未确认消息
- 将受影响实例标记为 `Degraded`
- 上报 Gateway，请求人工排查或重新收敛

### 6.2 Connector 异常

- 短时异常：实例内部有限重试
- 持续异常：Agent 单实例重启
- 超阈值：上报 Gateway 并进入异常态

若 Recovery Manager 自恢复失败：

- Agent 记录恢复失败原因
- 停止继续自动重复拉起同一异常流程
- 将对应任务或实例置为 `Degraded` 或 `Failed`
- 在下一次状态同步中显式上报 Gateway

### 6.4 迁移失败处理

- 新 owner 未在预期时间内返回 `readySignal` 时，旧 owner 不应被提前销毁
- 若旧 owner `drain` 失败，则 Gateway 应取消本轮迁移并恢复原目标状态
- 若新旧 owner 都不可用，则任务维持 `Rebalancing` 或进入 `Failed`，等待人工处理

### 6.3 磁盘与本地资源保护

- 日志轮转
- 队列容量上限
- 临界水位保护
- 可暂停任务优先暂停

## 7. 与其他文档的关系

以下内容不在本文档重复展开：

- Gateway 内部处理链：见 `02`
- Message 字段、状态机与公共接口结构：见 `04`
