# Proposal: align-console-config-lifecycle

## Summary

为 OpenClaw Office 控制台建立统一的“配置修改生命周期”方案，复用父级原生 OpenClaw 已有的 Gateway 能力，而不修改 OpenClaw 源码。

本变更将控制台中的配置修改入口统一规划为四类：

1. **动态 RPC 生效**：直接调用现有 RPC，修改后立即生效，不展示重启提示。
2. **保存后自动热加载**：通过 `config.set` 写入配置文件，让 Gateway 的内置 watcher 按 `gateway.reload.mode` 自动执行 hot/noop/restart 判定。
3. **显式应用并重启**：通过 `config.apply` 使用 Gateway 内置能力写入并调度重启。
4. **无法通过 UI 安全调度重启时**：明确展示 `openclaw gateway restart` 命令，并提示“修改已保存，重启后生效”。

## Problem

当前 OpenClaw Office 的配置管理存在两个结构性问题：

1. 多数页面将配置变更统一走 `config.patch`，而父级 OpenClaw 的 `config.patch` 会在后端无条件安排重启，掩盖了 Gateway 原生的 hot reload 能力。
2. 控制台缺少统一的“这次修改何时生效、是否需要重启、如何触发重启”的交互模型，导致不同页面行为不一致，用户难以理解。

## Goals

- 基于父级原生 OpenClaw 已有功能，为 Office 提供统一的配置修改体验。
- 让可热加载的配置项在修改后自动热生效。
- 对需要重启的配置项提供清晰、一致的应用与重启路径。
- 在无法安全通过 UI 调度重启时，提供明确的 CLI 指令提示。
- 统一规划 Settings / Agents / Skills / Channels / Cron 等控制台相关修改入口，避免每个功能独立设计。

## Non-Goals

- 不修改父级 `openclaw` 源码。
- 不新增 Gateway 后端 RPC（例如新的 `gateway.restart`）。
- 不重新设计整个控制台视觉风格。
- 不保证运行中的任务中途切换到新模型；本提案只定义“下一次运行何时读取新配置”的行为。

## Current-State Findings

- 父级 Gateway 已具备配置 watcher、reload plan 和 `off/restart/hot/hybrid` 模式。
- 父级 Gateway 的 `config.set` 只写文件，不主动安排重启。
- 父级 Gateway 的 `config.apply` 写文件后主动安排重启。
- 父级 Gateway 的 `update.run` 仅用于更新成功后的重启，不适合充当通用重启按钮。
- Office 当前缺少 `config.set` / `config.apply` 适配器能力，且多个页面通过 `config.patch` 写整份配置，导致本可热加载的路径也进入“保存即重启”。
- `agents.update`、`skills.update`、`cron.*` 等现有 RPC 天然属于动态或局部运行时操作，不应被纳入统一“保存后重启”的流程。

## Proposed Direction

### 1. 建立统一的配置生效策略矩阵

按“调用方式”而不是“页面位置”决定配置行为：

- **Runtime RPC**
  - 示例：`agents.update`、`skills.update(enabled/apiKey/env)`、`cron.add/update/remove/run`
  - 行为：保存后立即反馈成功，不显示重启 Banner
- **Config Save**
  - 调用：`config.set`
  - 适用：需要写配置文件，但可交给 Gateway watcher 自动判断 hot/noop/restart
  - 行为：显示“已保存”；若连接抖动或 watcher 导致重连，前端用现有连接状态反馈
- **Config Apply**
  - 调用：`config.apply`
  - 适用：用户明确接受“保存并重启”时
  - 行为：展示重启中状态与 reconnect 提示
- **CLI Fallback**
  - 当 `config.apply` 所需快照不可用，或界面上下文不足以安全发起重启时
  - 行为：展示命令 `openclaw gateway restart`

### 2. 用一个共享的“生效策略解析器”驱动所有相关页面

前端新增统一的策略层，对控制台修改入口标注：

- `runtime`
- `config-save`
- `config-apply`
- `config-save-with-cli-restart-hint`

所有页面共享同一套状态文案与交互组件，避免 Providers、Agent Tools、Agent Skills、Gateway Settings 各自发明一套重启逻辑。

### 3. 对现有页面进行统一归类

- **Agents 模型切换**
  - 继续优先使用 `agents.update`
  - 说明“新请求自动读取新配置；进行中的运行不切换”
- **Skills 页面启用/禁用、API Key、env**
  - 继续优先使用 `skills.update`
  - 不展示“立即重启”按钮
- **Cron 页面**
  - 继续使用 `cron.*`
  - 不纳入重启模型
- **Settings / Providers**
  - 从 `config.patch` 迁移到 `config.set` + 可选 `config.apply`
  - 默认先保存，让 Gateway watcher 自动 hot reload
  - 当 UI 能判断“这类修改需要用户确认重启”时，提供“保存并重启”
- **Agent Tools / Agent Skills Allowlist**
  - 不再用 `config.patch`
  - 改为整份配置快照编辑后走 `config.set`
  - 对需要重启的情况使用统一 Apply/CLI 提示，而不是错误调用 `update.run`

### 4. 替换误导性的“Restart Now”行为

当前 Agents 页签内的 `RestartBanner` 实际调用 `update.run`，语义错误。应统一替换为：

- 若具备 `config.apply` 条件：提供“应用并重启”
- 否则：展示 CLI 命令提示

## Why This Approach

- 最大限度复用父级原生 UI 与 Gateway 的既有行为模型。
- 不需要改 OpenClaw 后端即可释放现有 hot reload 能力。
- 避免继续扩大 `config.patch` 的使用面，减少“其实不用重启但被迫重启”的情况。
- 将“重启”从页面级临时行为提升为控制台级统一语义，后续扩展更稳定。

## User Impact

- 用户会看到更明确的结果类型：
  - “已立即生效”
  - “已保存，Gateway 将自动热加载”
  - “已保存，需重启后生效”
  - “请运行 `openclaw gateway restart`”
- 可热加载配置不再一改就强制重启。
- 需要重启的配置修改会有统一提示，不再出现把“更新程序”误当“重启 Gateway”的入口。

## Risks

- Gateway watcher 的重载结果前端无法直接获知具体判定（hot/noop/restart），只能依据调用路径和连接状态给出高可信提示。
- 某些配置路径虽被归类为 `config.set`，实际仍可能由 Gateway reload plan 判定为 restart；UI 需接受这种后端最终裁决。
- 会话级 model override 可能让“Agent 默认模型已修改”与“当前会话实际模型”短时不一致，需要在文案中说明边界。
