# Day 10：循环、迭代器和索引访问

## 今日目标

- 判断普通 `for` 循环的初始化、条件和更新顺序。
- 理解嵌套循环、范围 `for` 和 `while` 的执行过程。
- 区分索引、元素、迭代器和容器本身。
- 理解 `begin()`、`end()`、`*it`、`it->member` 和 `++it`。
- 判断范围循环中的 `auto`、`auto &`、`const auto &` 是否复制元素。
- 识别 `vector` 索引越界和多个并行容器的长度风险。
- 理解 `break` 与 `continue` 对当前循环的影响。

## 1. 普通 for 循环

```cpp
for (int i = 0; i < 3; ++i) {
  use(i);
}
```

执行顺序：

1. `int i = 0` 只执行一次；
2. 每轮开始前判断 `i < 3`；
3. 条件为真时执行循环体；
4. 循环体结束后执行 `++i`；
5. 再次判断条件。

这里 `i` 依次为 `0、1、2`。当 `i == 3` 时条件为假，循环结束。

## 2. 索引边界

```cpp
std::vector<int> values{10, 20, 30};
```

容器大小是 3，合法索引只有：

```text
0、1、2
```

正确：

```cpp
for (std::size_t i = 0; i < values.size(); ++i) {
  use(values[i]);
}
```

错误：

```cpp
for (std::size_t i = 0; i <= values.size(); ++i) {
  use(values[i]);
}
```

当 `i == values.size()` 时，访问的位置已经不存在。`vector::operator[]`通常不主动检查越界，因此没有立即崩溃也不能说明代码正确。

需要主动检查时可以使用：

```cpp
values.at(i);
```

越界时会抛出 `std::out_of_range`。

## 3. 嵌套循环

```cpp
for (int gy = 0; gy < num_y; ++gy) {
  for (int gx = 0; gx < num_x; ++gx) {
    SubMapInfo info;
  }
}
```

内层循环会在每个 `gy` 下完整执行一遍。总次数是：

```text
num_y × num_x
```

`info`是内层循环体中的局部对象，每轮重新创建，本轮结束时销毁。

## 4. 二维坐标转一维索引

常见公式：

```cpp
int index = y * width + x;
```

宽度为 4 时：

```text
第0行：0 1 2 3
第1行：4 5 6 7
第2行：8 9 10 11
```

坐标 `(2, 1)` 对应索引 `1 * 4 + 2 == 6`。

安全访问还需要同时保证：

```text
0 <= x < width
0 <= y < height
```

## 5. 范围 for

```cpp
for (元素声明 : 容器) {
  循环体;
}
```

`auto`根据单个元素推导，不是根据整个容器推导。

假设：

```cpp
std::vector<SubMapInfo> maps;
```

### 复制元素

```cpp
for (auto map : maps) {
  map.name = "new";
}
```

`map`的类型是 `SubMapInfo`，每轮复制一个元素。修改副本不会影响容器原元素。

### 引用元素

```cpp
for (auto & map : maps) {
  map.name = "new";
}
```

`map`是当前元素的引用，不复制。修改会直接作用于 `maps` 中的真实元素。

### 只读引用

```cpp
for (const auto & map : maps) {
  use(map);
}
```

不复制，也不能通过 `map`修改原元素，适合只读访问结构体、字符串等对象。

| 写法 | 是否复制 | 能否修改原元素 |
|---|---:|---:|
| `auto x` | 是 | 否 |
| `const auto x` | 是 | 否 |
| `auto & x` | 否 | 是 |
| `const auto & x` | 否 | 否 |

## 6. 迭代器

迭代器是表示“当前位于容器哪个元素”的对象。它不是容器，也不一定是普通裸指针，但提供了类似指针的访问方式。

```cpp
std::set<int> ids{1, 3, 5};
auto it = ids.find(3);
```

`auto`根据 `find()` 的返回类型推导，因此 `it`是：

```cpp
std::set<int>::iterator
```

不是 `std::set<int>`。

### begin 和 end

```cpp
auto it = ids.begin();
```

`begin()`表示第一个元素的位置。

```cpp
ids.end();
```

`end()`表示最后一个元素之后的结束位置，不是最后一个元素，不能解引用。

### 基本操作

```cpp
*it;       // 取得当前元素
++it;      // 前进到下一个元素
it->name;  // 访问当前对象的成员，概念上等于 (*it).name
```

典型遍历：

```cpp
for (auto it = ids.begin(); it != ids.end(); ++it) {
  use(*it);
}
```

### find 判断

```cpp
auto it = ids.find(3);

if (it != ids.end()) {
  // 找到，*it == 3
}
```

没找到时：

```cpp
it == ids.end()
```

## 7. map 迭代器

```cpp
std::map<std::string, int> scores;
scores["Tom"] = 90;
auto it = scores.find("Tom");
```

找到后：

```cpp
it->first;   // 键："Tom"
it->second;  // 值：90
```

`map`元素本质上是键值对。键不能通过迭代器直接修改，值通常可以修改：

```cpp
it->second = 100;
```

## 8. 结构化绑定

```cpp
std::map<std::string, std::vector<int>> groups;

for (auto & [key, indices] : groups) {
  indices.push_back(10);
}
```

- `key`对应当前键；
- `indices`对应当前值 `std::vector<int>`；
- 因为有 `&`，不复制整个键值对；
- `key`不能修改；
- `indices.push_back(10)`修改容器中当前键对应的真实 `vector`。

没有引用时：

```cpp
for (auto [key, indices] : groups)
```

会复制当前键值对内容。

## 9. break 和 continue

### continue

```cpp
if (invalid) {
  continue;
}
```

跳过当前这一轮剩余代码，进入下一轮。

### break

```cpp
if (found) {
  break;
}
```

立即结束当前这一层循环。嵌套循环中只退出最近的一层。

## 10. 多个容器使用同一个索引

```cpp
for (std::size_t i = 0; i < names.size(); ++i) {
  use(names[i], paths[i]);
}
```

条件 `i < names.size()`只能证明 `names[i]`存在，不能证明 `paths[i]`存在。

还必须确认：

```cpp
i < paths.size()
```

或者循环开始前确认：

```cpp
paths.size() >= names.size()
```

工程代码中若同一个索引访问多个容器，需要继续追查这些容器在哪里填充、是否同步添加，以及长度是否经过检查。

## 11. 遍历时修改容器大小

```cpp
for (auto & value : values) {
  values.push_back(4);
}
```

这种写法危险。`vector::push_back()`可能重新分配内存，使当前迭代器和引用失效，后续访问可能产生未定义行为。

基本规则：遍历 `vector` 时，不要随意执行改变元素数量的 `push_back`、`erase`、`clear`；确实需要时必须分析迭代器失效规则。

## 考试记录

### 第一轮：84/100

正确掌握：

- `i < size()`与索引越界；
- `auto &`引用容器元素；
- `set`迭代器的查找和解引用；
- 结构化绑定中的键和值类型。

主要问题：

- 把 `for (auto item : container)`误判为不复制；
- 把 `for (auto & [key, value] : map)`误判为复制；
- 将正确判断式 `it != ids.end()`写成 `it != it.end()`。

定向补测后已纠正：

```text
auto item                 → 复制元素
auto & item               → 引用原元素
auto & [key, value]       → 不复制键值对
container.end()           → end()由容器调用
```

### 第二轮：92/100

已正确掌握：

- 引用循环会修改原容器；
- `const auto &`不能修改元素；
- `continue`跳过当前轮剩余代码；
- 多个容器使用同一索引时需要分别验证长度。

唯一扣分点：

```cpp
auto it = values.begin();
++it;
```

`begin()`先指向第一个元素（索引0），`++it`后指向第二个元素（索引1），不是“第一个元素”。

## 完成状态

- 第一轮：84/100，关键概念经定向补测纠正
- 第二轮：92/100
- 代码修改：无
- 最终状态：✅ Day 10通过

## 下一步

Day 11：`enum`、条件分支和提前返回。重点判断状态值、布尔表达式、`if / else if / else`执行路径，以及 `return`如何终止当前函数。