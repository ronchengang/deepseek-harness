# Cordis 框架（底层插件框架）

> 这是读懂整个 dsh 的前提。Cordis 被 vendor 进 `vendor/`，官方入门见 `docs/cordis-primer.md`。

## Cordis 的五个核心概念

### 1. 插件就是实现了 Service 的对象

两种写法：

- **函数插件**：一个函数，带可选的 `inject` 和 `apply(ctx)` 字段。
- **Service 子类**：一个 `Service` 子类，由 Cordis 把它的生命周期挂进当前 context。

> 约定（见 `packages/AGENTS.md`）：service 包默认导出 service 类；函数插件用具名导出 `name` / `inject` / `Config` / `apply`，**不能有默认导出**。混用会让 Loader 丢弃函数插件的命名空间。

### 2. context 是服务的仓库

一个 service 会占用一个稳定的 `ctx.<key>`，比如：

- `ctx.tools` — 工具注册表
- `ctx.llm` — 模型消息/流式适配器
- `ctx.sessions` — 会话日志
- `ctx.agents` — 活跃 Agent 注册表

其他插件通过 **key 查找服务**，而不是 import 某个具体实现。这就是「可替换」的基础。

> 可选服务用 `ctx.get(name)` 读取；`ctx.<name>` 只保留给声明过的注入（injection），因为属性代理对拓扑敏感。

### 3. 用 `inject` 声明服务依赖

插件列出它需要的服务，Cordis 会等这些服务就绪后再加载它。**加载顺序由依赖关系推导，而不是手写启动序列。**

### 4. 类型化事件用于通信

服务通过 TypeScript 的 **声明合并（declaration merging）** 声明事件名，然后按不同「派发模式（dispatch mode）」派发。派发模式是事件公开契约的一部分，新事件要用 `@mode` 标注。

| 模式 | 是否 await | 派发顺序 | 有返回值 |
|---|---|---|---|
| `emit` | 否 | 按注册顺序观察 | 无 |
| `waterfall` | 否 | 按注册顺序观察 | 有 |
| `parallel` | 是 | 所有监听器并行观察 | 无 |
| `serial` | 是 | 按注册顺序观察 | 有 |
| `bail` | 否 | 按注册顺序，直到某个 bail | 有 |

### 5. 注册是可撤销的副作用

Prompt 片段、工具 schema、适配器、provider、监听器，全都通过 `ctx.effect()` 或 `ctx.on()` 安装，这样 reload / teardown 时能可预测地撤销。

> 仓库规则：**每个贡献都要走 `ctx.effect()` / `ctx.on()`**，registry 的 `register()` 返回 disposer（直接返回或通过注册句柄）。

## Waterfall 语义（重点，容易踩坑）

`ctx.waterfall` 是「环绕式中间件（around-middleware）」。监听器签名是 `(...args, next)`：

- **调用 `next()`** → 把（可能被包装过的）结果委托给下一个服务。
- **不调用 `next()` 直接返回** → 短路，后续监听器看不到。
- 值通过 `next()` 的返回值在链上传播。

规则：**只做注解/观察的监听器必须调用 `next()` 委托**；拥有决定权的策略监听器才可以短路。用 `prepend: true` 只在必须先于普通注册运行时使用。

## Loader 配置（cordis.yml）

- `@deepseek-ai/cordis-plugin-include` 会把 `!!js`（注意是两个感叹号，不是 `!js`）解析成表达式节点。
- Loader 对条目的 `config`（在声明注入激活后、针对该插件 context 求值）和 `disabled` 字段（每次挂载决策时、针对 loader context 求值）做插值。
- 其他条目元数据保持字面量。环境选择插件时用 overlay（覆盖层）。

## 实践要点

- 把行为封装进插件：工具管线事件属于 `ctx.tools`，模型流式属于 `ctx.llm`，活跃 Agent 协调属于 `ctx.agents`。
- 拦截和策略优先用事件；直接调用能力优先用服务方法。
- 每个注册都要有 disposer。如果 teardown 顺序重要，就把相关工作放进同一个 effect。
