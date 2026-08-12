# Day 16：Service、Request 和 Response

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- 真实源码基底：`multi_map_switcher / HPA`
- 今日主题：Service类型、Service Server/Client、Request、Response、回调、请求发送与响应取得
- 今日边界：不系统展开Future等待策略、TF和Action；Future只讲到能闭合Service链路

## 今日目标

- 区分Service类型、Service名字、Service Server对象和回调函数。
- 看懂`.srv`中Request和Response的字段边界。
- 区分Client侧Request变量与Server侧回调Request参数。
- 找到真正发送Request的`async_send_request()`。
- 理解Server回调读取Request、执行业务、填写Response的职责。
- 理解`std::bind()`、`spin(node)`与Executor在Service事件中的不同角色。
- 能画出Client→Request→ROS2→Server回调→Response→Future→Client完整链路。

## 基础讲义

已完成：

- `.srv`文件中`---`上方是Request，下方是Response。
- `srv::SplitMap`、`srv::SwitchMap`是Service类型，不是Service名字。
- `create_service<ServiceT>(name, callback)`在Node内部创建Service Server并注册回调。
- `"~/split_map"`、`"~/reload_map"`是Service名字；`~`使用Node私有命名空间。
- `std::bind(&Class::callback, this, _1, _2)`提前建立Service事件与成员回调函数的绑定关系。
- 注册回调不等于立即执行回调；Request到达后由Executor调度回调。
- Client创建并填写Request，再通过`async_send_request(request)`把请求交给ROS2发送。
- Server回调参数`request`不是重新创建Client的局部变量，而是Server侧用于读取本次请求数据的参数。
- Client侧`req`与Server侧`request`通常不是同一个C++变量或内存对象，但携带同一次请求的字段数据。
- `req->map_name`与`request->map_name`之所以能对应，是因为Client和Server使用同一个`SwitchMap::Request`消息类型；ROS2传输的是字段数据，不是把Client的`req`指针直接跨进程交给Server。
- 带函数体的`void handleReloadRequest(...){...}`是回调函数定义；其中`request`和`response`是形参变量，看到这两个参数不代表在函数签名处重新创建Request/Response业务数据。
- 对一次Service调用，Server回调中的`response`由rclcpp/ROS2为本次调用准备并传入，回调负责填写字段；通常不需要Server自己`make_shared<Response>()`。
- Server回调读取`request`、执行业务、填写`response`；`return;`只结束`void`回调，不是`return response`。
- Response返回Client后，对应Future变为ready；`future.get()`取得Response。
- `rclcpp::spin(node)`不是“spin某个Service”，而是让Executor持续处理这个Node拥有的ROS2回调事件。
- `spin()`不是Service Server专属机制；Subscription、Timer、Client异步结果回调等Node事件同样需要Executor调度。
- Node本身不等于Service Server；Node通过`create_service()`拥有一个或多个Service Server对象。
- 每次Service调用都有本次调用对应的Request/Response数据对象；不能把回调形参理解成一个永久存在、被所有请求反复覆盖的全局Request/Response。

## 源码阅读批次

### 第1批：SplitMap接口 + Service注册

源码：

1. `srv/SplitMap.srv`
2. `src/map_splitter_node.cpp`中的`create_service<srv::SplitMap>()`

接口字段：

```srv
string source_map_yaml
float64 tile_size
float64 overlap
string output_dir
---
bool success
string message
string[] map_names
string config_file
```

注册点：

```cpp
split_service_ = create_service<srv::SplitMap>(
  "~/split_map",
  std::bind(
    &MapSplitterNode::splitMapCallback,
    this,
    std::placeholders::_1,
    std::placeholders::_2));
```

已确认：

- `srv::SplitMap`：Service类型。
- `"~/split_map"`：Service名字。
- `split_service_`：保存Service Server对象的成员变量。
- `splitMapCallback`：Server回调函数。

主要纠正：

- 最初将`srv::SplitMap`误认为Service名字；纠正后已区分类型与名字。
- `split_service_`不是Client调用变量，而是Node中保存Service Server对象的成员。

### 第2批：SplitMap回调参数 + 错误返回路径

源码：

1. `MapSplitterNode::splitMapCallback()`函数签名
2. `src/map_splitter_node.cpp`中的回调实现

关键源码：

```cpp
void MapSplitterNode::splitMapCallback(
  const std::shared_ptr<srv::SplitMap::Request> request,
  std::shared_ptr<srv::SplitMap::Response> response)
{
  SplitterConfig config;
  config.tile_size_meters = request->tile_size;
  config.overlap_meters = request->overlap;
  config.output_dir = request->output_dir;
  splitter_->setConfig(config);

  if (!splitter_->loadSourceMap(request->source_map_yaml)) {
    response->success = false;
    response->message = "Failed to load source map";
    return;
  }

  if (!splitter_->split()) {
    response->success = false;
    response->message = "Failed to split map";
    return;
  }

  response->success = true;
  response->message = "Map split successfully";
}
```

已确认：

- `request`类型：`std::shared_ptr<srv::SplitMap::Request>`。
- `response`类型：`std::shared_ptr<srv::SplitMap::Response>`。
- `request->...`是Client→Server输入数据。
- `response->... = ...`是Server写返回结果。
- 失败分支中的`return;`会阻止后续业务和成功响应继续执行。

### 第3批：真实SwitchMap Client发送点 + Server注册点

源码：

1. `src/multi_map_manager.cpp`中的`hpa_reload_client_->async_send_request()`
2. `src/hpa/hpa_planner_node.cpp`中的`reload_service_ = create_service<srv::SwitchMap>()`

Client关键源码：

```cpp
if (hpa_reload_client_->service_is_ready()) {
  auto req = std::make_shared<srv::SwitchMap::Request>();
  req->map_name = source_map;
  hpa_reload_client_->async_send_request(req,
    [this](rclcpp::Client<srv::SwitchMap>::SharedFuture future) {
      auto res = future.get();
      if (res->success) {
        logInfo("HPA planner reloaded successfully");
      } else {
        logWarn("HPA planner reload failed: " + res->message);
      }
    });
}
```

Server关键源码：

```cpp
reload_service_ = create_service<srv::SwitchMap>(
  "~/reload_map",
  std::bind(&HpaPlannerNode::handleReloadRequest, this,
            std::placeholders::_1, std::placeholders::_2));
```

已确认：

- `req`是Client侧局部`shared_ptr`变量。
- `req->map_name = source_map`填写Request字段。
- `async_send_request(req, ...)`是真正发送Request的调用点。
- `future`是未来取得Response的结果对象，不是Response本身。
- `auto res = future.get()`真正取得Response。
- Client和Server必须使用相同Service类型`srv::SwitchMap`，并通过匹配的Service名字通信。

## 独立总结

完成Client→Server→Client闭环整理。

主要纠正：

- 独立总结中一度把`future.get()`误写成发送Request的位置；纠正为：
  - `async_send_request(req)`：发送Request。
  - `future.get()`：取得Response。

结果：通过。

## 正式考试

### 第一轮：86/100，因关键概念错误未通过

主要问题：

- 将Server回调误认为会创建Request。
- 将`std::bind()`与Request到达后真正的回调触发过程混在一起。

进一步补通：

```text
Client req
→ async_send_request(req)
→ ROS2通信
→ Server侧Request数据
→ Executor发现Service事件
→ 调用之前注册好的callback
→ callback读取request
→ callback填写response
→ ROS2返回Response
→ future ready
→ future.get()
```

并进一步明确：

```text
Client侧 req
→ 指向Client创建的Request对象
→ 填写 req->map_name

ROS2通信
→ 传输Request字段数据

Server回调 request
→ 是Server回调形参shared_ptr
→ 读取 request->map_name

req 与 request
→ 不是同一个C++变量
→ 通常也不是同一个跨进程内存对象
→ 但承载同一次请求对应的字段数据
```

### 第二轮：74/100，未通过

已掌握Request/Response和Node/Service Server关系，但仍混淆：

- `std::bind()`：提前注册/绑定回调关系。
- Executor：事件到达后真正调度callback。
- ROS2通信层：负责Request/Response数据传输。
- Server回调：读取Request、填写Response，不是填写Request。

同时完成关键对象关系澄清：

```text
HpaPlannerNode
→ 是Node对象，不等于Service Server

HpaPlannerNode构造函数中的create_service()
→ 创建Service Server

reload_service_
→ Node成员变量，保存SwitchMap Service Server对象

rclcpp::spin(node)
→ 让Executor持续处理HpaPlannerNode拥有的ROS2事件
```

补充澄清：

- `std::make_shared<HpaPlannerNode>()`只能证明创建了`HpaPlannerNode`对象并由shared_ptr管理，不能单独证明它“就是Server”。真正的Server证据是该Node构造函数中的`create_service()`。
- `spin(node)`中的`node`指向`HpaPlannerNode`对象；spin的是整个Node，不是单独的`reload_service_`。
- `spin()`本身不是Server专属；只要Node有需要Executor处理的回调事件，例如Subscription、Timer、异步Client结果回调，都属于同一执行模型。
- Server回调签名中的`request`/`response`是函数形参；Request来自本次收到的请求数据，Response由rclcpp/ROS2为本次调用提供给回调填写。
- 每次新的Service请求都会形成该次调用对应的Request/Response处理上下文，不应理解为所有请求共享同一个永久Request对象。

### 第三轮：100/100，通过

五项全部正确：

1. `std::bind()`是提前建立Service与回调函数的绑定关系。
2. Request到达后由Executor发现并调度Service回调。
3. 回调中`request`主要读取，`response`主要填写。
4. ROS2通信层负责Request/Response传输，Executor负责回调调度。
5. 能正确补全`async_send_request → ROS2通信 → Executor → callback → request → response`链路。

## 今日已掌握

- Service类型、Service名字、Service Server对象、Client对象、回调函数的区别。
- Request与Response的数据方向。
- Client侧`req`与Server侧`request`的区别。
- `req->map_name`和`request->map_name`使用同一消息字段定义，但属于发送端和接收端不同变量/对象上下文。
- 回调函数定义中的`request`/`response`是形参，不等于在函数签名处重新创建Client的Request。
- Server侧Response由本次Service调用上下文提供给回调填写，而不是靠回调返回C++对象。
- `async_send_request()`与`future.get()`的区别。
- `std::bind()`、`spin(node)`、Executor、ROS2通信层的职责边界。
- 一个Node可以通过多个`create_service()`拥有多个Service Server。
- `spin(node)`处理整个Node的ROS2回调事件，而不是只spin某一个Service，也不是Server专属机制。
- 每次Service调用都有独立的Request/Response处理数据。

## 仍需注意

- 不要把`srv::Type`写成Service名字。
- 不要把Client侧Request变量与Server侧回调Request参数当成同一个C++变量。
- 不要认为`req`指针本身直接跨进程变成Server侧`request`；真正跨通信边界的是Request字段数据。
- 不要看到回调签名里的`request`/`response`就认为这里重新创建了Request/Response。
- 不要默认Server回调需要自己`make_shared<Response>()`；当前Service回调模型中Response对象由调用上下文提供，回调负责填写。
- 不要把`std::bind()`说成Request到达后“执行回调的人”。
- 不要把Executor说成负责ROS2数据传输；Executor负责回调调度。
- 不要把Node对象本身直接等同于Service Server对象。
- 不要把`spin(node)`理解成只让Service Server工作；它让Executor持续处理这个Node的各种回调事件。
- 不要把回调参数理解成所有Service调用共用的永久Request/Response对象。

## 当前状态

- 基础讲义：完成。
- 真实源码阅读：3批，共6处源码，完成。
- 独立总结：通过。
- 正式考试：第一轮86/100未通过；第二轮74/100未通过；第三轮100/100通过。
- 关键概念错误：已纠正。
- 是否通过：是。
- Day 16状态：✅ 完成。

## 下一窗口交接

- 下一日：Day 17 Publisher、Subscriber、Timer和QoS基础。
- Day 17必须先从“Node内部有哪些Publisher/Subscriber/Timer对象、谁创建、谁拥有、谁触发回调”讲完整运行模型，避免只讲局部API。
- 源码按计划选择一个Topic输入、一个Topic输出和一个Timer，追踪消息创建、字段填写和`publish()`。
- 重点区分：Topic名字、消息类型、Publisher对象、Subscriber对象、消息对象、订阅回调、Timer回调。
- QoS只讲真实源码出现的可靠性和队列深度，不展开DDS全部策略。
