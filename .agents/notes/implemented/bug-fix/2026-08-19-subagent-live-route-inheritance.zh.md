# Agent Note: Subagents inherit the parent's live request route

Status: implemented

[English](2026-08-19-subagent-live-route-inheritance.md) | 中文

## Problem

进程内 subagent 创建原先从 `parent.options` 继承 `provider`、`model` 和 `maxTokens`。这些选项描述的是 parent Agent 的创建方式，却不一定等于其下一次请求实际使用的路由。因此，parent 在会话中切换 provider 或 model 后，新委派出的 child 仍会落到陈旧的创建时路由。

这个缺陷同时影响本地 one-shot 委派与 continuable 创建。One-shot child 会使用陈旧的 `AgentOptions` 进行物化；continuable 创建还会从陈旧的 parent options 推导持久描述符中的 `agentProvider` 与 `agentModel`，因此在 child 物化之前就已经把错误路由写入持久状态。

Session 已经拥有权威的增量请求路由快照：`session.requestHeader()` 返回下一次请求生效的 header。Child 路由应在委派边界使用同一份当前视图。

## Decision

进程内 child 的路由继承会在委派边界解析 parent 最新的 `session.requestHeader()?.config`。当其中存在对应值时，`provider`、`model` 和 `maxTokens` 都来自这份 live request config；如果尚无对应 live 值，则分别回退到 `parent.options`。显式的逐 child `agentOptions` 对所有继承值继续保持最高优先级。

`resolveChildAgentOptions()` 统一拥有两条委派路径的这项规则。

Continuable 创建会在第一次 await 之前只解析一次 `AgentOptions`。持久描述符从这份已解析快照读取 `provider` 与 `model`，之后的物化也复用同一个对象，而不会再次解析 parent 路由。因此，即使 parent 在委派开始后发生切换，描述符持久化与运行时 child 创建仍描述同一个委派时路由。

冷恢复不会重新读取 parent 当前路由。它继续依据 child 自己描述符中已持久化的路由进行重建，因此 parent 后续的切换不会追溯改变一个已经建立的持久 child。

## Alternatives considered

**继续从 `parent.options` 继承。** 拒绝，因为创建选项会有意保持稳定，而请求路由可以在 Agent 生命周期中变化。把创建选项当成当前路由状态，会在每次会话中途切换后重新产生陈旧 provider 问题。

**每次 parent 路由切换时同步修改 `parent.options`。** 拒绝，因为这会把创建元数据变成可变请求状态，并把一个 subagent bug fix 扩大成 Agent 全局选项语义变更。Session 已经显式记录当前请求配置。

**Continuable 描述符与 child 物化分别重新解析路由。** 拒绝，因为两者之间存在 await。如果期间 parent 再次切换，就可能持久化一个路由却物化另一个路由。

**只从 live header 复制 provider 和 model。** 拒绝，因为 `maxTokens` 属于同一份继承请求路由。让它与 provider/model 使用不同来源会使 child 请求配置内部不一致。

## Consequences

Parent 在 provider/model 切换之后进行委派时，child 现在会跟随 parent 下一次请求实际生效的路由，而不是 parent Agent 创建时使用的路由。在首个 request header 出现之前，通过逐字段回退到创建选项保持原有行为；显式 child override 不变。

Continuable 描述符现在持久化与实际 child 创建相同的 provider/model 解析快照。冷恢复仍保持稳定，因为它重建的是已持久化的 child 路由，而不是重新采样 parent 后续状态。

回归测试固定了两条路径：parent 的创建选项始终保持 `mock/mock`，而其 Session 从初始 request header 推进到 `live/live-model`，同时使用不同的 `maxTokens`。One-shot child 必须使用 live 路由物化；continuable 描述符与 child request header 则必须携带同一份 live 快照。
