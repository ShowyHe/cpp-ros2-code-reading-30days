# Day 21：第三周 ROS2 接口影响分析

## 启动检查

- 仓库：`ShowyHe/cpp-ros2-code-reading-30days`
- 目标分支：`main`
- README开始状态：Day 20 ✅，Day 21 🟨
- 今日主题：综合 Node、Service、Request/Response、callback、接口兼容性、最小修改和验证设计
- 今日边界：不新增主要ROS2语法；不要求实际修改源码、编译、启动、接口调用或现场实测
- 规则调整：学习者明确提出本训练仓库不再把实操作为硬性每日验收，后续统一以源码阅读、工程判断、修改方案和验证设计为主

## 今日目标

- 从真实Service创建点确认Node、Service类型、Service名称和Server callback。
- 区分Request输入与Response输出。
- 区分“ROS接口结构变化”和“同一接口中的返回内容变化”。
- 对一个低风险需求完成影响面分析。
- 提出最小修改方案，而不是借小需求扩大重构范围。
- 设计成功路径与失败路径的验证矩阵，并区分“验证方案”和“实际运行证据”。

## 真实源码阅读

### 源码1：Service创建点

阅读了`MapSplitterNode`中的Service注册关系：

```cpp
switch_service_ =
  create_service<srv::SwitchMap>(
    "~/switch_map",
    std::bind(
      &MapSplitterNode::switchMapCallback,
      this,
      std::placeholders::_1,
      std::placeholders::_2));
```

学习者能够确认：

```text
Service类型 = srv::SwitchMap
Service名称 = ~/switch_map
Server callback = MapSplitterNode::switchMapCallback
```

### 源码2：`switchMapCallback()`

核心数据关系：

```cpp
request->map_name
```

是Client已经填写、Server callback读取的Request输入字段。

```cpp
response->success
response->message
```

是Server callback填写、之后返回Client的Response字段。

失败分支原始核心内容：

```cpp
response->success = false;
response->message =
  "Failed to load map via map_server";
```

学习者最终能够正确区分：

```text
Request
→ Client填写
→ Server读取

Response
→ Server填写
→ Client读取
```

## 接口结构与内容变化

候选需求：失败Response中加入请求的地图名。

候选最小修改：

```cpp
response->message =
  "Failed to load map '" +
  request->map_name +
  "' via map_server";
```

已确认该方案不修改：

```text
SwitchMap.srv
Service名称
Request字段
Response字段名
Response字段类型
成功分支控制逻辑
```

因此这是：

```text
同一个ROS2 Service接口
→ Response message内容变化
```

不是ROS2 Service接口结构变化。

### 学习过程中的纠正

学习者最初把“只改`response->message`”判断成“ROS接口结构变化”，后纠正为：

```text
字段内容变化
≠
接口结构变化
```

判断接口结构是否变化应检查：Service类型、`.srv`定义、字段名称、字段类型和Service名称等，而不是只看字符串内容是否变化。

## 最小修改设计

影响面整理：

```text
Node
→ MapSplitterNode

Service
→ ~/switch_map

Service类型
→ srv::SwitchMap

Request
→ map_name，不修改结构

Response
→ success / message，不修改结构

callback
→ switchMapCallback()

修改分支
→ callLoadMap()失败分支

成功分支
→ 不修改

Topic / TF / Action / 线程模型
→ 不涉及
```

成功分支已经包含地图名，因此按最小修改原则不需要顺手修改。

学习者在尝试手写候选代码时曾写成类似：

```text
response->map_name
```

并出现字符串拼接语法错误。已纠正：`map_name`属于Request，应读取`request->map_name`；C++字符串拼接应让字符串字面量与`std::string`对象使用`+`正确连接。

## 验证设计

虽然新规则不要求实际执行，但学习者能够设计完整验证链：

```text
编译
→ 节点启动
→ Service可见且类型正确
→ 发送成功Request
→ 检查成功Response
→ 制造失败Request/场景
→ 检查失败Response包含map_name
```

必须覆盖两条业务路径：

```text
成功路径
→ 证明原有正常行为没有回归

失败路径
→ 证明新Response内容符合需求
```

并且能够区分：

```text
业务路径
≠
观察点
```

例如“Server日志”和“Client收到Response”是观察位置，不等于两条不同业务路径。

新全局规则下，本日只要求**设计验证方案**；没有实际执行时，不得写成“已经编译/启动/实测通过”。

## 正式考试

### 第一轮：88/100

主要表现：

1. 能列出Node、Service、Request、Response、callback等核心影响面。
2. 能判断只改变`response->message`内容不属于ROS接口结构变化。
3. 能判断“编译成功 + Service可见”仍不足以证明业务行为正确，并指出需要成功/失败Request对应的Response证据。
4. 能说明成功路径用于检查无回归，失败路径用于证明需求生效。
5. 能拒绝“顺便重构callLoadMap、修改Service字段、优化线程模型”等扩大范围的方案，并坚持最小修改原则。

纠正点：

- “修改Response字段”应更准确说成“修改Response字段的内容”；字段本身及其类型没有变化。
- 验证证据必须具体到实际Request/Response行为；不能只用“看起来只影响修改范围”替代业务验证设计。

最终没有未纠正的关键概念错误。

## 规则调整与通过判断

学习者明确要求：本30天仓库后续不再把实际源码修改、编译、启动、部署和回归实测设为硬通过条件，因为这些实操在真实工作中已有大量实践。

因此全局规则与Day21–30计划同步调整为：

```text
真实源码阅读
→ 工程判断
→ 影响面分析
→ 最小修改方案
→ 验证矩阵设计
→ 正式知识/源码考试
```

实际执行仅作为可选扩展。

按调整后的Day21通过标准：

- 源码阅读：完成
- 影响面分析：完成
- 最小修改方案：完成
- 验证矩阵设计：完成
- 正式考试：88/100
- 关键概念错误：无未纠正项
- 实际修改/编译/运行：未执行，且不再作为硬性要求
- Day21状态：✅ 完成

## 已掌握

- Request由Client填写、Server读取；Response由Server填写、Client读取。
- 修改Response字符串内容不等于修改ROS接口结构。
- 先分析Node、接口、callback、调用方和数据流，再提出修改。
- 小需求坚持最小修改，不顺手扩大到线程、协议或重构。
- 成功路径和失败路径都属于验证设计范围。
- 编译、日志、Service可见性和实际Response属于不同层级证据。
- 没有实际运行时，应说“验证方案”，不能说“已经验证通过”。

## 仍需注意

- “字段内容变化”与“字段/接口结构变化”用词要区分。
- 业务路径与日志/Response等观察点不要混淆。
- 写C++字符串拼接时先确认字段属于Request还是Response，再检查字符串类型和引号位置。

## 下一窗口交接

- 本日状态：✅ 完成
- 正式考试成绩：88/100
- 作废题目：无
- 定向补测：Request读取、Response内容变化、成功/失败验证路径已完成纠正
- 已掌握：ROS2接口影响面、Request/Response方向、接口结构vs内容、最小修改、验证设计
- 仍然薄弱：术语需要继续保持精确，尤其“字段”与“字段内容”
- 已完成源码批次：Service注册点、`switchMapCallback()`、`SwitchMap`接口与失败Response分析
- 下一批应阅读的两处源码：Day22中一个`std::thread`创建点 + 一个共享成员/锁或atomic访问点
- 尚未确认：候选Day21修改没有真实执行；按新规则不影响通过状态
- 仓库与分支：`ShowyHe/cpp-ros2-code-reading-30days` / `main`
- 提交SHA：写入后回读补充
- 下一窗口必须从这里开始：Day22 `thread`、`mutex`、`atomic`基础与真实并发源码
- 下一日禁止提前展开：不深入无锁算法、内存序，不要求真实复现或修复竞态
