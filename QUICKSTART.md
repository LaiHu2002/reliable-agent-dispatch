# 核心速读：五条核心

[English](README.en.md#five-core-principles)

完整经过、数字与限制见[第三版主文](docs/scripts-and-self-heal.md)和[第二版主文](docs/coordination-structure.md)，执行细则见[调度技能](skills/reliable-agent-dispatch/SKILL.md)。

## 1. 一个独立问题，一个执行者做到底

同一线程完成定位、修复、相关测试、必要构建、实机验收、保存回读和清理，不按阶段或机器换人。需要继续时恢复原线程；只有工具证实原线程恢复不了，才登记继任。见[现在的结构](docs/coordination-structure.md#四现在的结构)。

## 2. 限制性规矩要能指到用户原话

每道门槛问两个问题：能不能对到用户原话？护不护着硬边界？两个都答不上的，删掉或降成只记录。新增门槛和产品里加路障一样，要用户批准。默认“做不到”之前，先实际试一次。见[卡点从哪来](docs/scripts-and-self-heal.md#一卡点从哪来)。

## 3. 能写成脚本的规矩写成脚本

改状态的命令缺必填项就拒收，定时检查发现没做就报红；条文和脚本冲突以脚本为准。补位、数卡死、分配机器、修已知故障都交给脚本，经理只剩一层补位员。见[规矩写成脚本](docs/scripts-and-self-heal.md#二规矩写成脚本)和[脚本合同](skills/reliable-agent-dispatch/references/scripts.md)。

## 4. 机器按实时压力放行，出了事机器自己兜底

先放进去，平时不排队；预约跑起来后按实测纠正；持续严重过载，才把最新放进去的任务退回池子。已知故障由自愈手册直接修，新规则先观察再转正；失控进程由本机看门狗掐掉，不靠远程。见[机器怎么分配](docs/scripts-and-self-heal.md#三机器怎么分配)和[出了事，机器自己兜底](docs/scripts-and-self-heal.md#四出了事机器自己兜底)。

## 5. 每天看数，摩擦记账，借方法先试跑

看每个问题用了几个线程、中途来回几次、管理类 token 占比；我们的目标分别是 1–2 个、不超过 3 次、约 30%。要用户介入、空等、重复手工、产出作废都记进摩擦账本，优先用脚本消掉。别人的工作方法先对照前提，小批试跑一两天，变好才推广。见[摩擦账本](docs/scripts-and-self-heal.md#五摩擦账本)和[第二版的检查清单](docs/coordination-structure.md#附借别人工作方法前的检查清单)。
