# Day 17：Publisher、Subscriber、Timer 和 QoS 基础

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- README开始状态：Day 16 ✅，Day 17 🟨
- 真实源码基底：资料库 `multi_map_switcher(20260805-105428).zip`
- 今日主题：Publisher、Subscriber、消息对象、Topic、Timer、回调、Executor、QoS基础
- 今日边界：QoS只学习历史深度和可靠性；不展开DDS内部机制；不进入TF、Future、Action和线程调度细节

## 今日目标

- 区分Topic名字、消息类型、Publisher对象、Subscription对象和消息对象。
- 理解Publisher主动`publish(msg)`，Subscription在消息到达后由Executor调度回调。
- 理解Timer创建时注册回调，Timer到期后由Executor调度Timer callback。
- 理解`spin(node)`处理整个Node的ROS2回调事件，不会反复执行构造函数。
- 能从真实源码追踪“Timer → callback → 创建消息 → 填字段 → publish()”完整输出链路。
- 能从真实源码追踪“Topic消息 → Subscription ready → Executor → subscription callback”输入链路。
- 掌握QoS的KeepLast历史深度与Reliable可靠性基础含义。

## 基础讲义

已完成并纠正以下运行模型：

```text
Node初始化一次
→ 创建Publisher / Subscription / Timer等ROS2对象
→ main中spin(node)
→ Executor持续等待和处理Node事件
```

Publisher输出链：

```text
Timer到期
→ Timer ready
→ Executor调度Timer callback
→ callback创建并填写msg
→ Publisher publish(msg)
→ ROS2通信
→ Topic数据送往匹配的Subscription
```

Subscriber输入链：

```text
Topic消息到达
→ Subscription ready
→ Executor调度Subscription callback(msg)
→ callback处理消息
```

关键澄清：

- Publisher里面没有“Publisher回调”；真实源码中`publishVisualization()`是Node成员函数，被注册为Timer callback，函数内部主动调用Publisher的`publish()`。
- Publisher和Timer是Node拥有的两个独立ROS2对象。
- `spin(node)`不会持续调用类构造函数；构造函数只在Node创建时执行一次。
- Subscriber不需要Timer去主动轮询Topic；消息到达后由Executor调度订阅回调。
- ROS2通信层负责传输消息数据，Executor负责回调调度，两者职责不同。
- Publisher侧局部`msg`与Subscriber回调参数`msg`不能仅因变量名相同就视为同一个C++变量。

## 源码阅读

### 第1批：Topic输出 + Timer

源码1：`src/hpa/hpa_planner_node.cpp`，`HpaPlannerNode`构造函数中的Publisher/Timer创建。

```cpp
viz_pub_ =
  create_publisher<visualization_msgs::msg::MarkerArray>(
    "/map_hpa", 10);

if (viz_publish_rate_ > 0.0) {
  viz_timer_ = create_wall_timer(
    std::chrono::duration<double>(1.0 / viz_publish_rate_),
    std::bind(&HpaPlannerNode::publishVisualization, this));
}
```

源码2：`HpaPlannerNode::publishVisualization()`中的消息创建与发布。

```cpp
void HpaPlannerNode::publishVisualization()
{
  if (!is_ready_) {
    return;
  }

  visualization_msgs::msg::MarkerArray markers;
  // 填写markers
  ...
  viz_pub_->publish(markers);
}
```

独立回答结果：通过。

已确认：

- `visualization_msgs::msg::MarkerArray`是消息类型。
- `"/map_hpa"`是Topic名字。
- `viz_pub_`是保存Publisher智能指针的Node成员变量。
- `viz_timer_`是保存Timer智能指针的Node成员变量。
- `markers`是本次回调中的局部消息对象。
- 真正发布调用点是`viz_pub_->publish(markers)`。
- `is_ready_ == false`时提前`return`，本次回调不会执行到发布点。

主要纠正：

- 最初将Timer callback称为“Publisher回调”；纠正为：`publishVisualization()`是Node成员函数，被注册为Timer callback，并在函数内部调用Publisher。
- 最初将`spin(node)`理解成Executor反复调用类初始化；纠正为构造函数只执行一次，Executor之后持续处理已注册的事件。

### 第2批：Topic输入 + Subscription callback

源码1：`src/hpa/hpa_executor_node.cpp`中的Costmap Subscription创建。

```cpp
rclcpp::QoS costmap_qos(10);
costmap_qos.reliable();

costmap_sub_ =
  create_subscription<nav2_msgs::msg::Costmap>(
    costmap_topic_,
    costmap_qos,
    std::bind(
      &HpaExecutorNode::costmapCallback,
      this,
      std::placeholders::_1));
```

源码2：`HpaExecutorNode::costmapCallback()`。

```cpp
void HpaExecutorNode::costmapCallback(
  const nav2_msgs::msg::Costmap::SharedPtr msg)
{
  ...
  latest_costmap_ = msg;
  ...
}
```

已确认：

- `nav2_msgs::msg::Costmap`是消息类型。
- `costmap_topic_`保存实际Topic名字。
- `costmap_sub_`保存Subscription智能指针。
- Topic消息到达后，Subscription变为ready，由Executor调度`costmapCallback(msg)`。
- `msg`是指向`Costmap`对象的shared_ptr，所以用`msg->...`访问成员。
- `latest_costmap_ = msg`复制的是shared_ptr，不是复制整份Costmap数据；两个shared_ptr随后共同指向同一对象。

参数来源补充：

```text
"costmap_topic"
→ ROS2参数名

"/local_costmap/costmap_raw"
→ 默认参数值

costmap_topic_
→ std::string成员变量，保存当前实际参数值
```

并明确：`get_parameter("costmap_topic").as_string()`读取的是参数当前实际值，不一定是默认值。

## QoS基础

真实源码出现：

```cpp
rclcpp::QoS costmap_qos(10);
costmap_qos.reliable();
```

已掌握：

- `10`表示KeepLast历史深度，不是10Hz。
- `reliable()`表示可靠性策略，强调可靠送达并可重传，但不等于任何情况下Subscriber回调都必定成功执行。
- QoS不等于消息类型、Topic名字或Timer频率。
- `transient_local()`只识别为QoS策略，本日不展开DDS内部语义。

## 独立总结

学习者独立给出：

```text
Node里面有Publisher、Timer和Timer callback
→ spin(node)
→ Timer到期
→ Executor调度Timer callback
→ callback填充msg
→ publish到Topic
→ Subscription收到消息
→ Executor调度Subscription callback
→ callback执行
```

纠正后最终版本：

```text
Node创建Publisher + Timer
→ Timer注册Timer callback
→ spin(node)
→ Timer到期并ready
→ Executor调度Timer callback
→ callback创建并填写msg
→ Publisher publish(msg)
→ ROS2 Topic通信
→ Subscription收到可处理消息并ready
→ Executor调度Subscription callback(msg)
→ callback处理消息
```

结果：通过。

对象生命周期也已确认：

- Node、Publisher、Timer、Subscription通常由Node成员长期持有。
- `markers`等消息对象是一次回调中的局部对象，函数结束后销毁。
- `publish(msg)`完成后局部消息对象销毁，不等于把已经交给ROS2发布系统的数据撤回。

## 正式考试

### 第一轮：100/100，通过

1. 正确区分`MarkerArray`消息类型、`/map_hpa` Topic名、`viz_pub_` Publisher成员和`markers`消息对象。
2. 正确认识`create_wall_timer()`创建Timer并注册callback；Timer到期后由Executor调度callback。
3. 正确说明Subscriber不需要主动循环读取Topic；消息到达→Subscription ready→Executor→callback。
4. 正确区分QoS历史深度和Reliable可靠性，并确认两者都不是发布频率。
5. 正确说明局部消息对象与Publisher成员变量的生命周期区别，并定位`viz_pub_->publish(markers)`为实际发布点。

无关键概念错误。

### Timer周期定向补测：通过

源码：

```cpp
std::chrono::duration<double>(1.0 / viz_publish_rate_)
```

回答：

- Timer周期由`viz_publish_rate_`决定。
- 当`viz_publish_rate_ = 2.0`时，周期为`0.5s`。

补测通过。

## 今日已掌握

- Topic名字、消息类型、Publisher、Subscription和消息对象的区别。
- Publisher主动`publish()`与Subscription事件回调的不同工作方式。
- Timer callback的注册与Executor调度关系。
- `spin(node)`、Executor、ROS2通信层的职责边界。
- Timer周期来源与频率/周期换算。
- `shared_ptr`消息参数与成员保存之间的指针复制和共享所有权。
- QoS KeepLast历史深度与Reliable可靠性基础。
- 从成员数据生成消息并发布、以及Topic输入进入订阅回调的完整事件流。

## 仍需注意

- 不要称`publishVisualization()`为Publisher callback；它是Timer callback，内部调用Publisher。
- 不要认为`spin(node)`会反复执行Node构造函数。
- 不要认为Subscriber需要Timer才能从Topic取消息。
- 不要把Executor描述为消息传输层；Executor负责调度回调。
- 不要把参数名`"costmap_topic"`与参数值、成员变量`costmap_topic_`混为一谈。
- 不要把`reliable()`理解成任何情况下都绝对必达或回调必定成功执行。

## 当前状态

- 基础讲义：完成。
- 真实源码阅读：Topic输出、Topic输入和Timer全部完成。
- 独立总结：通过。
- 正式考试：100/100通过。
- Timer周期定向补测：通过。
- 关键概念错误：无。
- 是否正式通过：是。
- Day 17状态：✅ 完成。

## 下一窗口交接

- 下一日：Day 18 TF、Client、Future和异步回调。
- 延续Day16已经见过的`async_send_request()`/Future基础，但Day18要正式区分“立即返回的Future”和“稍后才到达的最终结果”。
- 真实源码按计划选择一个TF查询和一个异步Client调用；追踪请求发出、Future保存、结果回调和错误/超时分支。
- 重点区分：TF Buffer/Listener、源坐标系/目标坐标系、Client对象、Request对象、Future对象、Response对象和异步结果callback。
- 不深入线程调度器内部实现；并发与mutex统一留到Week 4。
