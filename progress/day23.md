# Day 23：状态、重试、超时和取消

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- README开始状态：Day 22 ✅，Day 23 🟨
- 今日主题：显式状态、状态转移、重试计数、冷却时间、超时检测、取消请求和Action终态
- 今日边界：不要求实际制造超时/取消场景，不强行重构大型状态机，不提前展开Day24的lambda捕获和`shared_ptr/weak_ptr`生命周期

## 今日目标

- 能把真实控制流拆成“当前状态/结果 → 事件 → 条件 → 动作 → 下一状态/终态”。
- 区分普通失败、可重试失败、需要replan的失败和取消。
- 理解重试次数必须从初值、比较条件、自增位置和调用方式证明，不能只根据变量名猜总次数。
- 理解cooldown与timeout职责不同。
- 理解子任务timeout不等于整个上层Action立即失败。
- 区分`is_canceling()`与`canceled(result)`。
- 判断`succeed/abort/canceled`之后是否真正形成终态闭合，必须继续追`return/break/控制流出口`。

## 完整基础讲义范围

本日基础讲义覆盖：

- 状态、事件、条件、动作、状态转移和终态。
- retry与attempt语义差异。
- 最大重试次数和无限重试风险。
- cooldown与timeout区别。
- `std::chrono::steady_clock`用于经过时间判断的基本用途。
- Action中的取消请求检查与取消终态收尾。
- 重复终态、超时后继续执行、取消后继续执行等典型控制流风险。

## 真实源码阅读

真实源码基底：`multi_map_switcher / HPA`中的`HpaExecutorNode`。

### 第1批：取消请求与执行侧响应

源码1：`HpaExecutorNode::handleCancel()`及其Action Server注册点。

确认`handleCancel()`之所以是Action Server cancel callback，不是因为函数名或`CancelResponse`返回类型本身，而是因为`create_server<ExecuteHpaPath>()`中明确绑定：

```cpp
std::bind(&HpaExecutorNode::handleCancel, this, ...)
```

源码2：`execute()`及相关执行流程中的：

```cpp
goal_handle->is_canceling()
goal_handle->canceled(result)
```

学习者最终能够区分：

```text
is_canceling()
→ 检查当前HPA Goal是否已经处于取消请求状态

canceled(result)
→ Action Server把当前Goal收尾成CANCELED终态，并以result完成Goal
```

同时确认：仅看到`canceled(result)`而没有看到后续`return/break`时，不能证明当前执行路径已经停止。

### 第2批：replan上限与动态timeout

源码1：`replanAndExecute()`中的：

```cpp
if (replan_attempt >= max_replan_attempts_) {
  ...
}
```

学习者能够判断`replan_attempt == max_replan_attempts_`时条件成立，但仅凭`if`本身不能证明函数已经退出，还需要继续检查控制流出口。

源码2：anchor导航动态超时逻辑：

```text
timeout =
anchor_nav_timeout_base_
+ dist_to_target * anchor_nav_timeout_per_meter_
```

并在超时路径调用：

```cpp
nav_client_->async_cancel_goal(nav_goal_handle);
```

学习者能够确认这里取消的是下层Nav2 `NavigateToPose` Goal，不是直接把上层HPA `ExecuteHpaPath` Goal设为ABORTED。超时后可能进入后续replan等流程，要继续看调用方控制流。

### 第3批：普通重试、特殊失败与SWITCH_TRANSIENT

源码包含：

```cpp
bool anchor_reached = false;
bool plan_failure = false;
```

以及`Retrying anchor ...`、取消检查和失败后replan路径。

学习者确认：

- `navigateToAnchor()`返回`SUCCEEDED`时，`anchor_reached`由`false`变为`true`。
- 某些规划快速失败被单独分类，因为源码认为原条件下继续机械重试无意义，应进入replan。
- 当前保存的摘录缺少完整重试循环头，因此无法根据现有证据精确计算`max_nav_retries_`对应的总`navigateToAnchor()`调用次数；不能猜。

最终Goal的`SWITCH_TRANSIENT`路径中，同一可见代码块里会再次调用一次`navigateToGoal()`，新的返回值覆盖`goal_nav_result`；若这个新返回值为`SUCCEEDED`，则跳转到成功路径并跳过普通replan。

### 第4批：成功、失败和取消终态

阅读了最终replan结果分支：

```cpp
if (replanAndExecute(...)) {
  goal_handle->succeed(result);
} else {
  if (!goal_handle->is_canceling()) {
    goal_handle->abort(result);
  } else {
    goal_handle->canceled(result);
  }
}
```

学习者能够准确整理：

```text
replan成功
→ SUCCEEDED

replan失败 + 没有取消请求
→ ABORTED

replan失败 + 正在取消
→ CANCELED
```

并说明`replanAndExecute()==false`不能无条件`abort()`，因为`false`路径中仍可能是用户取消导致的结束。

## 关键纠正

### 1. callback身份必须由注册点证明

最初根据`rclcpp_action::CancelResponse HpaExecutorNode::handleCancel(...)`判断其为cancel callback不够严谨。

正确判断：必须找到`create_server<ExecuteHpaPath>()`中的`std::bind(&HpaExecutorNode::handleCancel, ...)`注册关系。

### 2. 终态操作不等于控制流已经停止

本日反复确认：

```text
看到 canceled/succeed/abort
≠ 已经证明函数结束
```

还必须追`return`、`break`、`goto`、函数返回或其他明确控制流出口。

### 3. 下层Nav2取消不等于上层HPA立即abort

```cpp
nav_client_->async_cancel_goal(nav_goal_handle);
```

操作的是`NavigateToPose` Action Client持有的下层Goal。上层HPA最终是replan、canceled还是abort，需要继续看上层控制流。

### 4. 源码证据不足时不能猜重试次数

如果看不到完整循环条件、初值和自增方式，即使看到`max_nav_retries_`变量，也不能直接断言总调用次数。

## 正式考试

### 第一轮原始评分：76/100

原评分中两处需要后续修正：

- 第2题把学习者回答中的题目复述“会立刻abort”误当成了学习者结论。
- 第4题要求学习者从被`...`隐藏的代码中读出“等待”动作，题目上下文不足；同时对其余控制流理解扣分过重。

### Agent误判修正

原判断：第2题12/20，第4题8/20，总分76/100。

错误原因：

- 第2题误读学习者答案语义。学习者实际结论是：`async_cancel_goal()`只能证明异步取消Nav2 Goal，HPA最终结果仍需看具体控制流。
- 第4题给出的考试片段用`...`隐藏了等待代码，因此不能要求学习者从该片段证明“等待”；但内层`SUCCEEDED`判断确实属于外层`SWITCH_TRANSIENT`分支内部，这一控制流关系仍需要纠正。

撤回内容：撤回第2题错误扣分；撤回第4题关于隐藏“等待”动作的扣分和过重扣分。

新判断：

- 第1题：16/20
- 第2题：20/20
- 第3题：20/20
- 第4题：16/20
- 第5题：20/20

新分数：**92/100**。

是否影响通过状态：是。修正后达到通过线。

### 定向确认

最后针对第4题确认：

```cpp
goal_nav_result = navigateToGoal(...);

if (goal_nav_result == NavResult::SUCCEEDED) {
  goto final_goal_succeeded;
}
```

学习者确认：如果前一句是当前可见代码块中重新执行的`navigateToGoal()`，那么内层`if`检查的就是这次新返回值。

同时指出“第二次”只能描述为“当前可见代码中的再次调用”，不能武断断言它是整个任务生命周期中的绝对第二次调用。该判断正确。

## 已掌握

- 状态、事件、条件、动作、下一状态/终态的基本拆解方法。
- retry次数必须由实际控制流证明。
- cooldown与timeout的职责区别。
- 子任务timeout与上层Action终态的层级区别。
- `is_canceling()`是状态检查，`canceled(result)`是Action取消终态收尾。
- 下层Nav2 Goal取消与上层HPA Goal终态不是同一层。
- `SUCCEEDED/ABORTED/CANCELED`三种Action终态的选择逻辑。
- 判断终态闭合必须继续追控制流出口。
- 源码摘录不完整时明确说“无法判断”，不补全不存在的证据。

## 仍需注意

- 不要把`canceled(result)`仅描述成“向Client发送一个canceled字段”；它首先是在Server侧完成Goal的CANCELED终态。
- 不要把同一可见代码块中的“再次调用”未经证据扩大成整个任务生命周期中的绝对第2次、第3次调用。
- Day24会继续深入Day22已经见过的lambda线程入口，重点转到捕获方式以及`shared_ptr/weak_ptr`对异步生命周期的影响。

## 验证与实操证据

- 本日未修改`Cyber_dog_mini`业务源码。
- 未实际制造timeout/cancel场景。
- 未执行编译、启动或现场运行验证。
- 以上不属于Day23默认通过硬性条件。

## 最终通过状态

- 正式考试最终成绩：**92/100**
- 关键概念错误：已纠正，无未解决关键错误
- Day23状态：✅ 完成

## 下一窗口交接

- 本日状态：✅ 完成
- 正式考试成绩：92/100
- 作废题目：无整题作废；第4题中被`...`隐藏的“等待”要求撤回，不计学习者错误
- 定向补测：完成`SWITCH_TRANSIENT`内部重新调用后返回值检查关系确认
- 已掌握：状态转移、retry/replan区别、动态timeout、上下层Action取消关系、`is_canceling/canceled`、`succeed/abort/canceled`终态选择、终态闭合证据判断
- 仍然薄弱：术语上需继续避免把`canceled(result)`简化成单纯“发result”；描述调用次数时要限定证据范围
- 已完成源码批次：取消请求与执行侧响应；replan上限与动态timeout；普通重试与特殊失败；三种Action终态
- 下一批应阅读的两处源码：Day24真实异步lambda注册点；与该lambda相关的`shared_ptr/weak_ptr`持有或对象生命周期声明
- 尚未确认：当前保存摘录缺失完整`max_nav_retries_`循环头，无法精确证明其总调用次数
- 仓库与分支：`ShowyHe/cpp-ros2-code-reading-30days` / `main`
- 提交SHA：写入后由GitHub提交生成
- 下一窗口必须从这里开始：Day24 Lambda捕获和`shared_ptr`生命周期
- 下一日禁止提前展开：不要求实际重构，不展开复杂模板和函数式编程；Day25大函数拆解不提前学习
