# Reliable Agent Dispatch

我们在 APlus 的实际产品开发中使用多个 AI 代理，分工修改代码、编译，以及在 Mac 和 Windows 的 SketchUp 中验证结果。这个仓库记录其中形成的调度技能，也记录它为什么变成现在这样。

**先读：[这套多机多代理工作流是怎样形成的](docs/workflow-evolution.md)。** 文章展开讲任务状态、讨论结束后的漏接、独立定时任务的投递问题、对话内巡检、多机集中分配，以及主调度怎样与人沟通。

**使用技能：[可靠的多代理调度](skills/reliable-agent-dispatch/SKILL.md)。** 这是从上述经历中提炼的现行步骤；沟通细节见[主调度与人的协作](skills/reliable-agent-dispatch/references/communication.md)。

**2026 年 11 月更新：** 原来十几个智能体、约十条执行线就容易中间脱节。采用分层管理后，负责人观察到，可有效协作规模从十来条扩到约五十条，同时运行约五十个智能体仍能保持相当高效的协作。我们把日常接续交给按任务设置的经理，又把集成按层和文件分开推进；同时补上推广终点、轻量静态棘轮、成熟库优先，以及缺陷的止血线与类治理线。新文章从[主调度的处理带宽](docs/workflow-evolution.md#8-从提醒主调度到分担主调度的处理带宽)讲起，保留了试点停住、提前宣布完成等失败与修正。

最初主要由 Claude 做主调度，后来需要让 Codex 接续。负责人在反复使用和纠正的过程中修订规则，也请 Claude 总结过更顺畅的沟通方式。我们希望把这些实际经历分享给 DeepSeek Harness 和其他开发团队，交流哪些行为适合写成技能、哪些需要执行系统支持。

## 解决什么问题

- 执行代理交付了，主调度没有继续接手。
- 原任务负责人已经存在，新调度又重复派出相同工作。
- 代码、测试、目标程序操作、发布和用户验收被混称为“完成”。
- 某台机器或桌面焦点被占用，整批独立任务都停下来。
- 用户需要在不同执行代理之间反复转述要求、追问状态。
- 主调度长时间没有回应，或只回答最新讨论就丢下已有任务。
- 执行者增加以后，交付和集成都排在同一个处理入口。
- 单点修好后，同类问题继续出现；大类推广又容易停得过早或无限延伸。

适用规模从单一小任务到多组并行任务。单一问题保留最小闭环，不为了使用技能设置经理；有多条独立执行线、交付持续积压时，再按真实负荷分组。从十来条扩到约五十条，是作者对实际有效协作规模的观察，约为原来的五倍，不代表每单位时间完成了五倍工作，也不是通用容量上限。每位经理十条未结执行线是本次实测配置，等待验证与外部输入也算在内，需要按宿主工具、任务复杂度和接续速度调整。

## 内容与使用

入口是 [skills/reliable-agent-dispatch/SKILL.md](skills/reliable-agent-dispatch/SKILL.md)。它包含接手、分派、交付、核验、接续及结束前检查的完整步骤。

让支持技能的助手读取这个文件；安装位置和发现方式按所用工具的当前官方说明配置。也可以直接要求助手：“读取本仓库的调度技能，在现有授权范围内推进这批任务。”首次使用时按[宿主适配说明](skills/reliable-agent-dispatch/references/host-adaptation.md)确认实际可用的代理、恢复、队列和机器操作工具。

配套材料：

- [派单和交付模板](skills/reliable-agent-dispatch/references/templates.md)
- [主调度与人的协作](skills/reliable-agent-dispatch/references/communication.md)
- [宿主工具适配与验证方法](skills/reliable-agent-dispatch/references/host-adaptation.md)
- [动态经理制、资源分配与分层集成](skills/reliable-agent-dispatch/references/manager-hierarchy.md)
- [止血线、类治理线与推广终点](skills/reliable-agent-dispatch/references/bug-by-class.md)
- [成熟库与通用做法的评估提议](skills/reliable-agent-dispatch/references/mature-libraries.md)
- [虚构示例：交付以后怎样接续](examples/handoff.md)
- [与 DeepSeek Super-agent 公开原型的对照](docs/superagent-comparison.md)

公开内容包括经验文章、技能和模板。这套技能不是执行框架；实际队列、资源查询、消息投递、恢复执行与桌面操作都需要宿主工具实现并验证。文档可以描述接续责任，不能保证停止对话或重启应用后自动唤醒。项目内的使用观察与公开技能的跨工具运行效果分别说明，后者仍在验证。

## 想交流的问题

1. 换主调度工具以后，怎样验证交付不会漏接？
2. Worker 名额、机器负载、文件写权和桌面焦点怎样分别调度？
3. 怎样识别实质进展，同时避免无限审查和重复测试？

本仓库与 DeepSeek、Anthropic、OpenAI 均无官方关联。项目名称仅用于说明实际使用背景和讨论对象。

## 许可

本仓库原创技能与说明采用 [MIT 许可](LICENSE)。链接指向的第三方内容仍遵守各自许可；本仓库没有复制其实现。
