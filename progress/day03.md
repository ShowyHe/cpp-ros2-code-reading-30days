# Day 03：声明、定义、命名空间和作用域

## 今日学习

- 函数声明用于告诉编译器函数的返回类型、名称和参数，通常写在 `.hpp` 中。
- 函数定义提供真正执行的代码，带有 `{}` 函数体，通常写在 `.cpp` 中。
- `::` 是作用域解析运算符，用于说明一个名字属于哪个结构体、类或命名空间。
- 命名空间用于组织代码并避免重名。
- 函数参数和局部变量只在对应函数范围内使用；成员变量属于结构体或类的对象。

## HPA源码

阅读：

- `include/multi_map_switcher/map_region.hpp`
- `src/map_region.cpp`
- `MapRegion::containsPoint()`
- `MapRegion::distanceToEdge()`
- `include/multi_map_switcher/hpa/runtime_version.hpp`

## 核心辨认

```cpp
bool containsPoint(double x, double y, double threshold = 0.0) const;
```

这是函数声明：没有函数体，并以分号结束。

```cpp
bool MapRegion::containsPoint(double x, double y, double threshold) const
{
  return x >= min_x - threshold;
}
```

这是函数定义：包含真正执行的函数体；`MapRegion::`说明该函数属于`MapRegion`。

```cpp
multi_map_switcher::hpa::kHpaRuntimeLogPrefix
```

这是变量的完整名称，依次位于`multi_map_switcher`和`hpa`两层命名空间中。

## 今日纠错

- `MapRegion::containsPoint`中的`MapRegion`是结构体，`containsPoint`才是函数名。
- `x`是函数参数，作用范围是本次函数调用。
- `result`等在函数体内定义的名字是局部变量，只能在该函数体内使用。
- `min_x`是`MapRegion`的成员变量，不属于局部变量作用域。
- 在`.cpp`中定义成员函数时，需要使用`MapRegion::`说明函数属于`MapRegion`。

## 成绩

- 第一轮：96分
- 第二轮：92分
- 状态：✅ Day 03通过
