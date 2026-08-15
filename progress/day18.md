# Day 18：TF、Client、Future 和异步回调

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- README开始状态：Day 17 ✅，Day 18 🟨
- 真实源码基底：资料库 `multi_map_switcher(20260805-105428).zip`
- 今日主题：TF Buffer/Listener、PoseStamped、lookupTransform、doTransform、Client、Future、异步结果回调
- 今日边界：不深入TF数学推导、DDS内部机制、Executor线程内部、mutex并发和Action

## 今日目标

- 区分`PoseStamped`与TF/`TransformStamped`。
- 理解`tf_buffer_`、`tf_listener_`的对象关系以及`*tf_buffer_`解引用。
- 看懂`lookupTransform(target, source, time)`和`doTransform()`。
- 区分Client、Request、Future、Response。
- 区分Server callback与Client result callback。
- 理解`async_send_request()`后的真实异步时间关系。
- 比较`future.wait_for()`等待式处理和异步callback处理。

## TF基础

核心关系：

```text
TransformListener
→ 持续接收TF
→ 更新Buffer

业务代码
→ 查询Buffer
→ lookupTransform(target, source, time)
→ 得到source→target的TF
```

`tf_buffer_`与`tf_listener_`：

```cpp
tf_buffer_ =
  std::make_shared<tf2_ros::Buffer>(
    get_clock());

tf_listener_ =
  std::make_shared<tf2_ros::TransformListener>(
    *tf_buffer_);
```

已确认：

- `tf_buffer_`类型是`std::shared_ptr<tf2_ros::Buffer>`，指向真正的Buffer对象。
- `*tf_buffer_`是对shared_ptr解引用，得到Buffer对象本身，不是地址。
- `tf_listener_`类型是`std::shared_ptr<tf2_ros::TransformListener>`。
- 创建`TransformListener`时传`*tf_buffer_`，因为Listener构造需要Buffer对象的引用，并在后续监听TF时更新同一个Buffer。
- `tf_buffer_.get()`才返回`tf2_ros::Buffer *`原始指针。

主要纠正：

- 一度把`*tf_buffer_`理解成取得地址；已纠正为解引用取得Buffer对象。
- 一度把`tf_buffer_`理解成“当前TF”；已纠正为Buffer对象可缓存多个frame和多个时间点的TF关系。

## PoseStamped与TF

`geometry_msgs::msg::PoseStamped`表示带时间戳和坐标系信息的位姿：

```text
PoseStamped
├── header
│   ├── stamp
│   └── frame_id
└── pose
    ├── position
    └── orientation
```

已确认：

- `PoseStamped`：某个对象在某个坐标系下的位置和姿态。
- `header.frame_id`：该Pose当前由哪个坐标系表达。
- TF/`TransformStamped`：两个坐标系之间的变换关系。
- `PoseStamped`不是TF，TF也不是Pose。

## TF真实源码

### 源码1：Buffer / Listener创建

```cpp
tf_buffer_ =
  std::make_shared<tf2_ros::Buffer>(
    get_clock());

tf_listener_ =
  std::make_shared<tf2_ros::TransformListener>(
    *tf_buffer_);
```

重新考核后通过：

- 能准确写出两个shared_ptr完整类型。
- 能说明`make_shared<Buffer>()`创建Buffer对象并返回shared_ptr。
- 能说明`*tf_buffer_`取得Buffer对象本身。
- 能画出Listener更新Buffer、业务代码查询Buffer的关系。

### 源码2：Pose坐标转换

```cpp
bool HpaExecutorNode::transformPoseToFrame(
  const geometry_msgs::msg::PoseStamped & input,
  const std::string & target_frame,
  geometry_msgs::msg::PoseStamped & output) const
{
  geometry_msgs::msg::PoseStamped normalized = input;

  if (normalized.header.frame_id.empty()) {
    normalized.header.frame_id = global_frame_;
  }

  if (normalized.header.frame_id == target_frame) {
    output = normalized;
    return true;
  }

  try {
    const auto transform =
      tf_buffer_->lookupTransform(
        target_frame,
        normalized.header.frame_id,
        tf2::TimePointZero);

    tf2::doTransform(
      normalized,
      output,
      transform);

    output.header.frame_id = target_frame;
    return true;
  } catch (const tf2::TransformException & ex) {
    return false;
  }
}
```

已确认：

- `input`：源坐标系下的输入PoseStamped，const引用。
- `normalized = input`：复制一份局部PoseStamped，便于补`frame_id`而不修改输入对象。
- `target_frame`：目标坐标系名字。
- `output`：普通引用输出参数，不是函数返回值；函数真正返回`bool`表示成功/失败。
- `lookupTransform(target_frame, source_frame, TimePointZero)`查询source→target的最新可用TF。
- `TimePointZero`不是“0秒TF”，而是查询当前Buffer可提供的最新可用变换。
- `doTransform(normalized, output, transform)`使用TF把源坐标系下的Pose换算成目标坐标系下的Pose。
- 如果源frame与目标frame相同，直接`output = normalized`并返回成功。
- TF查询抛异常后不会继续`doTransform()`，最终返回`false`。

主要纠正：

- 一度把`target_frame`固定理解成`base_link`；已纠正为目标frame由调用参数决定，`base_link`只是可能值之一。
- 一度把`PoseStamped`描述成TF；已纠正为“Pose是对象位姿，TF是坐标系之间的转换关系”。

最终TF模型：

```text
源坐标系下的PoseStamped
+
source→target的TF
↓
doTransform()
↓
目标坐标系下的PoseStamped
```

## Client / Future基础

核心异步模型：

```text
Client创建并填写Request
→ async_send_request()
→ Request通过ROS2发送到Server
→ Request到达后Server ready
→ Executor调度Server callback
→ Server callback读取Request、填写Response
→ Response返回Client
→ 对应Future ready
→ Executor调度Client result callback
→ future.get()
→ 得到Response
```

已确认：

- Server callback负责处理Request并填写Response。
- Future属于Client侧，表示一次异步请求未来的结果，不是Response本身。
- Client result callback接收`SharedFuture`，通过`future.get()`取得真正Response。
- callback解决的是“代码什么时候执行”，而不是“代码写在哪个对象里”。
- Client也是Executor等待的ROS2事件源之一；不需要业务代码循环检查Response是否到达。

## 异步Client真实源码

### 源码1：callback式异步调用

```cpp
void MultiMapManager::loadMapAsync(
  const std::string & map_file)
{
  if (!load_map_client_->service_is_ready()) {
    is_switching_ = false;
    return;
  }

  auto request =
    std::make_shared<nav2_msgs::srv::LoadMap::Request>();

  request->map_url = map_file;

  load_map_client_->async_send_request(
    request,
    std::bind(
      &MultiMapManager::onLoadMapResponse,
      this,
      std::placeholders::_1));
}
```

结果callback：

```cpp
void MultiMapManager::onLoadMapResponse(
  rclcpp::Client<nav2_msgs::srv::LoadMap>::SharedFuture future)
{
  is_switching_ = false;

  auto result = future.get();

  if (
    result->result ==
    nav2_msgs::srv::LoadMap::Response::RESULT_SUCCESS)
  {
    logInfo("Async map load completed successfully");
  } else {
    logError(
      "Async map load failed: " +
      std::to_string(result->result));
  }
}
```

已确认：

- `request`是指向`LoadMap::Request`对象的shared_ptr。
- `request->map_url = map_file`填写Request字段。
- `async_send_request(request, callback)`主要完成发送Request和注册结果callback。
- `loadMapAsync()`结束时不能保证Response已经回来。
- Response回来后Future先ready，Executor再调度`onLoadMapResponse(future)`。
- 进入结果callback时Future已经ready，因此可直接`future.get()`，无需再`wait_for()`。

### 源码2：wait_for等待式处理

```cpp
auto future =
  load_map_client_->async_send_request(request);

auto status =
  future.wait_for(std::chrono::seconds(10));

if (status != std::future_status::ready) {
  return false;
}

auto result = future.get();
```

已确认：

- `async_send_request()`仍然是异步发送API。
- 这版之所以当前函数会等待，是因为后续主动调用`future.wait_for()`。
- `wait_for_service()`等待Service Server可用；`future.wait_for()`等待本次Request对应的结果ready。
- 超时后提前`return false`，不会执行后续`future.get()`。
- callback版不阻塞当前函数；Response未来回来后再由Executor调结果callback。

## 独立总结

学习者独立总结通过：

```text
PoseStamped(source)
→ normalized复制
→ lookupTransform(target, source, latest)
→ 得到source→target TF
→ doTransform()
→ PoseStamped(target)
```

以及：

```text
Client创建Request
→ 填写Request
→ async_send_request
→ ROS2发送到Server
→ Executor调Server callback
→ callback读取Request并填写Response
→ Response返回Client
→ Future ready
→ Executor调Client result callback
→ future.get()
→ Response
```

并能正确区分：

```text
wait_for版本
→ 异步发送后当前执行流主动等待

callback版本
→ 异步发送后当前函数不等待
→ Response回来以后再处理
```

## 正式考试

### 第一轮

前4题有效部分：74/80，折算92.5%，均达到核心要求。

有效题覆盖：

1. `PoseStamped`、`frame_id`、TF、Buffer对象区分。
2. `lookupTransform("odom", "map", TimePointZero)`和`doTransform()`结果判断。
3. Future与Response的区别、Future ready和`future.get()`。
4. Server callback与Client result callback的触发和职责。

### 第5题：作废

原题把Client结果回调命名为`responseCallback`，学习者按前面语境将其理解成Server侧Response处理回调；随后题目又要求把Client继续执行与Server异步处理强行排成唯一线性顺序。

该题存在两处教学问题：

1. callback命名产生歧义；
2. Client的后续普通代码与Server侧处理在异步系统中不一定具有源码可证明的固定先后顺序。

因此该题作废，撤回相关扣分，不记录为学习者错误。

后续定向确认中，学习者能够正确指出：

- `create_service(..., serverCallback)`注册Server callback。
- `async_send_request(request, clientResultCallback)`注册Client result callback。
- Request到达Server后才能由Executor调Server callback。
- Server callback填写Response后，Response返回Client使Future ready。
- Future ready后Executor才调Client result callback。
- `doSomethingElse()`与Server callback谁先执行，不能仅凭给定代码确定。

## 今日已掌握

- `PoseStamped`与TF/`TransformStamped`的本质区别。
- Buffer / TransformListener对象关系和shared_ptr解引用。
- source frame、target frame、`TimePointZero`。
- `lookupTransform()`与`doTransform()`职责。
- Client、Request、Future、Response四类对象。
- Server callback与Client result callback的职责边界。
- `async_send_request()`、`future.wait_for()`、`future.get()`。
- callback式异步处理和主动等待式处理的区别。
- ROS2异步代码的书写顺序不等于跨节点事件的唯一执行顺序。

## 仍需注意

- 不要把PoseStamped称为TF。
- 不要把`tf_buffer_`称为“当前TF”；它是指向Buffer对象的shared_ptr。
- 不要把`*tf_buffer_`说成取地址。
- 不要固定假设`target_frame`一定是`base_link`。
- 不要把Future当成Response；`future.get()`才取得结果。
- 不要因为函数名包含`response`就猜它是Server callback；应看注册点和函数参数。
- 不要给异步两端不存在证据的事件强行排唯一顺序。

## 当前状态

- 基础讲义：完成。
- TF真实源码：完成。
- 异步Client真实源码：完成。
- 独立总结：通过。
- 正式考试：有效部分通过；存在歧义的第5题已作废并完成定向确认。
- 关键概念错误：无未纠正项。
- 是否正式通过：是。
- Day 18状态：✅ 完成。

## 下一窗口交接

- 下一日：Day 19 Action Server基本结构。
- 重点区分Action类型、Goal消息、Goal Handle、Result，以及goal/cancel/accepted三个核心回调。
- 源码从Action Server创建点开始，依次定位三个入口；先理解目标被接受前后的对象关系，不提前展开完整执行循环、Feedback频率和Result完成逻辑。
- Day 20再继续Action执行、Feedback、Result和取消的完整生命周期。
