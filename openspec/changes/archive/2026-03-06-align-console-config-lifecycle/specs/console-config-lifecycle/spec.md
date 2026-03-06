## ADDED Requirements

### Requirement: 控制台 SHALL 统一判定配置修改的生效策略

系统 SHALL 为所有控制台配置修改入口应用统一的生效策略分类，而不是由各页面独立决定是否提示重启。

生效策略至少包含：

- `runtime`
- `save`
- `apply`
- `save_then_cli_restart_hint`

#### Scenario: Agent 模型修改走 runtime

- **WHEN** 用户在 Agents 页面修改某个 agent 的模型，且底层调用 `agents.update`
- **THEN** 系统 SHALL 将该操作归类为 `runtime`
- **THEN** 成功后 SHALL 提示“新请求将读取新配置”
- **THEN** SHALL 不展示“立即重启”按钮

#### Scenario: Provider 配置保存走 save

- **WHEN** 用户在 Settings 页面编辑 provider 配置并点击保存
- **THEN** 系统 SHALL 优先使用 `save` 策略
- **THEN** SHALL 调用 `config.set`
- **THEN** SHALL 提示 Gateway 会根据内置 reload 策略自动热加载或在必要时重启

#### Scenario: 用户显式选择应用并重启

- **WHEN** 某个配置修改界面提供“保存并重启”入口且用户主动确认
- **THEN** 系统 SHALL 使用 `apply` 策略
- **THEN** SHALL 调用 `config.apply`
- **THEN** SHALL 显示统一的重启中与重连状态反馈

#### Scenario: 无法安全通过 UI 调度重启时降级到 CLI

- **WHEN** 页面缺少有效的配置快照、baseHash 或其他发起 `config.apply` 所需条件
- **THEN** 系统 SHALL 使用 `save_then_cli_restart_hint`
- **THEN** 在保存成功后显示 `openclaw gateway restart` 命令
- **THEN** 提示用户“修改已保存，重启后生效”

### Requirement: 控制台 SHALL 为配置文件写入优先使用 config.set 而不是 config.patch

对于需要写入 Gateway 配置文件的控制台功能，系统 SHALL 默认使用 `config.set`，除非该功能明确需要“写入并立即安排重启”的 `config.apply` 语义。

#### Scenario: Agent tools 配置保存不再触发 patch 强制重启

- **WHEN** 用户保存 Agent Tools 配置
- **THEN** 系统 SHALL 基于当前配置快照生成更新后的完整配置
- **THEN** SHALL 使用 `config.set` 持久化
- **THEN** SHALL 不再依赖 `config.patch` 返回的重启调度结果

#### Scenario: Agent skills allowlist 保存不再触发 patch 强制重启

- **WHEN** 用户保存 Agent Skills allowlist
- **THEN** 系统 SHALL 基于当前配置快照生成更新后的完整配置
- **THEN** SHALL 使用 `config.set` 持久化
- **THEN** SHALL 通过统一生命周期反馈告知用户何时生效

### Requirement: 控制台 SHALL 统一展示配置修改结果

系统 SHALL 使用共享交互模式展示配置修改结果，避免页面之间出现不同文案或错误的操作建议。

#### Scenario: 运行时立即生效

- **WHEN** 某项修改属于 `runtime`
- **THEN** 页面 SHALL 展示“已生效”或等价反馈
- **THEN** SHALL 不展示 restart banner

#### Scenario: 保存后依赖 Gateway watcher

- **WHEN** 某项修改属于 `save`
- **THEN** 页面 SHALL 展示“已保存，Gateway 将自动处理热加载或后续生效”的反馈
- **THEN** 若 WebSocket 连接因 Gateway 内部 reload 出现短暂中断，页面 SHALL 复用现有连接状态反馈而不是额外弹出更新提示

#### Scenario: 需要命令行重启

- **WHEN** 某项修改属于 `save_then_cli_restart_hint`
- **THEN** 页面 SHALL 展示 `openclaw gateway restart`
- **THEN** SHALL 明确该命令是使保存的配置生效所必需的下一步

### Requirement: 控制台 SHALL 删除将 update.run 误用为重启入口的行为

系统 SHALL 不再把 `update.run` 用作普通配置重启按钮。

#### Scenario: RestartBanner 不再调用 update.run

- **WHEN** 用户在任意控制台页面看到“需要重启”的提示
- **THEN** 该提示对应的主要动作 SHALL 是 `config.apply` 或 CLI 命令提示
- **THEN** SHALL NOT 调用 `update.run` 作为重启替代手段
