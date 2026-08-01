# Day 08：struct、vector 和 string

## 今日目标

- 理解 `struct` 定义的是自定义类型，并区分类型、对象和成员变量。
- 理解 `std::vector<T>` 的完整类型、容器对象、元素类型和元素成员。
- 掌握 `push_back()`、`size()`、`empty()` 和 `clear()` 的基本含义。
- 理解 `std::string` 是字符串对象，并区分字符串与单个字符。
- 判断容器复制、普通引用和只读引用是否发生复制，以及修改会影响哪个对象。

## 阅读代码

### 结构体和成员容器

- `include/multi_map_switcher/map_splitter.hpp`
  - `MapMetadata`
  - `SubMapInfo`
  - `SplitterConfig`
  - `std::vector<uint8_t> source_image_`
  - `std::vector<SubMapInfo> sub_maps_`
  - `getSubMaps()`

### 创建并添加子地图元素

- `src/map_splitter.cpp`
  - `MapSplitter::calculateGrid()`
  - `SubMapInfo info`
  - `sub_maps_.push_back(info)`
  - `sub_maps_.clear()`
  - `sub_maps_.empty()`
  - `sub_maps_.size()`

### 引用参数修改调用者容器

- `src/config_loader.cpp`
  - `ConfigLoader::loadFromYaml()`
  - `std::vector<MapRegion> & maps`
  - `maps.clear()`
  - `maps.push_back(region)`

## 今日学习

### 1. struct 定义类型

```cpp
struct SubMapInfo
{
  std::string name;
  std::string yaml_path;
  int grid_x;
};
```

- `SubMapInfo` 是类型，不是某个具体子地图。
- `SubMapInfo info;` 创建一个类型为 `SubMapInfo` 的对象 `info`。
- `info.name`、`info.yaml_path` 和 `info.grid_x` 是该对象的成员。
- 对象使用 `.` 访问成员。

`struct` 和 `class` 都可以拥有成员变量、成员函数和构造函数。主要默认区别是：`struct` 默认 `public`，`class` 默认 `private`。

### 2. vector 容器和元素类型

```cpp
std::vector<SubMapInfo> sub_maps_;
```

- 完整类型：`std::vector<SubMapInfo>`。
- 容器变量名：`sub_maps_`。
- 元素类型：`SubMapInfo`。
- `sub_maps_[0]` 是第一个 `SubMapInfo` 元素。
- `sub_maps_[0].yaml_path` 是第一个元素中的 `yaml_path` 字符串成员。

不能把容器、容器元素和元素成员混为一谈。

### 3. push_back 复制元素

```cpp
SubMapInfo info;
sub_maps_.push_back(info);
```

`info` 是有名字的局部对象。`push_back(info)` 将其内容复制为容器中的新元素：

- 局部对象 `info` 与容器元素是两个独立对象。
- 随后修改 `info`，不会自动修改已经复制进容器的元素。
- 离开局部作用域后，`info` 被销毁；容器中的副本仍由 `sub_maps_` 持有。

### 4. vector 常用操作

```cpp
sub_maps_.size();
sub_maps_.empty();
sub_maps_.clear();
```

- `size()` 返回当前元素数量，不是容量或字节数。
- `empty()` 判断元素数量是否为零。
- `clear()` 销毁并清除全部元素，但 `sub_maps_` 这个 vector 对象仍然存在，可以继续添加元素。

### 5. string 是字符串对象

```cpp
std::string name = "map_01";
char letter = 'A';
```

- `std::string` 保存一串字符。
- `char` 保存单个字符。
- 字符串对象赋值或普通复制会得到独立的字符串内容。
- `std::to_string(gx)` 可以把整数转换为字符串，再通过 `+` 拼接路径或名称。

### 6. 容器复制和引用

```cpp
std::vector<SubMapInfo> maps = sub_maps_;
```

创建新的 vector 和新的元素副本。修改 `maps` 不会直接修改 `sub_maps_`。

```cpp
std::vector<SubMapInfo> & maps = sub_maps_;
```

不复制。`maps` 是同一个 vector 对象的别名，可以修改原容器。

```cpp
const std::vector<SubMapInfo> & maps = sub_maps_;
```

不复制，但不能通过 `maps` 修改容器或元素。

### 7. getSubMaps 返回只读引用

```cpp
const std::vector<SubMapInfo> & MapSplitter::getSubMaps() const
{
  return sub_maps_;
}
```

- 返回类型前的 `const`：调用者不能通过返回引用修改容器和元素。
- `&`：返回引用，不复制整个 vector。
- 成员函数末尾的 `const`：函数不能修改 `MapSplitter` 对象的普通成员变量。
- 真正拥有容器的是 `MapSplitter` 对象中的 `sub_maps_` 成员。
- 返回引用的有效期不能超过其所属 `MapSplitter` 对象的生命周期。

### 8. 引用参数修改实际传入的容器

```cpp
bool loadFromYaml(
  const std::string & config_path,
  std::vector<MapRegion> & maps,
  double & switch_threshold);
```

- `maps` 是普通引用，不复制调用者容器。
- 函数可以执行 `maps.clear()` 和 `maps.push_back(region)`。
- 修改的是调用时实际传入的那个容器。
- 仅看函数定义无法确定调用者变量名，必须检查调用位置。

## 考试记录

### 第一轮：97/100，通过

已正确掌握：

- `push_back(info)` 的复制关系。
- vector 复制与引用的区别。
- 只读引用允许读取但禁止修改。
- `std::vector<uint8_t>` 与 `std::vector<SubMapInfo>` 的元素类型。

表述纠正：

- 引用不是两个容器“存储同一份数据”，而是同一个对象的两个名字。
- `uint8_t` 是 8 位无符号整数，不是“8位数”。在当前代码中，一个元素用于保存一个图像像素值。

### 第二轮：通过

第二轮中曾把：

```cpp
std::vector<MapRegion> & maps
```

误答为“复制且不能修改”。纠正后确认：

- `&` 表示引用，不复制容器。
- 没有 `const`，所以可以 `push_back()` 和 `clear()`。

定向补测时，题目未在代码块中写明具体调用语句，却要求判断被修改的具体变量名，题目证据不足。该错误判定已撤回。最终规则是：

```cpp
load(sub_maps_);       // maps 引用 sub_maps_
load(original_maps);   // maps 引用 original_maps
```

引用参数修改的是调用时实际传入的对象。

## 仍需保持的判断习惯

阅读容器代码时，按以下顺序判断：

1. 完整容器类型是什么；
2. 元素类型是什么；
3. 当前表达式是容器、元素还是元素成员；
4. 当前语句是复制、普通引用还是只读引用；
5. 修改最终影响哪个实际对象；
6. 对象由谁持有，生命周期到哪里结束。

## 完成状态

- 第一轮：97/100
- 第二轮：通过
- 关键概念错误：已纠正
- 代码修改：无
- 最终状态：✅ Day 08 通过

## 下一步

Day 09：`map`、`set` 和 `unordered_map`。重点比较按索引访问与按键查找，并继续判断容器、键、值、复制和引用关系。
