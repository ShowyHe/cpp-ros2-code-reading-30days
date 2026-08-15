# Day 20：Action执行、反馈和结果

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- README开始状态：Day 19 ✅，Day 20 🟨
- 真实源码基底：资料库 `multi_map_switcher(20260805-105428).zip`
- 今日主题：`execute(goal_handle)`、Feedback、Result、成功/失败/取消终态
- 今日边界：不深入`std::thread`、`mutex`、`atomic`、Executor线程调度和HPA算法细节

## 今日目标

- 从Day 19的`handleAccepted(goal_handle)`继续进入执行阶段。
- 理解为什么`execute()`需要GoalHandle，而不是只需要Goal消息或Pose。
- 理解`goal_handle->get_goal()`取得原始Goal数据。
- 区分Feedback与Result的生命周期。
- 区分`publish_feedback()`、`succeed()`、`abort()`、`canceled()`。
- 区分“接受取消请求”和“真正完成取消”。
- 能独立画出正常、失败、取消三条Action退出路径。

## 执行阶段核心对象

Day 19停在：

```text
Client发送Goal
→ handleGoal
→ ACCEPT
→ ROS2建立GoalHandle
→ handleAccepted(goal_handle)
```

Day 20继续：

```text
handleAccepted(goal_handle)
→ execute(goal_handle)
→ goal_handle->get_goal()
→ 取得原始Goal
→ 执行长任务
```

已确认：

- `execute()`接收GoalHandle，是因为执行阶段不仅需要原始Goal数据，还需要围绕“这一次具体Action任务”发布Feedback、检查取消并设置最终状态。
- `goal_handle->get_goal()`从GoalHandle取得这一次任务对应的原始Goal消息。
- Goal只保存任务输入数据；GoalHandle是管理这一次已接受Action任务的对象。

## Result对象与Action终态

真实代码中：

```cpp
auto result =
  std::make_shared<ExecuteHpaPath::Result>();
```

表示创建一个Result对象并由shared_ptr管理。

填写：

```cpp
result->success = true;
result->message = "OK";
```

只是在填写Result数据，**不会让Action自动进入成功终态**。

真正的Action终态由GoalHandle设置：

```cpp
goal_handle->succeed(result);
goal_handle->abort(result);
goal_handle->canceled(result);
```

对应：

```text
succeed(result)  → SUCCEEDED
abort(result)    → ABORTED
canceled(result) → CANCELED
```

主要纠正：

- 一度把`result->success = true`理解成Action成功状态本身；已纠正为它只是自定义Result消息中的业务字段。
- Action真正进入SUCCEEDED终态的是`goal_handle->succeed(result)`。
- Server决定Action最终状态，Client接收状态和Result，不需要再批准一次。

## Feedback

真实执行过程中：

```cpp
auto feedback =
  std::make_shared<ExecuteHpaPath::Feedback>();

...

goal_handle->publish_feedback(feedback);
```

已确认：

- `feedback`是独立的Feedback消息对象。
- Server填写Feedback字段后，通过GoalHandle把这一次任务的Feedback提交给Action框架。
- Feedback可以在长任务中多次发布，发布后Action继续执行。
- `publish_feedback()`不是终态收尾。

主要纠正：

- 一度把`publish_feedback(feedback)`说成“从GoalHandle里读取Feedback”；已纠正为“通过GoalHandle发布刚刚创建并填写好的Feedback对象”。
- 一度把Feedback/Result说成GoalHandle内部字段；已纠正为：Feedback和Result是独立消息对象，GoalHandle负责管理这次Action并使用这些对象完成发布和收尾。

可接受的简写：

```text
GoalHandle管理这次任务的Feedback / Result / Cancel / 状态
```

但不能写成：

```text
GoalHandle里面有Feedback字段、Result字段
```

## 取消流程

Day 19前半段：

```text
Client发Cancel请求
→ handleCancel()
→ CancelResponse::ACCEPT
```

表示：

```text
Server同意取消请求
```

但此时不等于任务已经取消完成。

Day 20后半段：

```cpp
if (goal_handle->is_canceling()) {
  ...
  goal_handle->canceled(result);
  return;
}
```

表示：

```text
execute()检查到这次Goal正在取消
→ 停止正常执行
→ 填写取消Result
→ canceled(result)
→ Action进入CANCELED终态
→ 当前执行路径结束
```

最终固定区别：

```text
CancelResponse::ACCEPT
= 接受取消请求

canceled(result)
= 取消处理完成，Action进入CANCELED终态
```

## 成功、失败、取消三条退出路径

### 正常

```text
execute()
→ 任务执行成功
→ 填写Result
→ goal_handle->succeed(result)
→ SUCCEEDED
→ ROS2 Action框架把最终状态和Result返回Client
```

### 失败

```text
execute()
→ 执行没有成功
→ is_canceling() == false
→ 填写失败Result
→ goal_handle->abort(result)
→ ABORTED
→ ROS2 Action框架把最终状态和Result返回Client
```

### 取消

```text
Client发Cancel
→ handleCancel()返回ACCEPT
→ execute()继续运行到取消检查点
→ is_canceling() == true
→ 停止正常执行
→ 填写取消Result
→ goal_handle->canceled(result)
→ CANCELED
→ ROS2 Action框架把最终状态和Result返回Client
```

关键判断：当某个执行函数返回失败时，不能直接一律`abort()`，还要检查`is_canceling()`，因为“没有正常成功”可能是真失败，也可能是用户取消导致的退出。

## Action通信补充

为补齐Server到Client链路，本日补充理解：

```text
Server创建并填写Feedback
→ goal_handle->publish_feedback(feedback)
→ Action框架负责传输
→ Client收到中间Feedback
```

以及：

```text
Server创建并填写Result
→ goal_handle->succeed/abort/canceled(result)
→ Server侧确定Action终态
→ Action框架负责把最终状态和Result返回Client
```

这一补充仅用于理解业务层接口，不在本日继续深入`rclcpp_action`、`rcl_action`、RMW或DDS内部实现。

## 正式考试

### 第一轮：68/100，未直接通过

主要问题：

1. 把GoalHandle描述成“里面包含Feedback、Result字段”。
2. 对`handleCancel()`返回ACCEPT与`canceled(result)`两个阶段区分不够准确。
3. 对`result->success`业务字段和Action的SUCCEEDED终态区分需要进一步固定。

根据全局规则，虽然部分流程题方向正确，但存在关键概念错误，因此第一轮不判通过。

### 定向补测

最终能够准确回答：

```text
Feedback和Result是独立消息对象；
GoalHandle管理这一次Action任务，并通过成员函数发布Feedback、检查取消和设置最终状态。
```

以及：

```text
CancelResponse::ACCEPT
= 接受取消请求

canceled(result)
= 完成取消并进入CANCELED终态
```

定向补测通过，关键概念错误已纠正。

## 今日已掌握

- `handleAccepted(goal_handle) → execute(goal_handle)`。
- GoalHandle与Goal在执行阶段的职责区别。
- `goal_handle->get_goal()`。
- Feedback对象创建、字段填写和`publish_feedback()`。
- Result对象创建、字段填写和三种终态收尾。
- `result->success`与Action SUCCEEDED终态不是同一层概念。
- `succeed / abort / canceled`三条终态。
- `is_canceling()`是检查取消状态，不是主动发送Cancel。
- “接受取消请求”与“取消完成”的两阶段区别。
- 正常、失败、取消三条完整退出链。

## 仍需注意

- 不要把Feedback或Result称为GoalHandle内部字段。
- 不要把`result->success = true`当成Action已经SUCCEEDED。
- 不要把`execute()`说成“返回三个状态”；它是`void`，执行过程中根据情况调用GoalHandle的终态函数。
- 不要把`is_canceling()`说成正在接收Cancel消息；它检查的是这次Goal当前是否处于取消流程。
- 不要把Cancel请求ACCEPT和最终CANCELED混成同一步。

## 当前状态

- 基础讲义：完成。
- 真实源码阅读：完成。
- 正常/失败/取消三条链：完成。
- 正式考试第一轮：68/100，因关键概念错误未通过。
- 定向补测：通过。
- 关键概念错误：无未纠正项。
- 是否正式通过：是。
- Day 20状态：✅ 完成。

## 下一窗口交接

- 下一日：Day 21 第三周ROS2修改与测试。
- Day 21不再新增主要ROS2语法，而是综合Node、参数、Topic、Service、TF、Future和Action阅读能力完成一次低风险接口级小修改。
- 修改前必须先列出受影响节点、接口、调用方、配置、启动方式和验证方法。
- 仅编译通过不能判定修改有效，必须检查节点启动、接口可见、真实请求/消息数据和异常路径。
- 不选择同时改多个协议或需要重构线程模型的任务。
