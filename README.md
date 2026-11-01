# Reliable Agent Dispatch

我们在 APlus 的实际产品开发中使用多个 AI 代理，分工修改代码、编译，以及在 Mac 和 Windows 的 SketchUp 中验证结果。这个仓库记录其中形成的调度技能，也记录它为什么变成现在这样。

**先读：[这套多机多代理工作流是怎样形成的](docs/workflow-evolution.md)。** 文章展开讲任务状态、讨论结束后的漏接、独立定时任务的投递问题、对话内巡检、多机集中分配，以及主调度怎样与人沟通。

**使用技能：[可靠的多代理调度](skills/reliable-agent-dispatch/SKILL.md)。** 这是从上述经历中提炼的现行步骤；沟通细节见[主调度与人的协作](skills/reliable-agent-dispatch/references/communication.md)。

最初主要由 Claude 做主调度，后来需要让 Codex 接续。负责人在反复使用和纠正的过程中修订规则，也请 Claude 总结过更顺畅的沟通方式。我们希望把这些实际经历分享给 DeepSeek Harness 和其他开发团队，交流哪些行为适合写成技能、哪些需要执行系统支持。

## 解决什么问题

- 执行代理交付了，主调度没有继续接手。
- 原任务负责人已经存在，新调度又重复派出相同工作。
- 代码、测试、目标程序操作、发布和用户验收被混称为“完成”。
- 某台机器或桌面焦点被占用，整批独立任务都停下来。
- 用户需要在不同执行代理之间反复转述要求、追问状态。
- 主调度长时间没有回应，或只回答最新讨论就丢下已有任务。

## 内容与使用

入口是 [skills/reliable-agent-dispatch/SKILL.md](skills/reliable-agent-dispatch/SKILL.md)。它包含接手、分派、交付、核验、接续及结束前检查的完整步骤。

让支持技能的助手读取这个文件；安装位置和发现方式按所用工具的当前官方说明配置。也可以直接要求助手：“读取本仓库的调度技能，在现有授权范围内推进这批任务。”首次使用时先填写技能里的能力表，确认实际可用的代理、恢复、队列和机器操作工具。

配套材料：

- [派单和交付模板](skills/reliable-agent-dispatch/references/templates.md)
- [主调度与人的协作](skills/reliable-agent-dispatch/references/communication.md)
- [宿主工具适配与验证方法](skills/reliable-agent-dispatch/references/host-adaptation.md)
- [虚构示例：交付以后怎样接续](examples/handoff.md)
- [与 DeepSeek Super-agent 公开原型的对照](docs/superagent-comparison.md)

公开内容包括经验文章、技能和模板；实际队列、资源查询与桌面操作工具需要在所用执行环境中配置。当前完成了文档与示例检查，公开技能的跨工具运行效果仍在验证。

## 想交流的问题

1. 换主调度工具以后，怎样验证交付不会漏接？
2. Worker 名额、机器负载、文件写权和桌面焦点怎样分别调度？
3. 怎样识别实质进展，同时避免无限审查和重复测试？

本仓库与 DeepSeek、Anthropic、OpenAI 均无官方关联。项目名称仅用于说明实际使用背景和讨论对象。

## 许可

本仓库原创技能与说明采用 [MIT 许可](LICENSE)。链接指向的第三方内容仍遵守各自许可；本仓库没有复制其实现。
