# Day 11：enum、条件分支和提前返回

## 今日目标

- 区分枚举类型、枚举变量和枚举值。
- 理解 `enum class` 的作用域和类型约束。
- 判断 `if`、`else if`、`else` 与多个独立 `if` 的执行路径。
- 理解 `!`、`&&`、`||` 及短路求值。
- 判断 `return` 结束的是当前函数，而不是当前 `if` 或整个节点。
- 跟踪内部函数返回值是否被外层调用者检查。
- 识别赋值 `=` 与比较 `==` 的区别。

## 1. 枚举类型、变量和枚举值

```cpp
enum class NavResult
{
  SUCCEEDED,
  CANCELED,
  PLAN_FAILED,
  EXEC_FAILED
};

NavResult result = NavResult::SUCCEEDED;
```

判断：

- `NavResult`：枚举类型；
- `result`：`NavResult` 类型的变量；
- `NavResult::SUCCEEDED`：该类型允许保存的枚举值之一。

枚举变量可以在同一枚举类型的状态之间重新赋值：

```cpp
result = NavResult::PLAN_FAILED;
```

不能把枚举直接与字符串比较：

```cpp
result == "SUCCEEDED";                 // 错误
result == NavResult::SUCCEEDED;        // 正确
```

## 2. enum class 的特点

`enum class` 要求使用类型名限定枚举值：

```cpp
NavResult::SUCCEEDED
```

它能减少名称冲突，并提供更严格的类型检查。两个不同枚举类型中即使都有 `RUNNING`，也不是同一种值。

## 3. 条件分支

### 独立 if

```cpp
if (value > 0) {
  positive();
}

if (value > 10) {
  large();
}
```

两个条件都成立时，两个分支都可能执行。

### if / else if / else

```cpp
if (score >= 90) {
  excellent();
} else if (score >= 60) {
  pass();
} else {
  fail();
}
```

分支从上到下检查，一旦某个条件成立，后面的分支不再检查。因此范围更严格的条件通常放在前面。

## 4. 逻辑运算符

```cpp
!enabled                // 逻辑非
ready && valid          // 两边都为真
canceled || timeout     // 至少一边为真
```

复杂表达式应使用括号明确优先级：

```cpp
if (a || (b && c)) {
}
```

## 5. 短路求值

```cpp
if (ptr != nullptr && ptr->ready()) {
  start();
}
```

当 `ptr == nullptr` 时：

1. 左侧为 `false`；
2. `&&` 发生短路；
3. 不调用 `ptr->ready()`；
4. 整个条件为 `false`。

对应规则：

```text
false && 右侧表达式：右侧不执行
true  || 右侧表达式：右侧不执行
```

## 6. 提前返回

```cpp
bool process()
{
  if (!load()) {
    return false;
  }

  if (!split()) {
    return false;
  }

  save();
  return true;
}
```

`return` 立即结束当前函数。前面任一步失败后，后面的步骤不会执行。

`void` 函数可以使用单独的 `return;`：

```cpp
void timerCallback()
{
  if (!enabled) {
    return;
  }

  check();
}
```

## 7. 内层函数返回不自动结束外层函数

```cpp
bool prepare()
{
  return false;
}

void run()
{
  prepare();
  start();
}
```

`prepare()` 返回 `false` 只结束 `prepare()`。由于 `run()` 丢弃了返回值，仍然执行 `start()`。

外层要受影响，必须显式检查：

```cpp
if (!prepare()) {
  return;
}
```

## 8. 赋值与比较

```cpp
state = NavResult::SUCCEEDED;   // 赋值
state == NavResult::SUCCEEDED;  // 比较
state != NavResult::SUCCEEDED;  // 不等于
```

阅读条件时必须先确认当前使用的是 `=`、`==` 还是 `!=`。

## 9. 源码阅读方法

阅读布尔函数时按以下顺序：

1. 确认函数返回类型；
2. 找出所有 `return`；
3. 给每个返回出口标记触发条件；
4. 判断每个 `return` 后哪些代码不会执行；
5. 回到调用者，检查返回值是否被使用。

## 考试与纠错

### 第一轮：94/100

已正确掌握：

- 枚举类型、变量和值；
- 独立 `if` 与分支链；
- `!`、`||`；
- 提前返回；
- 导航结果状态分支。

主要纠错：`result == SUCCEEDED` 时，外层 `result != SUCCEEDED` 为假，因此不会调用 `retry()`，会直接到最后的 `return true`。

### 第二轮：88/100

主要错误：

1. 误认为枚举变量不能重新赋值；
2. 误认为空指针情况下仍会执行 `&&` 右侧。

### 定向补测：2/2

已正确说明：

- 枚举变量可以改为同类型的其他枚举值；
- `ptr == nullptr` 时，`&&` 短路，不调用右侧成员函数。

## 完成状态

- 第一轮：94/100
- 第二轮：88/100
- 定向补测：2/2
- 代码修改：无
- 最终状态：✅ Day 11通过

## 下一步

Day 12：文件读取、YAML 和错误处理。重点学习文件流、路径、`YAML::Node`、`as<T>()`、默认值、异常捕获、错误日志与返回值传播。