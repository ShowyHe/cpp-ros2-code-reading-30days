# Day 19：Action Server 基本结构

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- README开始状态：Day 18 ✅，Day 19 🟨
- 真实源码基底：资料库 `multi_map_switcher(20260805-105428).zip`
- 今日主题：Action类型、Goal、Feedback、Result、Goal Handle、goal/cancel/accepted三个回调
- 今日边界：不详细追踪完整`execute()`循环、Feedback发布频率、最终Result收尾和线程并发细节；这些进入Day 20及后续课程

## 今日目标

- 理解为什么长任务使用Action而不是普通Service。
- 区分Action接口中的Goal、Result、Feedback。
- 区分Goal消息和GoalHandle管理对象。
- 从Action Server创建点找到`handleGoal`、`handleCancel`、`handleAccepted`三个核心回调。
- 理解三个回调分别由什么事件触发、各自负责什么。
- 画出Goal从到达、被接受到获得GoalHandle的基本决策流程。

## Action基础

Service适合相对短的请求—响应任务：

```text
Request
→ Server callback
→ Response
```

Action用于生命周期更长、执行过程中需要反馈并可能取消的任务：

```text
Goal
→ 长时间执行
→ Feedback...
→ Result

执行期间还可能收到Cancel请求
```

Action接口由三段消息组成：

```text
Goal
---
Result
---
Feedback
```

其中：

- Goal：Client希望Server完成什么的输入数据。
- Result：整个Action最终结束后的结果数据。
- Feedback：任务未结束时的中间进度或状态数据。

## 真实Action接口

`action/ExecuteHpaPath.action`：

```text
# Goal
geometry_msgs/PoseStamped goal
---
# Result
bool success
string message
int32 maps_switched
---
# Feedback
int32 current_waypoint_index
int32 total_waypoints
string current_chunk
geometry_msgs/PoseStamped current_pose
```

已确认：

- `geometry_msgs/PoseStamped goal`属于Goal，表示本次Action的目标Pose数据。
- `success / message / maps_switched`属于Result，用于描述最终结果。
- `current_waypoint_index / total_waypoints / current_chunk / current_pose`属于Feedback，用于描述长任务当前进度和状态。
- Feedback的直接职责是把执行过程状态提供给Client；Client可以据此显示进度或决定后续操作，但Feedback本身不是“自动调整任务”的机制。

## Goal与GoalHandle

今日最重要的对象区分：

```text
Goal
≠
GoalHandle
```

### Goal

```text
Goal
→ Action接口中的任务输入消息
→ 描述“这次任务要做什么”
```

### GoalHandle

真实类型关系：

```cpp
using GoalHandleExecute =
  rclcpp_action::ServerGoalHandle<ExecuteHpaPath>;
```

其含义是：

```text
GoalHandle
→ Server管理“某一次具体已接受Goal”的句柄/上下文对象
```

需要特别区分名称：

```text
handleGoal
= handle + Goal
= 函数名，handle是动词“处理”

GoalHandle
= Goal + Handle
= 类型/对象名，Handle是名词“句柄”
```

类似：

```text
openFile   → 函数
FileHandle → 文件句柄对象
```

学习过程中曾多次把`handleGoal`和`GoalHandle`混写，已通过定向补测纠正。

## Action Server创建点

真实源码：`src/hpa/hpa_executor_node.cpp`

```cpp
action_server_ =
  rclcpp_action::create_server<ExecuteHpaPath>(
    this,
    "~/execute_path",
    std::bind(
      &HpaExecutorNode::handleGoal,
      this,
      std::placeholders::_1,
      std::placeholders::_2),
    std::bind(
      &HpaExecutorNode::handleCancel,
      this,
      std::placeholders::_1),
    std::bind(
      &HpaExecutorNode::handleAccepted,
      this,
      std::placeholders::_1));
```

已确认：

- `create_server()`在这里创建Action Server并注册三个callback。
- 注册callback不等于立即执行callback。
- callback由后续对应ROS2 Action事件触发，并由Executor调度执行。

## 三个核心callback

### `handleGoal`

触发事件：

```text
Client发送新Goal
↓
Action Server收到新Goal请求
↓
Executor调度
↓
handleGoal(...)
```

核心职责：

```text
决定这个新Goal是否接受
```

例如可返回接受或拒绝。若Goal被拒绝，不会进入`handleAccepted()`。

### `handleAccepted`

前提：Goal已经被接受。

```text
handleGoal返回ACCEPT
↓
ROS2 Action框架建立该Goal对应的GoalHandle
↓
handleAccepted(goal_handle)
```

核心职责：

```text
拿到这次已接受任务的GoalHandle
→ 准备把它交给后续长任务执行逻辑
```

重要纠正：GoalHandle不是`handleGoal()`函数自己生成的，而是在Goal被接受后由ROS2 Action框架建立并交给`handleAccepted()`。

### `handleCancel`

触发事件：

```text
已存在某次Action任务
↓
Client发送Cancel请求
↓
Action Server收到Cancel请求
↓
Executor调度
↓
handleCancel(goal_handle)
```

核心职责：

```text
决定是否接受这次取消请求
```

必须区分：

```text
接受Cancel请求
≠
长任务已经真正停止并完成canceled收尾
```

真正停止执行、设置终态和Result属于Day 20。

## 三个callback的事件关系

三个callback不是固定按：

```text
handleGoal
→ handleCancel
→ handleAccepted
```

顺序执行。

正确事件骨架：

```text
                 新Goal
                    ↓
               handleGoal
                /      \
             REJECT    ACCEPT
                         ↓
                 ROS2建立GoalHandle
                         ↓
                  handleAccepted
                         ↓
                    长任务执行
                         │
             如果Client发送Cancel
                         ↓
                   handleCancel
```

如果Client从不取消，则`handleCancel()`可能一次也不会执行。

## 独立回答与纠正

### 第一批源码问题

学习者能够正确判断：

- Goal字段属于Goal。
- `success / message / maps_switched`属于最终Result。
- waypoint/chunk/current_pose字段属于Feedback。
- `create_server()`注册`handleGoal / handleCancel / handleAccepted`。
- 三个callback不会在`create_server()`时立即执行。

术语纠正：

- Feedback的直接作用是报告过程状态，不应直接描述为“方便Client随时调整任务”。
- `handleAccepted`应描述为“Goal已经接受后调用”，避免说成“收到Goal”而与`handleGoal`混淆。

### GoalHandle定向理解

学习者确认：

```text
handleGoal
→ callback函数
→ 新Goal来了，决定接不接

GoalHandle
→ 管理对象
→ 管理某一次已接受Goal

handleAccepted
→ callback函数
→ Goal接受后拿到GoalHandle

handleCancel
→ callback函数
→ Cancel请求来了，决定是否接受取消
```

## 正式考试

第一轮：81/100。

分项：

1. Goal与GoalHandle：16/20。
2. 新Goal与`handleGoal`：20/20。
3. `handleAccepted`前提与GoalHandle：17/20。
4. `handleCancel`与取消请求：16/20。
5. 完整流程：12/20。

虽然总分达到80，但第5题再次把`handleGoal`函数和`GoalHandle`对象混写，属于当天核心概念错误，因此按规则暂不判通过。

### 定向补测

题目：

```text
Client发送Goal
↓
__________()        ← callback函数
↓
返回ACCEPT
↓
ROS2建立 __________ ← 管理对象
↓
__________(goal_handle) ← callback函数
```

学习者回答：

```text
handleGoal
GoalHandle
handleAccepted
```

全部正确，关键概念错误已修正。

最终固定模型：

```text
Client发送Goal
↓
handleGoal()
↓
返回ACCEPT
↓
ROS2建立GoalHandle
↓
handleAccepted(goal_handle)
↓
准备进入长任务执行

执行过程中Client发送Cancel
↓
handleCancel(goal_handle)
```

## 今日已掌握

- Service与Action在任务生命周期上的基本差异。
- Action的Goal、Result、Feedback三段数据。
- Goal消息与GoalHandle管理对象的区别。
- `handleGoal`函数名与`GoalHandle`类型名的英文结构区别。
- Action Server创建时三个核心callback的注册。
- `handleGoal`由新Goal事件触发并决定接受/拒绝。
- Goal接受后由ROS2框架建立GoalHandle，再调用`handleAccepted`。
- `handleCancel`由Cancel请求触发并决定是否接受取消请求。
- 注册顺序不代表三个callback固定依次执行。
- 接受Cancel请求不等于任务已经完成取消收尾。

## 仍需注意

- 不要把`handleGoal`函数和`GoalHandle`对象混为一谈。
- 不要说`handleGoal()`自己生成GoalHandle；是Goal被接受后由ROS2 Action框架建立。
- 不要把`handleAccepted`说成“收到新Goal”；新Goal先进入`handleGoal`。
- 不要把接受Cancel请求说成任务已经取消完成。
- Day19不继续深入`execute()`、Feedback发布、Result填写和`succeed/abort/canceled`。

## 当前状态

- 基础讲义：完成。
- Action接口三段数据：完成。
- Action Server创建点：完成。
- Goal/GoalHandle对象关系：完成。
- 三个核心callback触发关系：完成。
- 正式考试：第一轮81/100，但存在关键概念错误；定向补测通过后关键错误已纠正。
- 是否正式通过：是。
- Day 19状态：✅ 完成。

## 下一窗口交接

- 下一日：Day 20 Action执行、Feedback、Result和取消收尾。
- 从Day19的`handleAccepted(goal_handle)`继续进入被接受Goal的执行函数。
- 跟踪GoalHandle如何取得Goal、Feedback如何创建和发布、Result如何填写，以及正常成功、失败、取消分别如何结束。
- 必须继续区分“Cancel请求被接受”与“任务真正进入canceled终态”。
- Day20才开始阅读`execute()`主链；线程并发、mutex和竞态问题仍不在本日深入，留到Week 4。
