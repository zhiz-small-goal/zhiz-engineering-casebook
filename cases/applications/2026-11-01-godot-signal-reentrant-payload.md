---
id: ENG-20261101-001
date: 2026-11-01
updated: 2026-11-01
status: resolved_verified
domain: applications
category: event-signals
systems:
  - Godot 4.6.1 Stable (GDScript)
  - Windows
tags:
  - godot
  - signal
  - reentrancy
  - regression-test
---

# Godot NPC 已经成功出门，第二个信号监听器却漏记事件

## 遇到了什么？

宿舍攻防原型里，NPC 抢床失败时会开门、实际移动到房外、关门、重新选床。首次加入真实物理帧回归后，这些动作全部通过，但两条“exit_reached 信号必须恰好出现一次”的断言失败。不能因此断言 NPC 没走出门，因为主游戏已经完成了关门。

## 环境

Windows、Godot 4.6.1.stable.official.14d19694e、GDScript（非 .NET），独立 headless Godot 进程执行测试。Main 和测试记录器均连接 NPC 的 exit_reached(actor_id, door_cell) 信号。

## 排查过程与证据

第一次用户本机完整运行：

    Godot gameplay regressions: FAIL
    Cases: 62 passed; 2 failed
    Suite 01-gameplay: PASS (45 passed, 0 failed)
    Suite 02-physical-flow: FAIL (17 passed, 2 failed)
    FAIL: 物理单人：真实移动恰好触发一次 exit_reached
    FAIL: 物理多人：两个 NPC 各自只产生一次真实出门信号

查看源码发现：

1. NPC 原本通过 exit_reached.emit(actor_id, _exit_door_cell) 直接传递自身成员变量。
2. Main 在 NPC 创建时先注册监听器，测试在场景创建后才注册事件记录器。
3. Main 处理第一个监听回调时，会同步重新选房，最终清除 NPC 的 _exit_door_cell 成员。
4. Godot 上游公开 issue #88885 描述了 Signal.emit 中成员字段被同步监听回调修改时，后续监听器所收到的参数可能变化的问题。

采用最小修复：先将门格复制到局部变量 finished_door_cell，再调用 exit_reached.emit(actor_id, finished_door_cell)。没有删除或放宽“恰好一次”断言。另外为同类状态信号的参数采用局部快照，并增加原始事件参数诊断输出。

第二次用户本机完整运行：

    Godot gameplay regressions: PASS
    Cases: 64 passed; 0 failed
    Suite 01-gameplay: PASS (45 passed, 0 failed)
    Suite 02-physical-flow: PASS (19 passed, 0 failed)
    PASS: 物理单人：真实移动恰好触发一次 exit_reached
    PASS: 物理多人：两个 NPC 各自只产生一次真实出门信号

## 现在能确定什么？

- **事实**：信号快照改动之前两项断言失败，改动之后同版本 Godot 两项均通过；单人与双人的实际撤离、关门及重新选床断言也通过。
- **可信推断**：Signal.emit 的可重入监听使后续监听器读取了被前一个回调修改的成员值；与 Godot 上游 issue 的机制一致。
- **未知**：没有通过引擎级调试器追踪每一个值的存储，所以不能将该推断说成已完成底层机制证明，也不能保证所有 Godot 版本一致。

案例状态 resolved_verified 表示**业务层回归故障修复后已复测通过**，不表示官方引擎问题获得确认或修复。

## 下次怎么做？

当动作本身已经完成，但多个监听者记录的事件次数或参数不一致时，优先检查监听注册顺序、同步回调修改成员变量的代码、各监听者实际收到的参数。对可能被同步回调改变的事件数据，先复制为局部快照再发射，保留严格的回归断言验证效果。

## 参考

- [Godot issue #88885](https://github.com/godotengine/godot/issues/88885)
