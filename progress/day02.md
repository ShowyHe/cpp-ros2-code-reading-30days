# Day 02：类型、变量、函数和返回值

## 今日目标

- 从代码中识别变量的类型、名称和初始值。
- 识别函数的返回类型、函数名和参数列表。
- 区分成员变量、函数参数和局部变量。
- 理解 `return` 将计算结果交回函数调用位置。

## 阅读源码

- `include/multi_map_switcher/map_region.hpp`
- `src/map_region.cpp`
- `MapRegion::containsPoint()`
- `MapRegion::distanceToEdge()`

## 今日学习

### 变量的基本组成

```cpp
double threshold = 0.5;
```

- `double`：类型。
- `threshold`：变量名。
- `0.5`：初始值。

### 函数的基本组成

```cpp
bool containsPoint(double x, double y, double threshold = 0.0) const;
```

- 返回类型：`bool`。
- 函数名：`containsPoint`。
- 参数：`double x`、`double y`、`double threshold`。
- `threshold` 的默认值为 `0.0`。

### 三类变量

```cpp
struct MapRegion
{
  double min_x{0.0};  // 成员变量

  double distanceToEdge(double x, double y) const;  // x、y是参数
};
```

```cpp
double MapRegion::distanceToEdge(double x, double y) const
{
  double dist_to_min_x = x - min_x;  // 局部变量
  return dist_to_min_x;
}
```

- 成员变量：定义在 `struct/class` 内、函数外，属于对象，例如 `min_x`。
- 参数：定义在函数括号内，值由调用者传入，例如 `x`、`y`。
- 局部变量：定义在函数体内，仅供本次函数执行使用，例如 `dist_to_min_x`。

### 返回值

```cpp
bool inside = region.containsPoint(2.0, 3.0);
double distance = region.distanceToEdge(2.0, 3.0);
```

- `containsPoint()` 返回的 `bool` 保存到 `inside`。
- `distanceToEdge()` 返回的 `double` 保存到 `distance`。
- 函数返回类型只约束最终返回值，不要求函数内部所有变量都是同一种类型。

## HPA源码辨认

```cpp
double MapRegion::distanceToEdge(double x, double y) const
```

- 返回类型：`double`。
- 所属结构体：`MapRegion`。
- 函数名：`distanceToEdge`。
- 参数：`double x`、`double y`。

## 今日错误与纠正

### 1. 题目缺少上下文

最初的题目只截取了局部代码，且未提前讲解成员变量、参数和局部变量，无法可靠判断变量类别。该轮测试作废，不计分。

### 2. 把所属结构体当成函数名

错误判断：`MapRegion` 是函数名。

正确判断：

```text
MapRegion：函数所属的结构体
distanceToEdge：函数名
```

### 3. 对成员变量取值理解不完整

`min_x{0.0}` 表示默认初始化为 `0.0`，但后续代码仍可能重新赋值，不能认为它永远等于 `0.0`。

### 4. 对函数返回类型理解不完整

错误理解：函数返回 `double`，内部变量就都必须是 `double`。

正确理解：只要求 `return` 最终提供的结果能够作为 `double` 返回，函数内部可以存在其他类型的变量。

### 5. 返回结果保存位置不够具体

不能只说“保存在调用者”。应明确指出结果保存到调用位置左侧的变量，例如 `inside` 或 `distance`。

## 测试记录

- 第一轮：92分。
- 第二轮：89分。
- 最初缺少上下文的测试：作废，不计分。

## 结论

- 状态：✅ Day 02通过。
- 已掌握：类型、变量名、初始值、函数名、返回类型、参数、成员变量、局部变量和返回值保存位置。
- 后续需要继续保持：阅读函数签名时，准确区分“所属类型”和“函数名”。