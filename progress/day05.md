# Day 05：class、构造函数和成员变量

## 学习目标

- 区分类、对象、成员变量和成员函数。
- 识别构造函数及其自动执行时机。
- 看懂构造函数初始化列表。
- 理解 `public`、`protected`、`private` 三种访问权限。
- 理解 `class` 与 `struct` 的默认访问权限差异。

## 核心知识

### 类与对象

```cpp
class MapSplitter
{
public:
  void loadMap();

private:
  int map_count_;
};

MapSplitter splitter;
```

- `MapSplitter`：类名，也是类型。
- `splitter`：具体对象，类型为 `MapSplitter`。
- `loadMap()`：成员函数。
- `map_count_`：成员变量，每个对象各自拥有一份。

### 构造函数

```cpp
MapSplitter()
: map_count_(0),
  loaded_(false)
{
}
```

- 构造函数名称与类名相同。
- 构造函数没有返回类型，连 `void` 也不写。
- 创建对象时自动执行。
- 初始化列表中的 `map_count_(0)` 和 `loaded_(false)`用于初始化成员变量，不是普通函数调用。

### 访问权限

| 访问位置 | public | protected | private |
|---|---:|---:|---:|
| 当前类自己的成员函数 | 可以 | 可以 | 可以 |
| 子类成员函数 | 可以 | 可以 | 不可直接访问 |
| 类外普通代码 | 可以 | 不可以 | 不可以 |

- 类外调用：不在该类自己的成员函数内部，通过对象访问成员，例如 `robot.start()`。
- `private`成员仍可被当前类自己的成员函数直接访问。
- 外部通常通过 `public`成员函数间接操作`private`成员。
- `class`未写权限时默认是`private`。
- `struct`未写权限时默认是`public`。

## 训练过程

最初测试后发现三种访问权限尚未系统讲解，因此该轮不计总分。补充讲解了：

- `public`、`protected`、`private`的访问范围。
- “类内部访问”和“类外部访问”的区别。
- 权限标签持续生效到下一个权限标签或类结束。
- 构造函数本身也受访问权限控制。
- `class`与`struct`的默认权限差异。

正式测试成绩：**98/100**。

## 主要纠正

- 构造函数不是“调用成员变量的函数”；它是与类同名、没有返回类型、创建对象时自动执行的特殊成员函数。
- `robot.speed_ = 0.5`属于访问成员变量，不是函数调用。
- `private`不是谁都不能访问，而是仅当前类自己的代码可以直接访问。
- 类外代码不能直接访问`private`成员，但可以调用`public`接口间接操作。

## 完成状态

- 成绩：98/100
- 状态：✅ 通过
