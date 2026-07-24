# Day 01：ROS2包结构与程序入口

## 今日学习

- 阅读：`package.xml`、`CMakeLists.txt`、`hpa_planner_main.cpp`、`hpa_executor_main.cpp`、`hpa_planner.launch.py`。
- `package.xml`：描述包名、版本、作者、依赖和构建类型；本包版本为 `1.0.1`，构建类型为 `ament_cmake`。
- `CMakeLists.txt`：规定源码如何编译、生成哪些库和可执行程序，以及链接关系。
- `add_library(... SHARED ...)`：生成动态共享库。
- `main()`：C++程序的执行入口。
- 启动链路：源码 → CMake编译 → 可执行程序 → `main()` → 创建ROS2节点 → `spin()`处理回调。

## 核心区别

- `hpa_planner_main.cpp`：C++源码文件。
- `hpa_planner_node`：编译生成的可执行程序。
- `/hpa_planner`：程序运行后在ROS图中的节点名称。
- `ros2 run`先启动可执行程序，随后程序创建并运行ROS2节点。

## 今日纠错

- `SHARED`不是“公共库”，而是动态共享库。
- 不能把可执行程序和ROS2节点名称混为一谈。
- `spin()`用于等待和处理回调，不是按固定频率执行整个节点。
- 后续考试涉及类、对象、智能指针和多节点进程，超出Day 01范围，不计入本日成绩。

## 成绩

- 最终评分：**85/100**
- 状态：**✅ Day 01通过**
