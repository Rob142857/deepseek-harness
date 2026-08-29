# Agent Note: 根据调用会话投影 pwsh 升级 schema

Status: implemented

[English](2026-08-29-session-scoped-pwsh-escalation-schema.md) | 中文

## Problem

原生 OpenAI Responses 路由在无害 PowerShell 命令运行前产生了确定性失败。会话已经使用 `danger-full-access`，但模型仍以默认值填满可选参数，并请求 `sandbox_permissions: "danger-full-access"`。执行守卫正确地以「并非严格更宽」拒绝该调用，但注册表全局 schema 向调用会话公布了一个不可能的选项。仅靠提示词未能阻止模型反复发送默认占位值，例如空 `workdir`、`run_in_background: false` 或冗余的升级参数对。

[子进程 sandbox 决策](../feature/2026-07-06-sandbox.zh.md)有意保留全局升级词汇，因为注册时默认值无法描述按会话覆盖。按会话重新注册工具仍是错误的归属模型，但原生请求 assembly 现在携带确切的调用 agent，因此可以在不改变注册表的情况下呈现更窄的模型界面。

## Decision

在约束型组合中，`dsh-tool-pwsh` 保留一个注册表全局定义，其中包含完整的 `ESCALATION_TARGETS` 词汇。在 `system-prompt/assemble` 中，插件等待下游 assembly 完成，确认可见的 `pwsh` 定义正是自身注册项，再通过 `ctx.sandboxPolicy` 解析调用会话，并只复制该定义，以 `WIDER_MODES[effectiveMode]` 投影：

- `read-only` 提供 `workspace-write` 与 `danger-full-access`。
- `workspace-write` 只提供 `danger-full-access`。
- `danger-full-access` 移除 `sandbox_permissions`、`justification` 与仅用于升级的说明。

无 agent 的诊断 assembly 因没有可供投影的会话事实而保留注册表基线。按请求复制可防止具有不同策略的并发 agent 改变或观察彼此的定义。作用域替换或隐藏 `pwsh` 也保持不变，因为只有可见定义与已注册工具完全相同时才会投影。

每个可选字段现在都首先说明省略语义。描述会指示模型在普通前台调用中只发送 `command` 与 `description`，只以 `true` 发送 `run_in_background`，并且只在实际 sandbox 拒绝后发送升级字段。这些描述也会到达根据约束型组合的注册表基线渲染的 PTC 模式 SDK 文本。

执行仍是安全边界。`approveEscalation()` 会在批准或启动进程前，根据调用解析后的有效模式独立检查严格放宽，因此过期 schema 与注入参数仍会 fail-closed。

## Alternatives considered

**只增加提示词说明。** 否决，因为现有工具描述已经禁止推测性升级，但模型仍然填充每一个可选属性。

**静默丢弃非严格放宽的升级参数对。** 否决，因为忽略模型请求的语义后再以既有更宽访问执行，会掩盖畸形调用并削弱审计轨迹。

**为每个会话重新注册独立工具 schema。** 否决，因为注册表身份是进程全局且稳定的。按请求投影无需可变注册即可提供所需的模型界面。

**只在机器匹配到先前拒绝后才公开升级。** 暂缓，因为命令身份会受到引号、workdir、环境前缀和管道影响，十分脆弱。现有拒绝提示、请求描述、人工批准与执行守卫继续构成强制执行流程。

## Verification

聚焦单元覆盖会并发 assembly read-only、workspace-write 与 danger-full-access agent，固定无 agent 的注册表基线，并注入一次精确的 danger-full-access 到 danger-full-access 调用，以证明它既不提示也不执行。包构建与类型感知 lint 通过。无密钥 Windows `pwsh-tool-turn` replay 使用 danger-full-access 下真实的 sandbox PowerShell 执行器，并快照一个不含升级参数对的请求 schema。

## Consequences

直接的原生 `pwsh` schema 不再提供 harness 必须拒绝的同模式升级。在约束型组合中，注册表检查、无 agent 诊断与 PTC 模式 SDK 文本有意保留完整词汇；同模式 PTC 调用与任何其他不受信任输入仍会在执行守卫处 fail-closed。

当 `pwsh` 仍可见时，会话 sandbox 模式切换可能改变原生请求的工具 header，并从首个变化的 schema token 起使 KV Cache 复用失效。接受这一代价，是因为模式切换很少，而公布不可能的可选值会造成确定性的用户可见失败。该实现有意只应用于 `pwsh`；其他工具家族保留既有 schema 决策，直到同等证据支持独立变更。
