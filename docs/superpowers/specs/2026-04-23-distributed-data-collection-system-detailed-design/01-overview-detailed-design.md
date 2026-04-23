# Distributed Data Collection System 详细设计总览

## 文档信息

- 文档名称：`Distributed Data Collection System 详细设计总览`
- 文档版本：`v0.1`
- 文档状态：`Draft`
- 创建日期：`2026-04-23`
- 最后更新：`2026-04-23`

## 1. 目的与范围

本文档是 `Distributed Data Collection System` 详细设计文档集的总览文档，用于说明详细设计的拆分方式、文档边界、统一约束、关键链路和建议实现顺序。

本文档不重复展开逻辑架构方案中的全部内容，而是承接以下上游文档：

- [Distributed Data Collection System 总体逻辑架构方案](../2026-04-22-distributed-data-collection-system-logical-architecture-design.md)

本轮详细设计文档重点覆盖：

- 功能设计
- 运行流程
- 控制流程
- 接口职责、请求/响应结构和关键字段说明

本轮详细设计文档暂不覆盖：

- 精确 API 契约
- 存储引擎选型
- 数据库表结构
- 部署脚本
- 运维 SOP

## 2. 文档清单与边界

本次详细设计按系统角色和公共模型拆分为四份文档：

### 2.1 01-overview-detailed-design.md

作用：

- 说明详细设计整体范围
- 串联后续三份文档
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

## 3. 详细设计与逻辑架构的映射

| 逻辑架构主题 | 详细设计文档 |
| --- | --- |
| Control Plane / Data Plane 边界 | 02 |
| Agent 宿主管理与 Connector 执行边界 | 03 |
| 消息模型、Checkpoint、状态机、接口 | 04 |
| 统一约束、落地顺序、文档关系 | 01 |

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

- 单客户单实例部署
- 客户间物理隔离
- 一个客户下可部署多个 Agent

### 5.2 安全边界

- `Agent <-> Gateway` 使用 `TLS`
- 节点身份认证优先采用 `mTLS`
- Connector 源端凭证不得写死在程序包中
- 运行日志不得明文记录敏感凭证

### 5.3 控制与数据分离

- Gateway 逻辑上拆分为 `Control Plane` 和 `Data Plane`
- 控制流与数据流接口分离
- Control Plane 不处理采集正文数据

### 5.4 通信模式

- 控制面采用“控制拉”
- 数据面采用“数据推”
- `desiredState` 是事实来源，命令是收敛信号

### 5.5 可靠性约束

- Agent 本地持久化消息队列是第一可靠落点
- `Checkpoint` 与消息入队必须在 Agent 本地原子提交
- Gateway 仅在接入持久化成功后返回 `Accepted`
- 系统采用“至少一次 + 幂等去重”

### 5.6 运行隔离约束

- 每个 Connector Instance 默认独立进程运行
- Agent 是本地宿主和 supervisor
- 配置变更默认通过实例级受控重启或受控切换生效

## 6. 统一设计原则

### 6.1 先目标状态，后执行命令

所有控制行为最终都收敛到 `desiredStateVersion`，而不是依赖历史命令重放。

### 6.2 先局部吸收波动，后逐级传播压力

背压遵循“先本层消化，超阈值再向上游传播”的原则。

### 6.3 先保存数据，再推进进度

任何 checkpoint 推进都必须以本地可靠持久化完成为前提。

### 6.4 允许少量重复，不允许无界丢失

通过 `transportMessageId` 和 `recordDedupKey` 两层标识实现可靠传输与业务去重。

### 6.5 单主控制，避免多写冲突

第一版 Gateway Control Plane 写路径采用单 Leader 协调，避免控制冲突。

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

建议实现顺序如下：

### Phase 1：控制与运行骨架

- Agent 注册
- 心跳与目标状态同步
- Task / Agent / Package 元数据管理
- Connector 独立进程托管

### Phase 2：可靠采集闭环

- Agent 本地持久化队列
- 本地原子提交
- 数据面上传与接收确认
- Gateway Ingress 幂等

### Phase 3：处理与稳定性

- 标准化、去重、路由、入库
- 失败隔离
- 背压反馈
- 离线恢复

### Phase 4：增强能力

- 程序包升级与回滚
- 监控与审计完善
- 动态分发

## 9. 阅读路径

建议按以下顺序阅读本轮详细设计：

1. `01-overview-detailed-design.md`
2. `04-message-state-interface-detailed-design.md`
3. `02-gateway-detailed-design.md`
4. `03-agent-connector-detailed-design.md`

推荐原因：

- 先建立总览和约束
- 再掌握公共模型、状态和接口
- 再分别进入 Gateway 和 Agent / Connector 设计

## 10. 后续深化方向

本轮详细设计完成后，下一步建议继续输出：

- API 契约草案
- 数据库存储设计
- 部署拓扑设计
- 关键时序图
- 运维与故障处理手册
