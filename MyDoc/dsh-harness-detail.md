# DeepSeek Harness 面试深度详解

> 2 小时精读。跟进到函数名与边界条件。概念索引：[dsh-harness.md](dsh-harness.md)。
> 深度模拟测验：[dsh-harness-detail-cards.html](dsh-harness-detail-cards.html)

口诀（另一角度）：**看见就记账；idle 才续跑；激活不随盘；证据才能完。**

---

## §1 Cordis：everything is a plugin

源：[`docs/cordis-primer.md`](../docs/cordis-primer.md)、[`docs/architecture.md`](../docs/architecture.md)

没有特权内核。模型适配器、工具注册表、session log、agent loop 本身都是插件，卸载时 `ctx.effect()` / `ctx.on()` 的注册会按序撤回。

### §1.1 五条

1. Plugin 是实现 `Service` 的对象（函数插件带 `name`/`inject`/`Config`/`apply`，无 default export；服务类 default-export）。
2. Context 是服务仓库：`ctx.tools`、`ctx.llm`、`ctx.sessions`。
3. `inject` 表达依赖，而不是手写 boot 顺序。
4. 事件用 declaration merging 扩类型；dispatch 模式是公开契约：`emit` / `waterfall` / `parallel` / `serial`。
5. 注册必须可逆。

函数插件若再 default-export，Loader 会丢掉 named namespace（[postmortem 0001](../docs/postmortem/0001-acp-default-export-drops-inject.md)）。可选服务用 `ctx.get(name)`，不要走拓扑敏感的属性代理。

### §1.2 Waterfall

`ctx.waterfall` 是环绕中间件：`(…args, next)`。调用 `next()` 把可能被包装的结果交给下游；不调则短路。策略事件（`agent/pre-step`、`tools/pre-execute`）短路是设计；只观察的监听器必须委托。

```ts
// 伪代码：监听器 2 看到 blocked 直接返回，最内层默认逻辑永不跑
ctx.on('demo/transform', async (input, next) => {
  if (input.includes('blocked')) return 'REPLACED'
  return (await next()).toUpperCase()
})
```

闭环联合：`SessionEventMap`、`MessageSourceMap`、`ContentBlockMap` 都靠 declaration merging 扩；`switch` 判别标签，封闭联合用 `assertNever`，可合并联合走文档化 default。

### §1.3 Profile / bundle

运行中的 `dsh` 是有序图层：bundle 列表 → profile `cordis.patch.yml` → home patch → `--patch`。`dsh-base` 是每条 profile 的第一层。`dsh --profile web --dump-config` 打印实际树。

---

## §2 Turn / Step / Inbox

源：[`docs/architecture.md#turn-flow`](../docs/architecture.md#turn-flow)、[`docs/agent-lifecycle.md`](../docs/agent-lifecycle.md)、[`docs/subsystems/core.md`](../docs/subsystems/core.md)

```text
turn/start
  claim next-step + 一条 queued message
  assemble prompt + tool schemas
  -> agent/pre-step   reject | enter(messages)
     reject 或首轮 enter 被改空 -> 关 turn，零 step（日志仍记这次尝试）
     step/start
     user/message*
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message
     tool/call* -> tools/pre-execute -> execute -> post-execute -> tool/result*
     step/end
     工具还欠一次请求，或 next-step 又来了 -> 下一 step
  -> agent/turn-stopping   (serial，无 next)
turn/end
```

`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*` 是**耐久** session 事件；`agent/pre-step`、`agent/request`、`llm/stream`、三段 `tools/*` 是 **waterfall**；`agent/turn-stopping` 是 **serial**。

### §2.1 Agent 句柄

`Agent.status` 只有 `'idle' | 'running'`。`running` 是 driver 排水区间，可能跨多个排队 turn，**不能**证明某个 turn 仍开着。disposal 发 `agent/disposed`，不是第三种 status。

`whenIdle()` 观察整个 agent，不是某条 followup 的完成。若干条 followup、steering、inject 可能共享一个 `running` 区间。

### §2.2 投递三件套

| API | inbox 目标 | 唤醒？ |
|---|---|---|
| `followup(msg)` | next-turn，独占该 turn 的普通消息 | 是 |
| `steer(msg)` | 最近 step 边界 | idle 则开 turn；running 则下一 step |
| `inject(msg)` | next-step 上下文 | **否**；idle 一直等到别人唤醒 |

`send(message, target, wakeup)` 是统一入口。被拒的 step 把 steering 停在 inbox，直到下一次 wake。

`followup()` **不返回**可等待的结果句柄：`MessageId` 只标识 inbox 插入/认领/丢弃。自动化调用方若真的「拥有一次 run」，必须自己定义区间（例如从 inbox receipt 到下一次 whole-agent idle）。

### §2.3 扩展点，不是改 loop

新行为挂文档化扩展点。改 `agent-loop` 必须改 `docs/architecture.md`。扩展插件依赖 `dsh-agent`，**永不**直接依赖 `dsh-agent-loop`。

`agent/pre-step` 的 `PreStepDecision`：`{kind:'reject'}` 或 `{kind:'enter'; messages}`。返回的决策是权威的；包装 `next()` 的监听器除非故意替换，应保留下游 messages。

---

## §3 Session log

源：[`docs/subsystems/session.md`](../docs/subsystems/session.md)

`Session` = append-only `SessionEvent` 日志。LLM 历史由 `deriveMessages()` **投影**，从不另存。每条有单调 `seq`、`time`、判别 `data`。

**Model-visible means logged.** 运行时 invariant 断言这一点。新的模型可见输入 → 扩 `SessionEventMap` 并从 log 渲染。

Goal 把 `goal/change` merge 进 `SessionEventMap`；payload 是完整 post-mutation snapshot 或 clear tombstone。Inbox 变动**不影响** goal 状态。

`todo/write` 是整表快照、last-write-wins，条目无稳定 id。

---

## §4 Capability seam

源：[`docs/glossary.md#capability-seam`](../docs/glossary.md#capability-seam)、[`docs/capability-seams.md`](../docs/capability-seams.md)

完整能力 = Definition（拥有 `ctx.<key>` 的 Cordis `Service` 抽象类或注册表，**不是** TS `interface`）+ Provider + Consumer。`packages/shell` 是范本：`dsh-shell` / `dsh-bash-local` / `dsh-tool-bash`。

换一个 FS/subprocess provider 指向远程 sandbox，Bash、PTY、LSP 一起走，无需各写一份。

Subagent 特殊：同一 context 里**多个** named provider 共存（像 LLM adapter registry）；workflow engine 则每 context **一个**。

---

## §5 Same-session Goal

源：[`packages/goal/goal/README.md`](../packages/goal/goal/README.md)、[`docs/subsystems/goal.md`](../docs/subsystems/goal.md)、[`packages/goal/goal-round-driver/README.md`](../packages/goal/goal-round-driver/README.md)、[`packages/goal/tool-goal/README.md`](../packages/goal/tool-goal/README.md)

对照 Codex goal（`codex-rs/ext/goal`）：两边都是「跨 turn 粘住目标」，但 dsh **刻意拆开**持久状态、续跑策略、模型工具、人类命令。

### §5.1 四包拆分

| 包 | 角色 | 不负责 |
|---|---|---|
| `dsh-goal` | 事件溯源状态、CAS、`goal/change` | 何时续跑、token 预算 |
| `dsh-goal-round-driver` | idle + armed 时排队 `<goal_round>` | 语义是否完成 |
| `dsh-tool-goal` | 模型 `get/create/update_goal` + 权限 | 调度 |
| `dsh-command-goal` | `/goal` 人类命令，不进模型 | 模型可见记录（仍由 domain 写） |

最多一条 current goal。未完成的必须 edit / 转换 / clear；**已 complete 的才允许用全新 id 替换**。

### §5.2 Phase vs Activation（面试高频）

```ts
type GoalPhase = 'active' | 'paused' | 'blocked' | 'complete'
type GoalActivation = 'armed' | 'disarmed'  // 从不落盘
```

- `phase`：目标发生了什么，strict replay 只从 `goal/change` 折叠。
- `activation`：本进程是否允许自动再开一轮。新鲜 cache、每次 `agent/session-start`、driver unload、flush 失败都会 **disarm**，即使 durable phase 仍是 `active`。
- `disarm()` 不写 revision、不发 mutation。resume/fork 后必须人类或模型走 `resume` 再授权。

这是相对 Codex 的关键差异：Codex 在 restart 上 **re-arm idle loop**；dsh 认为进程边界后的自动续跑是危险的，必须再授权。

Provider 限额、配置预算、执行错误、等人输入，全部挤进 **一个** durable `blocked` phase（带 `code` + `message`），不增殖生命周期状态。dsh **没有** Codex 的 `UsageLimited` / `BudgetLimited`；`maxGoalRounds` 只计 admitted round。

### §5.3 Round 合同

Driver 在 **whole-agent idle**、active + armed、仍有容量时：

1. `ctx.sessions.flush()` 并在 await 后再核对 revision 与竞争输入。
2. 预留 `roundsStarted + 1`（过期预留不消耗编号）。
3. `followup` 一条 `<goal_round>`，`source.kind === 'goal'`。
4. `agent/pre-step` 在下游前后都核验；**只有 entered `user/message` 才递增 `roundsStarted`**。
5. 人类消息抢先入 inbox → 自动工作让路；混合 batch 里的自动 prompt 被拒，checkpoint 后再预留。

Prompt 把 objective `JSON.stringify` 进标签，避免多行/类标签文本破坏结构；要求把 workspace、tool results、durable session 当权威，**叙述不当真**。

取消：与 goal 尝试相关的取消会在下一 idle **pause** 目标（pause 失败则 fallback disarm），避免取消后自动重启。无关取消只 disarm。

### §5.4 工具权限

源：[`packages/goal/tool-goal/src/authority.ts`](../packages/goal/tool-goal/src/authority.ts)

```ts
// 必须是 registry 里那份精确 live Agent、status===running、currentInitiator===agent、开着 turn
ctx.agents.get(agent.id) === agent && agent.status === 'running'
  && ctx.agents.currentInitiator() === agent
```

- create / edit / pause / resume：当前 **runtime-root** turn 里要有 `source.kind === 'user'` 的已接受消息。`followup()`/`steer()` 若省略 source 会默认 `user`，所以插件/调度器必须自带 source，否则会偷到人类权限。
- complete / blocked：人类 **或** 当前 goal 的精确 admitted round（id + revision + round === `roundsStarted`）。
- goal-round 的 `blocked` 在 `blockedAfterConsecutiveRounds`（默认 3）之前机械拒绝；人类可立即停。
- 自主 round 成功 `complete`/`blocked` 会 `concludeTurn()`；人类突变不会，助手仍可致谢、steering 仍可用。

模型侧政策原文（阈值被插值）：

> Mark complete only when the objective is actually achieved. Mark blocked only after the same blocking condition persists for at least 3 consecutive goal rounds … difficulty, uncertainty, or useful remaining work is not blocked.

语义是否「同一条件」仍是**模型判断**；运行时只强制 distinct admitted-round 计数。独立 evaluator 延期。

### §5.5 `/goal` 命令陷阱

控制词仅当占据**完整**输入时大小写不敏感。`/goal pause after verification` 创建字面目标，不是 pause。未完成目标不会被裸 `/goal <obj>` 替换。command 输出是 UI 状态；耐久记录仍是 `goal/change`。

### §5.6 vs Ralph / vs Codex

| | Same-session Goal | Ralph | Codex goal（对照） |
|---|---|---|---|
| 会话 | 同一 session 续跑 | 每 round 全新 child，无 parent/prior 种子 | 同 thread |
| 跨 round 记忆 | 完整 session 历史 | workspace + 有界 handoff | thread 历史 |
| 预算 | `maxGoalRounds` | Ralph round limit | token + wall-clock |
| restart | **disarm**，要再 resume | 新 child | **re-arm** idle loop |
| 完成 | 取证 + 工具 mutation | worker 报告 complete/blocker | 取证 |
| 谁调度 | 独立 driver 插件 | workflow/subagent 工具政策 | extension runtime |

口诀：**Goal 粘会话；Ralph 换脑不忘活；Codex 重启用，dsh 要人点头。**

### §5.7 已知限制（面试加分）

- 无独立 evaluator。
- 无 token/货币/墙钟计量。
- 无异常自动重试。
- 可信进程内生产者可伪造 `goal/change`；strict replay 做完整性检测，不是插件隔离。
- Cordis unload 异步：已 accepted 的 prompt 可能在 teardown 前开跑并消耗 round，随后取消 + disarm，不再开下一 round。

---

## §6 Scope

源：[`docs/subsystems/scope.md`](../docs/subsystems/scope.md)、[`docs/glossary.md#agent-scope`](../docs/glossary.md#agent-scope)

两级扁平：global 或恰好一个 scope key（惯例：活 `Agent` 对象本身）。**不向 subagent 继承**。子树用 `parentSession` / `delegationDepth` / `subagentDepth` 数据，不用 scope 结构。

`shadowing`：同名 scoped 工具/section/variable 对该 scope 替换 global。`tools.restrict` 先过滤 global，再 merge scope-local；被滤掉的 global 在 prompt 与执行上等同不存在。

`setup` 窗口：scope 与 agent 对象已在、尚未 publish / `agent/session-start` / 首次拼 prompt。setup 只注册，不驱动。

---

## §7 Invariants / 防御模式

源：[`docs/subsystems/invariants.md`](../docs/subsystems/invariants.md)、[`docs/defensive-patterns.md`](../docs/defensive-patterns.md)

每个包有 `./invariant` companion，按 **npm 全名** 注册。断言权威事件流或可变数据；空 installer 必须以 `No runtime invariant:` 开头并解释。禁止用「服务在不在」当关系。

防御高频：

- 正交结果独立上报（timeout **且** exit 0 要同时看见）。
- 公共契约两边都守（adapter 可 throw 或 emit error chunk，runtime 对外只归一成 terminal finish）。
- 异步状态 ≠ 同步状态（`whenIdle` ≠ 某条 followup 的结果）。
- dispose 必须等到 quiescence，不只发 kill。
- 回调异常在 dispatcher 内吞掉，一个坏订阅者不拆核心。
- 不把环境密钥和可预测路径交给不信任输出。

---

## §8 测试政策

源：[`docs/testing.md`](../docs/testing.md)

| 层 | 命令 | 要点 |
|---|---|---|
| unit | `pnpm run test` | 每 registry 要 HMR-safety（dispose fiber 看清理） |
| coverage | `pnpm run test:coverage` | `packages/*/*/src` **逐文件 100%**（CI 门） |
| e2e | `pnpm run test:e2e` | 真 API；无 key 自跳过 |
| snapshot | `pnpm run test:snapshot` | 组装应用的 keyless 转录；包测/e2e/mock **不能替代** |

产品可见插件必须有真实 Loader `cordis.yml` 组合测试。验证世界（重读文件），不要信模型自报。SessionEventMap / loop / lifecycle 变更要同时更新 TS 与 Python SDK snapshot。

---

## 追问链（How / Why / What if）

**Why 激活不落盘？** 进程崩溃、fork、driver 替换后自动续跑可能在错误世界里烧 round。人类再 `resume` 是授权边。

**How round 编号防双花？** 先 CAS 预留 `roundsStarted+1`；pre-step 拒绝则编号不消耗；只有 entered `user/message` 推进 fold。

**What if flush 失败？** `agent/error` 路径 disarm，不再开 round。

**Compare goal vs compaction？** compaction 改 derived history 后缀；goal round 是 append-only user 消息，直到 compaction 把它阴影掉。

**Debug：resume 后不续跑？** 先 `get_goal` 看 `activation`。是 `disarmed` 就对了；走 `/goal resume` 或 `update_goal action resume`。

**Interview：为什么不把 driver 写进 `dsh-goal`？** 状态与调度分离：ACP 可只要 domain+tools、不要命令；headless demo 默认可关 goals，避免 one-shot 变成多 round。
