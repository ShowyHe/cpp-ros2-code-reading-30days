# Day 04：const、值传递和引用传递

## 学习目标

- 区分值传递、普通引用和只读引用。
- 判断函数调用时是否复制对象。
- 判断函数能否修改调用者原对象。
- 看懂返回只读引用和成员函数末尾的 `const`。

## 核心知识

```cpp
T x;          // 值传递：复制一份
T & x;        // 引用传递：不复制，可以修改原对象
const T & x;  // 只读引用：不复制，不能通过 x 修改原对象
func() const; // 该成员函数不能修改对象的普通成员变量
```

### HPA代码示例

```cpp
void MapSplitter::setConfig(const SplitterConfig & config)
{
  config_ = config;
}
```

- 参数 `config` 是只读引用，传入时不复制整个对象。
- `config_ = config` 会把内容复制到成员变量 `config_`。

```cpp
const std::vector<SubMapInfo> & MapSplitter::getSubMaps() const
{
  return sub_maps_;
}
```

- `&`：返回引用，不复制整个 `vector`。
- 返回类型前的 `const`：调用者不能通过返回引用修改 `sub_maps_`。
- 函数末尾的 `const`：该函数不能修改对象的普通成员变量。

## 关键纠错

- 值传递会产生副本；修改函数内的副本，不会改变调用者原对象。
- `const T &` 中，`&` 负责避免复制，`const` 负责禁止通过引用修改原对象。
- `maps` 引用 `sub_maps_` 时，不是“把值传给 maps”，而是让 `maps` 成为原对象的只读别名。
- 返回值前的 `const` 与成员函数末尾的 `const` 作用不同，不能混为一谈。

## 测试结果

- 第一轮：92分
- 第二轮：92分
- 状态：✅ 通过
