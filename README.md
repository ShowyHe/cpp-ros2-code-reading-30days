# 30天 C++ / ROS2 代码阅读训练

以真实的 `multi_map_switcher / HPA` 功能包为代码基底，训练通用的 C++ 与 ROS2 工程代码阅读、修改和验证能力。

> HPA 是贯穿全程的真实代码教材，不是学习目标。遇到算法细节，优先确认输入、输出、调用关系和副作用，不在当前阶段展开算法推导。

## 最终目标

完成30天训练后，应能够：

- 快速定位陌生 C++ ROS2 包的入口、节点、接口、配置和主要类。
- 判断变量类型、对象关系、指针所有权、复制行为和生命周期。
- 跟踪函数调用链、控制流和数据流。
- 阅读 Topic、Service、Action、Timer、TF、Future 和异步回调。
- 借助 AI 完成小到中等修改，并独立审核、编译、测试和验证。
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
| Day 12 | 🟨 | 文件读取、YAML 和错误处理 | `progress/day12.md` |
| Day 13 | ⬜ | 跨文件调用链 | `progress/day13.md` |
| Day 14 | ⬜ | 第二周小修改与复盘 | `progress/day14.md` |
| Day 15 | ⬜ | Node、参数和日志 | `progress/day15.md` |
| Day 16 | ⬜ | Service、Request 和 Response | `progress/day16.md` |
| Day 17 | ⬜ | Publisher、Subscriber 和 Timer | `progress/day17.md` |
| Day 18 | ⬜ | TF、Client、Future 和异步回调 | `progress/day18.md` |
| Day 19 | ⬜ | Action Server基本结构 | `progress/day19.md` |
| Day 20 | ⬜ | Action执行、反馈和结果 | `progress/day20.md` |
| Day 21 | ⬜ | 第三周测试与ROS2小修改 | `progress/day21.md` |
| Day 22 | ⬜ | thread、mutex 和 atomic | `progress/day22.md` |
| Day 23 | ⬜ | 状态、重试、超时和取消 | `progress/day23.md` |
| Day 24 | ⬜ | lambda捕获和shared_ptr生命周期 | `progress/day24.md` |
| Day 25 | ⬜ | 大函数拆解方法 | `progress/day25.md` |
| Day 26 | ⬜ | 编译错误、日志和问题定位 | `progress/day26.md` |
| Day 27 | ⬜ | HPA真实问题修改 | `progress/day27.md` |
| Day 28 | ⬜ | 回归验证和第四周复盘 | `progress/day28.md` |
| Day 29 | ⬜ | 陌生ROS2 C++包阅读考试 | `progress/day29.md` |
| Day 30 | ⬜ | 陌生包独立修改考试 | `progress/day30.md` |

状态统一使用：`⬜ 未开始`、`🟨 进行中`、`✅ 完成`、`🔁 需要重学`。

## 文档

- [完整30天计划](docs/30-day-plan.md)
- [每日学习记录模板](templates/day-template.md)
- [每周复盘模板](templates/weekly-review-template.md)

## 每日Git提交格式

```text
day-01: analyze ROS2 package structure
day-14: add MapSplitter input validation
day-30: complete unfamiliar package modification
```

## AI使用顺序

```text
自己先读并写出判断
→ 让AI检查判断
→ 回到原代码核实
→ 审核修改diff
→ 自己编译和实测
→ 记录验证证据
```

禁止直接让 AI 代替阅读、直接生成整个结论，或在未检查 diff 和测试结果时接受修改。
