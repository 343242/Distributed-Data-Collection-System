# Distributed Data Collection System Agent / Connector 详细设计

## 文档信息

- 文档名称：`Distributed Data Collection System Agent / Connector 详细设计`
- 文档版本：`v0.1`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

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

### 3.2 Agent 运行流程

#### 3.2.1 Agent 启动流程

1. 加载本地状态
2. 初始化本地队列与存储
3. 向 Gateway 注册
4. 拉取目标状态
5. 启动收敛流程

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
3. 本地队列持续积压
4. 恢复连接后先同步状态
5. 继续补传积压消息

### 3.3 Agent 本地可靠性设计

#### 3.3.1 本地原子提交

Agent 必须保证：

- 消息入队
- checkpoint 更新

属于同一原子提交单元。

处理原则：

- 成功则两者同时可见
- 失败则两者都不生效

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

#### 4.2.4 Health Reporter

- 向 Agent 上报存活、吞吐、错误、重连状态

### 4.3 Connector 运行模型

- 每个实例默认独立进程运行
- 每个实例拥有独立运行目录
- 每个实例拥有独立配置与日志上下文
- 同一程序包可同时运行多个实例

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

#### 4.5.3 ReportRuntimeStatus

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

- 默认采用受控切换
- 旧实例在一致性边界后停止
- 回滚通过切换到旧包版本重新启动

## 6. 恢复与异常处理

### 6.1 Agent 重启恢复

1. 恢复本地状态
2. 恢复未完成消息
3. 重新同步目标状态
4. 重建必要实例

### 6.2 Connector 异常

- 短时异常：实例内部有限重试
- 持续异常：Agent 单实例重启
- 超阈值：上报 Gateway 并进入异常态

### 6.3 磁盘与本地资源保护

- 日志轮转
- 队列容量上限
- 临界水位保护
- 可暂停任务优先暂停

## 7. 与其他文档的关系

以下内容不在本文档重复展开：

- Gateway 内部处理链：见 `02`
- Message 字段、状态机与公共接口结构：见 `04`
