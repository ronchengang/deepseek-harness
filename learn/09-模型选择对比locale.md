# 对比案例:模型选择 vs 语言偏好——两种设置的本质区别

> 来源(都在代码里坐实):`packages/client/ui-model-selection/README.md`、`packages/api/session-controller/src/{index,commands,types}.ts`、`packages/core/agent/src/model-selection.ts`(经测试 `model-selection.spec.ts` 坐实)、`packages/core/agent-default-model/src/index.ts`。

## 为什么拿这两个对比

`08` 讲的 locale 是「纯前端全局偏好」。模型选择看起来也是一个设置项(截图里「模型」那一行),但它的存储和生效机制**完全不同**——因为它会影响后端行为、而且要进 session 日志。搞懂这组对比,就理解了 dsh 里设置的两条分野。

## 一张表看懂区别

| 维度 | 语言偏好(locale) | 模型选择(model selection) |
|---|---|---|
| 作用域 | **全局**(整个 home 共享一份) | **每个会话(per-session)**,各会话独立 |
| 存哪 | 用户设置文档(`settings.yaml` / `cordis.patch.yml`),按命名空间 | **会话日志**里的 `model/selection` 事件 |
| 存储性质 | Cordis 插件配置(volatile config) | 只追加的 `SessionEvent`,是会话历史的一部分 |
| 谁拥有 | 前端 `LocaleRuntime`(纯浏览器) | 后端 Agent / Session(前端只提交) |
| 生效方式 | 前端内存快照 + revision,立即重渲染 | 提交给 Host → append 事件 → 下一次请求组装时生效 |
| 影响模型请求吗 | 否(纯 UI 文案) | **是**(决定发给哪个 provider/model) |
| 能从日志重建吗 | 不需要(不进日志) | **必须**(模型可见 ⟺ 已记录) |
| 「默认值」在哪 | 浏览器自动检测 | `agentDefaultModel` 插件配置(走 settings) |

## 模型选择的完整链路

```
① 用户在 composer 的模型控件 或 /model 弹窗 选一个模型
   两个入口共用「每会话的目录」ModelDirectory(ctx.modelDirectories)
        │ 都调 session.selectModel 提交
        ▼
② 前端把它当作 pending 立即显示(spinner),但不改本地真值
   —— 真值由后端决定,等 Host 确认
        ▼
③ RPC: SessionController.selectModel(request)   @Remote('selectModel')
   (packages/api/session-controller/src/index.ts)
        │ 转给 commands.selectModel
        ▼
④ commands.selectModel(request)
   (packages/api/session-controller/src/commands.ts)
        ├─ resolveAgent(sessionId)  找到这个会话的 Agent
        ├─ 校验:provider/model 是否可路由、是否支持当前会话内容(如图片)
        │   不行就返回 ok:false(session/model-unavailable 等)
        └─ 通过 → agent.model.select(selection)
        ▼
⑤ AgentModel.select() → session.append('model/selection', selection)
   —— 往会话日志追加一个持久事件(不是写全局设置!)
        ▼
⑥ session projection 把 model/selection 事件折叠成状态
   modelSelectionProjectionDefinition → { lastUsed, next/pending }
challenge:  agent.model.current() 读的就是这个投影
        ▼
⑦ 下一个 step 组装时生效
   agent-loop 在 agent/pre-step 快照 modelSelection,freeze 进 request/header
   —— 正在跑的 step 保持它开始时的模型不变(README: "a running step keeps…")
        ▼
⑧ (可选)顺带更新"默认模型"
   被接受的选择可以存成 agentDefaultModel 的配置(走 settings,像 locale 那样)
   —— 这一步才是写全局设置,给"以后新建、从没选过模型的会话"用
```

## 三个关键设计点

### 1. 真值在后端,前端只是"提交 + 乐观显示"

locale 的真值在前端 `LocaleRuntime`,`setLocale` 本地说了算。模型选择相反:

- 前端选完先显示为 **pending(spinner)**,**不改本地真值**。
- 必须等 `session.selectModel` 的 RPC 回来,Host 校验通过(provider 可路由、模型支持会话里的图片等)才算数。
- 被拒绝(`ok:false`)则菜单保持打开,显示错误。

原因:模型能不能用取决于后端的凭证、catalog、会话内容,这些前端不知道,所以不能前端说了算。

### 2. 存进 session 日志,而不是全局设置

这是最本质的区别。`agent.model.select()` 做的是:

```
session.append('model/selection', selection)
```

也就是说,**换模型是会话历史里的一个事件**,和 user/message、tool/call 一样是只追加日志的一部分。这带来几个性质:

- **per-session**:每个会话有自己的模型选择轨迹,互不影响。
- **可重放**:从日志重建会话时,模型选择跟着重建(测试里 `Session.create(..., snapshotEvents())` 后 `current()` 还是之前选的那个)。
- **满足铁律**:模型选择直接影响发给模型的请求,按「模型可见 ⟺ 已记录」它必须进日志。locale 只是 UI 文案,不进日志。

### 3. 「当前会话选择」和「全局默认」是两套东西

- **当前会话的选择** → `model/selection` 事件(会话日志,per-session)。
- **全局默认模型** → `agentDefaultModel` 插件的配置(`packages/core/agent-default-model`),走的是和 locale **一样**的 settings 机制(volatile config,存用户设置文档)。给"新建的、从没选过模型的会话"用。

所以模型选择其实**横跨两种存储**:切当前会话走日志;把它固化成默认走全局设置。而 locale 只有全局设置这一种。

## 投影状态:lastUsed vs pending/next

模型选择的 session projection(`modelSelectionProjectionDefinition`)把日志事件折叠成:

- `lastUsed` — 已被某次请求 header 真正采用过的选择
- `next` / `pending` — 已选但还没被下一次请求采用的选择

测试里能看到:选了之后 `{ lastUsed: selected, next: selected }`;只改 reasoningEffort、或选了和当前一样的路由,不会重复 append(去重)。这套「pending → 被 header 采用 → lastUsed」的两段状态,正是为了表达「选了但要下一个 step 才生效,正在跑的 step 不受影响」。

## 一句话总结

**locale 是"全局 UI 偏好",存设置文档、前端即时生效、不进日志;模型选择是"会话级、影响模型请求的决策",存 session 日志(`model/selection` 事件)、后端校验后生效、必须可重放。** 两者唯一的交集是:模型选择可以额外固化成 `agentDefaultModel` 全局默认,这一步才和 locale 用同一套 settings 存储。

这组对比揭示了 dsh 设置的分野:**只影响"怎么显示"的 → 全局设置文档;影响"模型看到什么 / 会话怎么跑"的 → session 日志事件。** 判据就是那条铁律——会不会进入模型请求、要不要能从日志重建。

## 相关文件速查

| 文件 | 角色 |
|---|---|
| `client/ui-model-selection/src/client` | 两个入口(composer seat + /model 弹窗),共用 per-session ModelDirectory |
| `api/session-controller/src/index.ts` | `@Remote('selectModel')` 入口 |
| `api/session-controller/src/commands.ts` | `selectModel`:resolveAgent + 校验 + `agent.model.select` |
| `core/agent/src/model-selection.ts` | `AgentModel.select` → append `model/selection`;projection 定义 |
| `core/agent-default-model/src/index.ts` | 全局默认模型(走 settings,类比 locale) |
| `core/agent-loop` | 在 `agent/pre-step` 快照 modelSelection,freeze 进 request/header |
