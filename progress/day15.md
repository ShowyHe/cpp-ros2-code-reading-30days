# Day 15：Node、参数和日志

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- 真实源码基底：`multi_map_switcher`
- 今日主题：`rclcpp::Node`继承、构造函数、参数声明/读取、日志、节点启动基本流程
- 今日边界：不系统展开Topic、Service、Action完整接口行为

## 今日目标

- 区分源码文件名、CMake可执行名、C++变量名和ROS2默认节点名。
- 看懂`class X : public rclcpp::Node`的基本继承关系。
- 看懂Node构造函数中的基类初始化。
- 区分`declare_parameter()`的参数声明/默认值与`get_parameter()`的当前值读取。
- 跟踪ROS2参数值如何保存到C++成员变量。
- 理解日志只能提供执行位置和当时输出值的证据，不能自动证明业务正确。
- 区分`MapSplitterNode`对象与管理它的`std::shared_ptr`变量。

## 基础讲义

已完成：

- `class MyNode : public rclcpp::Node`：派生Node类。
- `Node("name", options)`：构造基类并给出默认ROS2节点名。
- `declare_parameter("name", default)`：声明参数并提供默认值。
- `get_parameter("name").as_*()`：读取当前参数值并转换为相应类型。
- 参数名与保存参数的C++成员变量不是同一个概念。
- `RCLCPP_INFO/WARN/ERROR`用于日志输出；日志本身不会自动改变控制流。
- `std::make_shared<MapSplitterNode>()`创建`MapSplitterNode`对象并调用构造函数，返回`std::shared_ptr<MapSplitterNode>`。
- `rclcpp::spin(node)`中的`node`是C++共享指针变量，不是节点名。

## 源码阅读批次

### 第1批：Node类声明 + MapSplitterNode构造参数

源码：

1. `include/multi_map_switcher/map_splitter_node.hpp`
2. `src/map_splitter_node.cpp`中的`MapSplitterNode`构造函数参数段

已确认：

- `MapSplitterNode`公有继承`rclcpp::Node`。
- 默认ROS2节点名由`: Node("map_splitter_node", options)`确定。
- `global_frame`参数名、默认值`"map"`、成员`global_frame_`能够区分。
- `tile_size`参数名、默认值`50.0`、读取类型`double`、成员`tile_size_`能够区分。
- `declare_parameter()`负责声明参数/默认值；`get_parameter()`负责读取当前值。

主要纠正：

- 学习者最初把参数名`global_frame`、`tile_size`误当成默认值；纠正后已掌握“参数名 / 默认值 / 成员变量”三者区别。

### 第2批：MultiMapManager启动顺序 + 参数初始化

源码：

1. `src/multi_map_manager.cpp`构造函数
2. `MultiMapManager::initializeParameters()`

已确认：

- 默认节点名：`multi_map_manager`。
- `initializeParameters()`在`initializeTF()`之后执行。
- `check_frequency`：参数名`check_frequency`、默认值`5.0`、读取类型`double`、成员`check_frequency_`。
- `enable_auto_switch`：默认值`true`、类型`bool`、通过`.as_bool()`读取。
- 参数日志能证明控制流已经越过前面的读取语句并到达日志处，但不能仅凭日志证明参数值符合业务要求。

### 第3批：可执行名、对象、变量名和节点名

源码：

1. `CMakeLists.txt`中的`add_executable(map_splitter_node src/map_splitter_node.cpp)`
2. `src/map_splitter_node.cpp`中的`main()`

已确认：

- 源码文件名：`map_splitter_node.cpp`。
- CMake可执行名：`map_splitter_node`。
- C++局部变量名：`node`。
- 默认ROS2节点名：`map_splitter_node`。
- `std::make_shared<multi_map_switcher::MapSplitterNode>()`真正创建的对象类型是`multi_map_switcher::MapSplitterNode`。
- `node`变量类型是`std::shared_ptr<multi_map_switcher::MapSplitterNode>`。

主要纠正：

- 对象类型不能写成`std::shared_ptr<...>`；共享指针是变量类型/管理方式，被创建的对象本身仍是`MapSplitterNode`。

## 独立总结

学习者完成了以下启动链整理：

```text
main()
→ make_shared创建MapSplitterNode对象
→ 自动执行MapSplitterNode构造函数
→ Node("map_splitter_node", options)确定默认节点名
→ declare_parameter声明参数与默认值
→ get_parameter读取当前参数值
→ 保存到C++成员变量
→ 日志提供执行位置和当时值的证据
→ rclcpp::spin(node)使用管理Node对象的shared_ptr
```

独立总结中的主要纠正：

- `get_parameter()`读取的是当前参数值，不保证等于声明时默认值；启动参数文件/命令行等可能覆盖默认值。
- `node`是C++局部变量名，不是参数名，也不是ROS2节点名。

结果：通过。

## 正式考试

### 第一轮：94/100

1. Node继承与节点名：20/20
2. 参数数据流：20/20
3. 源码文件名 / 可执行名 / 局部变量名 / 默认节点名：20/20
4. `make_shared`对象与共享指针：20/20
5. 日志与控制流：14/20

第5题纠正：

- 日志打印`tile_size: 30.0`只能证明程序执行到该日志且当时`tile_size_`输出为30.0，不能证明30.0一定符合业务要求。
- ROS2参数后续被修改，不等于成员变量`tile_size_`会自动同步；是否同步需要继续查看参数回调或再次`get_parameter()`等源码证据。

### 第二轮

- 不需要。
- 第一轮≥80且无关键概念错误。

## 今日已掌握

- `rclcpp::Node`基本继承关系。
- 默认ROS2节点名的源码证据。
- 参数名、默认值、当前值、成员变量之间的区别。
- 参数当前值的读取与类型转换。
- CMake可执行名、源码文件、C++变量和ROS2节点名的区别。
- Node对象类型与`shared_ptr`变量类型的区别。
- 日志能证明什么、不能证明什么。

## 仍需注意

- 不要把参数名当默认值。
- 不要把`shared_ptr<T>`当成被创建对象本身的类型。
- 不要把日志输出值等同于业务合法性验证。
- 不要假设ROS2参数变化会自动同步普通C++成员变量；必须看更新机制源码。

## 当前状态

- 基础讲义：完成。
- 真实源码阅读：3批，共6处源码，完成。
- 独立总结：通过。
- 正式考试第一轮：94/100。
- 关键概念错误：无。
- 是否通过：是。
- Day 15状态：✅ 完成。

## 下一窗口交接

- 下一日：Day 16 Service、Request、Response。
- Day 16应先完整讲解Service基本结构，再按每批两处真实源码阅读。
- 优先从`MapSplitterNode`真实`create_service`注册点与对应回调开始。
- 重点区分：Service对象、Service类型、Request、Response、回调函数。
- 不提前系统展开Topic、Action和Future异步等待。
