# 30天 C++ / ROS2 代码阅读训练

以真实的 `multi_map_switcher / HPA` 功能包为代码基底，训练通用的 C++ 与 ROS2 工程代码阅读、问题定位、修改方案设计和风险审查能力。

> HPA 是贯穿全程的真实代码教材，不是学习目标。遇到算法细节，优先确认输入、输出、调用关系和副作用，不在当前阶段展开算法推导。

> 从 Day 21 起进一步明确：实际修改源码、编译、启动、部署和现场实测不作为每日通过的硬性条件；本仓库重点训练真实源码阅读、工程判断、最小修改方案和验证设计。若实际执行，则作为可选扩展记录真实证据。

## 新学习窗口启动要求

新的GPT窗口不得只根据本README的一行进度直接开始教学。必须依次读取：

1. [`AGENTS.md`](AGENTS.md)：全局教学、考试、评分、Git和交接规则；
2. 本README：当前进度；
3. [`docs/30-day-plan.md`](docs/30-day-plan.md)：当前日目标、边界、源码范围和衔接；
4. 最近完成日与当前进行日的`progress/dayXX.md`：实际成绩、错误、作废题和薄弱点；
5. 当天涉及的真实源码或学习者提供的代码材料。

正式教学前必须先报告：

```text
仓库与目标分支
README显示的当前日
最近完成记录
今天必须掌握
今天禁止扩展
发现的进度或分支冲突
```

如果README与`progress`或其他分支冲突，不得猜测，也不得擅自合并；应先核对并报告。

## 最终目标

完成30天训练后，应能够：

- 快速定位陌生 C++ ROS2 包的入口、节点、接口、配置和主要类。
- 判断变量类型、对象关系、指针所有权、复制行为和生命周期。
- 跟踪函数调用链、控制流和数据流。
- 阅读 Topic、Service、Action、Timer、TF、Future 和异步回调。
- 根据真实源码定位问题、判断影响范围并设计最小修改方案。
- 设计正常、失败、边界和回归验证矩阵，并区分“验证方案”和“真实运行证据”。
- 面对陌生专业算法时，先看懂工程边界，再补充领域知识。

## 时间要求

- 工作日：每天约 2.5 小时
- 周六：约 3.5 小时
- 周日复盘：约 2 小时
- 总投入：约 75～80 小时

## 当前进度

| 天数 | 状态 | 主题 | 记录 |
|---|---|---|---|
| Day 01 | ✅ | ROS2包结构与编译入口 | [学习记录](progress/day01.md) |
| Day 02 | ✅ | 类型、变量、函数和返回值 | [学习记录](progress/day02.md) |
| Day 03 | ✅ | 声明、定义、命名空间和作用域 | [学习记录](progress/day03.md) |
| Day 04 | ✅ | const、值传递和引用传递 | [学习记录](progress/day04.md) |
| Day 05 | ✅ | class、构造函数和成员变量 | [学习记录](progress/day05.md) |
| Day 06 | ✅ | 指针、对象和智能指针 | [学习记录](progress/day06.md) |
| Day 07 | ✅ | 第一周复盘与闭卷测试 | [学习记录](progress/day07.md) |
| Day 08 | ✅ | struct、vector 和 string | [学习记录](progress/day08.md) |
| Day 09 | ✅ | map、set 和 unordered_map | [学习记录](progress/day09.md) |
| Day 10 | ✅ | 循环、迭代器和索引访问 | [学习记录](progress/day10.md) |
| Day 11 | ✅ | enum、条件分支和提前返回 | [学习记录](progress/day11.md) |
| Day 12 | ✅ | 文件读取、YAML 和错误处理 | [学习记录](progress/day12.md) |
| Day 13 | 🟨 | 跨文件调用链 | [学习记录](progress/day13.md) |
| Day 14 | ⬜ | 第二周小修改与复盘 | `progress/day14.md` |
| Day 15 | ✅ | Node、参数和日志 | [学习记录](progress/day15.md) |
| Day 16 | ✅ | Service、Request 和 Response | [学习记录](progress/day16.md) |
| Day 17 | ✅ | Publisher、Subscriber 和 Timer | [学习记录](progress/day17.md) |
| Day 18 | ✅ | TF、Client、Future 和异步回调 | [学习记录](progress/day18.md) |
| Day 19 | ✅ | Action Server 基本结构 | [学习记录](progress/day19.md) |
| Day 20 | ✅ | Action执行、反馈和结果 | [学习记录](progress/day20.md) |
| Day 21 | ✅ | 第三周ROS2接口影响分析 | [学习记录](progress/day21.md) |
| Day 22 | 🟨 | thread、mutex 和 atomic | `progress/day22.md` |
| Day 23 | ⬜ | 状态、重试、超时和取消 | `progress/day23.md` |
| Day 24 | ⬜ | lambda捕获和shared_ptr生命周期 | `progress/day24.md` |
| Day 25 | ⬜ | 大函数拆解方法 | `progress/day25.md` |
| Day 26 | ⬜ | 编译错误、日志和问题定位 | `progress/day26.md` |
| Day 27 | ⬜ | 真实问题定位与最小修改设计 | `progress/day27.md` |
| Day 28 | ⬜ | 回归风险与测试矩阵设计 | `progress/day28.md` |
| Day 29 | ⬜ | 陌生ROS2 C++包独立阅读 | `progress/day29.md` |
| Day 30 | ⬜ | 陌生代码综合审查 | `progress/day30.md` |

状态统一使用：`⬜ 未开始`、`🟨 进行中`、`✅ 完成`、`🔁 需要重学`。

## 文档

- [完整30天计划](docs/30-day-plan.md)
- [每日学习记录模板](templates/day-template.md)
- [每周复盘模板](templates/weekly-review-template.md)

## 每日Git提交格式

```text
day-01: analyze ROS2 package structure
day-21: analyze ROS2 interface impact
day-30: review unfamiliar ROS2 change
```

## AI使用顺序

```text
自己先读并写出判断
→ 让AI检查判断
→ 回到原代码核实
→ 设计或审核最小修改方案
→ 设计验证矩阵
→ 明确哪些结论已有源码证据、哪些仍需真实运行证据
```

如果当天明确选择做真实实操，再追加：

```text
检查diff
→ 编译/启动/实测
→ 记录真实证据
```

禁止直接让 AI 代替阅读、直接生成整个项目结论，或把尚未实际执行的验证方案写成“已经验证通过”。
