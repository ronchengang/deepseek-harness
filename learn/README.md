# DeepSeek Harness 学习笔记

这是对 `deepseek-harness` 仓库做整体扫描后整理的学习笔记，用中文写，方便快速上手。

> 注意：本目录是**个人学习笔记**，不是项目官方文档。项目官方文档在 `docs/`（中英双语，有严格规范）。

## 阅读顺序

| 文件 | 内容 |
|---|---|
| [00-项目概览.md](00-项目概览.md) | dsh 是什么、「一切皆插件」理念、顶层目录、运行方式 |
| [01-Cordis框架.md](01-Cordis框架.md) | 底层插件框架 Cordis 的五个核心概念（读懂一切的前提） |
| [02-包结构.md](02-包结构.md) | `packages/` 的能力族分组、core 骨架、capability seam、profile/bundle |
| [03-核心运行流程.md](03-核心运行流程.md) | Turn/Step 执行流程、事件三域、会话日志、「模型可见⟺已记录」 |
| [04-开发与测试.md](04-开发与测试.md) | 常用命令、测试体系、编码约定、门禁 |
| [05-desktop前后端视角.md](05-desktop前后端视角.md) | 用「前端/后端」理解 apps/desktop：前端=Web 客户端，后端=Host 子进程，Electron=薄壳 |
| [06-前端也是插件.md](06-前端也是插件.md) | 前端界面本身就是 Cordis 插件树：Slot 系统、五类 props、三层单向知识、模块图 vs DI |
| [07-设置页拆解.md](07-设置页拆解.md) | 实例：一张设置页由十几个插件拼出来，逐行坐实每行来自哪个插件 |
| [08-切换语言端到端.md](08-切换语言端到端.md) | 端到端纵切：切语言怎么立即生效、存到哪（`$DSH_HOME/cordis.patch.yml`）、为何是一条 Cordis 配置 |
| [09-模型选择对比locale.md](09-模型选择对比locale.md) | 对比：模型选择为何存 session 日志（`model/selection` 事件）而非全局设置，揭示 dsh 设置的分野 |
| [10-无前端只跑后端.md](10-无前端只跑后端.md) | 能不能只跑后端：headless/sdk/acp 四个无 GUI profile，后端是独立完整的 Cordis 树 |

## 一分钟速记

- **dsh** = DeepSeek AI 的开源 Agent 运行框架，核心理念「一切皆插件」。
- 底层是 **Cordis**：context 是服务仓库，插件贡献 service + 类型化事件 + 可撤销副作用。
- 代码主体在 `packages/`，骨架在 `packages/core/`（session / tools / agent / agent-loop / llm）。
- 运行时核心是 **turn → step** 循环，会话是**只追加日志**，是模型上下文的唯一来源。
- 最重要的约束：**模型可见的一切都必须能从 session 日志重建**。
- 扩展方式：**挂插件、用事件**，不要改主循环。

## 想深入时去哪看

- 架构权威文档：`docs/architecture.md`
- Cordis 入门：`docs/cordis-primer.md`（还有 `docs/cordis-tutorial/`）
- 包地图：`packages/README.md`
- 事件表：`docs/event-producer-consumer.md`
- 开发/测试：`docs/development.md`、`docs/testing.md`
- 给 agent 的规则：根 `AGENTS.md`、`packages/AGENTS.md`、`docs/AGENTS.md`
