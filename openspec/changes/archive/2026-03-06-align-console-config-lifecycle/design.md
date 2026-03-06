# Design: align-console-config-lifecycle

## Overview

本设计引入一个统一的“配置生效策略”层，作为控制台页面与 GatewayAdapter 之间的中间抽象。目标不是新增后端能力，而是把父级 OpenClaw 已有的三种原生能力正确投射到 Office：

- `config.set`
- `config.apply`
- 运行时 RPC（`agents.update` / `skills.update` / `cron.*` 等）

同时定义 CLI fallback，用于无法安全通过 UI 发起重启的场景。

## Constraints

- 不修改父级 `openclaw` 源码。
- 只能使用当前父级 Gateway 已暴露的方法。
- 必须兼容 MockAdapter。
- 不能把 `update.run` 当作通用重启手段。

## Architecture

### 1. Config Mutation Intent

新增统一的前端意图类型：

- `runtime`
- `save`
- `apply`
- `save_then_cli_restart_hint`

该意图不与页面绑定，而与“修改动作类型”绑定。

### 2. Shared Resolution Layer

新增共享解析器，输入：

- 修改目标类型
- 是否有配置快照 `raw/hash`
- 是否允许 UI 发起 `config.apply`
- 是否属于动态 RPC

输出：

- 要调用的 adapter 方法
- 成功提示文案类型
- 是否展示 restart banner
- 是否展示 CLI 命令

### 3. Adapter Surface

为现有 `GatewayAdapter` 增加以下能力：

- `configSet(raw, baseHash)`
- `configApply(raw, baseHash, params?)`

不新增 `restartGateway()`，因为父级 Gateway 没有公开对应 RPC。重启仅通过已有 `config.apply` 达成。

### 4. Shared UI Feedback

新增统一反馈组件/状态枚举：

- `effective-now`
- `saved-hot-reload`
- `saved-restart-required`
- `saved-cli-restart-required`
- `apply-restarting`

替换当前零散的 `RestartBanner` 逻辑。

## Behavior by Console Area

### Settings / Providers

- 编辑 provider 配置时，默认走 `config.set`
- 保存后提示“Gateway 将按内置 reload 策略自动生效”
- 若用户主动点击“保存并重启”，则改走 `config.apply`
- 若 `raw/hash` 不可用，则降级展示 CLI 命令

### Agent Tools / Agent Skills Allowlist

- 仍然通过整份配置快照进行修改，但改用 `config.set`
- 不再通过 `config.patch` 触发强制重启
- 保存后统一显示配置生命周期结果

### Agent Model

- 保持 `agents.update`
- UI 文案明确：
  - 新请求读取新模型
  - 进行中的运行不切换
  - 会话级 model override 仍可能覆盖默认值

### Skills / Cron

- 保持动态 RPC 路径
- 不引入额外重启入口

## Key Tradeoffs

### Why not keep using `config.patch`

因为父级 Gateway 的 `config.patch` 会无条件安排重启，这与“尽量释放热加载能力”的目标相冲突。

### Why not add a fake restart RPC in Office

因为用户明确要求不能修改父级 OpenClaw 源码。Office 只能消费现有后端能力，不能假设新的后端接口。

### Why allow `config.apply` as UI restart path

这是父级 Gateway 与原生 UI 已提供的正式能力，属于“原始版本内置功能”。虽然它本质是“写配置并安排重启”，但在 Office 中可作为“应用并重启”的合法实现。

### Why still keep CLI fallback

有些界面上下文可能只有结构化 patch，没有完整原始配置快照；或当前配置状态不适合直接发起 apply。CLI fallback 可以避免前端做不安全的猜测。

## Migration Plan

1. 先扩展 adapter 和 config store 能力。
2. 引入统一配置生效策略层与共享反馈组件。
3. 将所有 `config.patch` 写路径迁移到新策略。
4. 清理页面内自建的 restart banner / update.run 误用。

## Validation Strategy

- 单元测试：
  - 策略解析器
  - store 对 `save/apply/runtime` 三类路径的状态流转
- 组件测试：
  - 成功提示、重启提示、CLI 命令提示
- 手工验证：
  - Provider 配置保存
  - Agent tools/skills allowlist 保存
  - Agent model 修改
  - Skills API key/env 保存
