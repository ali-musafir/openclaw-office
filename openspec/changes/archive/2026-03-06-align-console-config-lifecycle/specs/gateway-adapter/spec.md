## MODIFIED Requirements

### Requirement: GatewayAdapter 统一接口

系统 SHALL 定义 `GatewayAdapter` TypeScript 接口，包含与 OpenClaw Gateway 方法一一对应的类型安全方法签名，并覆盖控制台配置生命周期所需的原生 Gateway 能力。

#### Scenario: 接口方法覆盖控制台配置生命周期

- **WHEN** GatewayAdapter 接口用于支持控制台配置管理
- **THEN** 接口 SHALL 在现有领域方法之外补充以下能力：
  - 配置读取：`configGet()`, `configSchema()`
  - 配置保存：`configSet()`
  - 配置应用：`configApply()`
  - 更新执行：`updateRun()`
- **THEN** 接口 SHALL 保留现有运行时修改方法，例如 `agentsUpdate()`, `skillsUpdate()`, `cronAdd()`, `cronUpdate()`, `cronRemove()`, `cronRun()`

#### Scenario: 类型区分 save 与 apply

- **WHEN** 调用 GatewayAdapter 的配置写入方法
- **THEN** `configSet()` 与 `configApply()` SHALL 作为语义不同的独立方法暴露
- **THEN** 调用方 SHALL 能区分“仅保存”和“保存并重启”两种行为

### Requirement: WsAdapter 实现

系统 SHALL 提供 `WsAdapter` 类，实现 `GatewayAdapter` 接口，通过现有 WebSocket 客户端与真实 Gateway 通信。

#### Scenario: WsAdapter 映射 config.set

- **WHEN** 调用 WsAdapter 的 `configSet(raw, baseHash)`
- **THEN** SHALL 内部调用 Gateway RPC `config.set`
- **THEN** SHALL 返回类型化结果供控制台判断保存是否成功

#### Scenario: WsAdapter 映射 config.apply

- **WHEN** 调用 WsAdapter 的 `configApply(raw, baseHash, params)`
- **THEN** SHALL 内部调用 Gateway RPC `config.apply`
- **THEN** SHALL 支持透传 `sessionKey`、`note`、`restartDelayMs` 等原生参数

### Requirement: MockAdapter 实现

系统 SHALL 提供 `MockAdapter` 类，实现 `GatewayAdapter` 接口，返回模拟数据用于离线开发。

#### Scenario: MockAdapter 支持 save/apply 生命周期模拟

- **WHEN** 调用 MockAdapter 的 `configSet()`
- **THEN** SHALL 返回“保存成功且未强制安排重启”的模拟结果

#### Scenario: MockAdapter 支持 apply 生命周期模拟

- **WHEN** 调用 MockAdapter 的 `configApply()`
- **THEN** SHALL 返回“保存成功且已安排重启”的模拟结果
