# 与 DeepSeek Super-agent 公开原型的对照

两套流程在持续负责人、独立执行、可靠交付和减少人工盯单方面相近；主调度职责和宿主程序保障不同。

对官方原型一列的依据，仍是本仓库十一月初首次整理时阅读的公开文档与提示词，未在本次更新中重新测评其最新实现。当时未运行原型，未读到可访问的官方架构文章全文；所读 README 也说明公开原型与内部版本、文章所述实现不完全相同，部分文章设计尚未实现。下表只对照当时所读材料，不能推广为其当前完整系统的结论。

| 方面 | 官方公开原型 | 本技能 |
| --- | --- | --- |
| 主调度 | Router 负责接单、路由与沟通，Worker 自行调查与实现；Router 提示词要求不添加实现方案 | 当前按任务设经理，经理核本组意图、交付和验收；主调度负责全局与用户沟通，避免逐条收全部执行交付 |
| 执行归属 | 每个任务有独立会话与目录，后续讨论按订阅进入对应 Worker | 同题优先回原负责人，接手时恢复已有记录并核实际执行状态 |
| 持久交付 | 用数据库记录任务和待执行动作，通知与投递由宿主执行 | 规定先保存后通知、落实接续后确认；没有附带数据库或队列实现 |
| 异步恢复 | 后台工作完成事件可唤醒 Worker | 要求识别实际启动与消息的区别；自动恢复能力须单独验证 |
| 多机与桌面 | 所读文档不能证明文章设想的完整多机系统已实现 | 分开核算代理名额、机器资源、文件写权和桌面焦点，实际能力由外部工具提供 |
| 真实完成 | Worker 按其任务和工具验证结果 | 将源码、构建、所需平台操作、集成、发布和用户验收分别记录 |

当前最想交流：跨主调度迁移时哪些规则应固化进宿主；怎样判断执行者的实质进展；怎样在真实桌面验证期间接续其他独立任务。

十一月新增的分层调度、资源统筹与集成车道见 [manager-hierarchy.md](../skills/reliable-agent-dispatch/references/manager-hierarchy.md)。这些是本团队的实践选择，不构成对其他系统有无相同能力的判断。

## 官方来源

- [README 与实现边界](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/README.md)
- [工作理念](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/000-philosophy.md)
- [领域模型](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/001-domain.md)
- [执行与异步恢复](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/003-execution.md)
- [实现架构](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/docs/007-architecture.md)
- [Router 提示词](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/prompts/orchestrator.md)
- [Worker 提示词](https://github.com/deepseek-ai/dsh-superagent-prototype/blob/main/prompts/worker.md)
