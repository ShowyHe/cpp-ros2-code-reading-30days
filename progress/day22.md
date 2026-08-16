# Day 22：thread、mutex 和 atomic

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- README开始状态：Day 21 ✅，Day 22 🟨
- 今日主题：`std::thread`、lambda线程入口、`joinable/join`、共享数据、`std::mutex`、`std::lock_guard`、锁内复制锁外使用、`std::atomic<bool>`、基本data race判断
- 今日边界：不深入内存序、lock-free算法、复杂死锁设计、`condition_variable`和完整ROS2 Executor并发模型；不要求真实复现竞态或提交业务源码修复

## 今日目标

- 看懂`std::thread(...)`如何创建新的执行流。
- 能解释lambda作为线程入口时`[]`、`()`、`{}`的基本作用。
- 区分`joinable()`与`join()`。
- 能从真实源码找出共享成员的写点、读点和同步方式。
- 理解mutex保护的是一段共享访问过程，而不是变量名本身。
- 理解`lock_guard`的作用域与自动解锁。
- 掌握“锁内复制、锁外使用”以缩短持锁时间。
- 知道简单单变量状态可用`atomic<bool>`进行线程安全读写，并能与mutex适用场景区分。

## 前置补充：lambda基础

真实线程源码直接出现：

```cpp
std::thread([this, goal_handle]() {
  execute(goal_handle);
});
```

学习过程中发现：如果不先补最基础lambda语法，会妨碍Day22真实源码阅读。因此补充了仅满足本日需要的内容：

```text
[capture](parameters) {
  body
}
```

学习者最终能够区分：

```text
[this, goal_handle]
→ 捕获外部作用域中的this和goal_handle

()
→ lambda自身的参数列表，此处为空

{ execute(goal_handle); }
→ lambda被调用时执行的函数体
```

Day24仍保留值捕获/引用捕获、`this`生命周期、`shared_ptr/weak_ptr`和异步生命周期等深入内容。

## 真实源码阅读

### 第1批：线程创建与收尾

阅读了`HpaExecutorNode::handleAccepted()`中的线程创建：

```cpp
execute_thread_ =
  std::thread([this, goal_handle]() {
    execute(goal_handle);
  });
```

以及析构函数中的：

```cpp
if (execute_thread_.joinable()) {
  execute_thread_.join();
}
```

学习者能够确认：

```text
handleAccepted()
→ 创建新线程
→ 新线程执行execute(goal_handle)
→ handleAccepted()自身不必等待execute()结束
```

纠正的关键概念：

- `joinable()`不是判断“当前是否双线程运行”，而是判断该`std::thread`对象当前是否关联着一条可以被join的线程。
- `join()`不是“线程融合”，而是当前线程停在这里，等待`execute_thread_`管理的线程结束，然后继续执行。

### 第2批：costmap共享数据与mutex

写侧`costmapCallback()`在同一把`costmap_mutex_`保护下更新：

```text
latest_costmap_
last_costmap_msg_steady_
has_costmap_
```

读侧在`costmap_mutex_`保护下把共享数据复制到局部变量，再释放锁并继续使用局部副本。

学习者能够解释：

```text
写线程拿costmap_mutex_
→ 更新共享成员
→ 释放锁

读侧拿同一把costmap_mutex_
→ 把共享Costmap复制到局部costmap
→ 释放锁
→ 后续使用局部costmap
```

纠正点：`first_costmap`是callback中的局部变量，不是成员变量。

### atomic

当前保存的HPA源码摘录中没有确认到`std::atomic`真实使用点，因此本日没有伪造HPA atomic源码；atomic使用简化代码完成基础理解和考试。

核心理解：

```cpp
std::atomic<bool> stop{false};
```

一个线程：

```cpp
stop.store(true);
```

另一个线程：

```cpp
while (!stop.load()) {
  doWork();
}
```

学习者能够说明：如果线程B已经进入一次`doWork()`，线程A把`stop`改成`true`不会强制中断当前`doWork()`；B执行完当前工作后再次回到while条件，下一次`load()`读到`true`才退出。

## 已掌握

- `std::thread`对象用于管理新创建的线程。
- lambda可以作为`std::thread`的执行入口。
- `[]`是捕获列表，`()`是lambda参数列表，`{}`是函数体。
- `joinable()`和`join()`职责不同。
- 共享数据的读写双方要使用同一同步机制才形成有效互斥。
- 同一把mutex同一时刻只能被一个线程持有；另一个线程在拿锁处等待，直到锁释放后才可能继续。
- `std::lock_guard<std::mutex>`在作用域内持锁，离开作用域自动释放。
- 锁内复制、锁外使用可以减少持锁时间，避免其他线程被不必要阻塞。
- `atomic<bool>`适合简单独立的单变量状态；多个相关成员需要整体一致时通常仍应使用mutex。

## 正式考试

### 第一轮：84/100

主要正确项：

- 能解释新线程执行`execute(goal_handle)`，`handleAccepted()`无需等待。
- 能解释同一把mutex下的等待关系。
- 能解释atomic停止标志的时序。
- 能区分多个相关成员使用mutex、单一停止开关使用atomic。

关键错误：

- 对`joinable()`仍理解成“是否需要等待其他线程”。

因此虽然分数超过80，当时仍不能通过。

### 定向补测

学习者最终准确回答：

```text
joinable()
→ 判断这个thread对象当前是否关联着一条可以被join的线程

join()
→ 当前线程停在这里，等待execute_thread_管理的那条线程结束
```

关键概念错误消除。

### 最终成绩：94/100

- 正式考试：达标
- 关键概念错误：已全部纠正
- 实际竞态复现/编译/运行：未执行，且不属于本日硬性通过条件
- Day22状态：✅ 完成

## 仍需注意

- 不要把线程称作进程。
- 不要把`join()`描述成“融合线程”。
- 判断共享数据安全性时要追所有可能并发的读写点，不能只看到某一个写点有锁就说“变量被保护”。
- lambda生命周期、值/引用捕获以及`shared_ptr/weak_ptr`延后到Day24。

## 下一窗口交接

- 本日状态：✅ 完成
- 最终成绩：94/100
- 关键补测：`joinable()` / `join()`已通过
- 已完成源码批次：线程创建与析构收尾；costmap写侧与读侧mutex保护
- 已掌握：thread、lambda基础、join/joinable、mutex、lock_guard、锁内复制锁外使用、atomic基本用途、data race基本判断
- 下一日：Day23 状态、重试、超时和取消
- Day23必须掌握：显式状态/状态转移、重试计数、冷却时间、超时检测、取消请求和终态闭合
- Day23禁止提前扩展：不要求实际制造超时/取消场景，不强行重构大型状态机；Day24生命周期问题仍不提前展开
- 仓库与分支：`ShowyHe/cpp-ros2-code-reading-30days` / `main`
