# Day 09：map、set 和 unordered_map

## 今日目标

- 判断 `std::map<K, V>`、`std::unordered_map<K, V>` 的键类型、值类型和容器变量名。
- 理解 `std::set<T>` 保存唯一元素，元素本身也是查找键。
- 区分 `find()` 与 `operator[]`：前者只查找，后者在键不存在时会插入默认值。
- 理解 `it->first` 是键、`it->second` 是值，`*it` 用于取得 `set` 元素。
- 判断 `auto value` 与 `auto & value` 分别是复制还是引用。
- 比较 `map`、`unordered_map`、`set` 与 `vector` 的访问方式和顺序特点。

## 阅读代码

- `src/hpa/connected_components.cpp`
  - `std::map<int, int> label_map`
  - `label_map.find(root)`
  - `label_map[root] = compact_id++`
- `src/hpa/anchor_graph.cpp`
  - `std::set<double> cell_sizes`
  - `std::map<std::pair<int, int>, int> grid_to_cluster`
- HPA预处理相关头文件
  - `std::map<std::string, WalkableMask>`
  - `std::map<std::string, std::vector<int>>`
- HPA执行器相关代码
  - `std::unordered_map<int32_t, FailureCounterState>`
  - `auto & state = counters[key]`

## 今日学习

### 1. map保存键值关系

```cpp
std::map<std::string, int> scores;
```

- 完整类型：`std::map<std::string, int>`。
- 键类型：`std::string`。
- 值类型：`int`。
- 键唯一，给同一键重新赋值会更新原值，不会增加第二个相同键。
- `std::map`按照键的比较规则保持有序，不能把键误认为数字索引。

### 2. operator[]可能插入

```cpp
std::map<int, int> values;
int number = values[8];
```

键`8`不存在时，`operator[]`会创建一条新记录，并对值进行默认初始化。这里结果为：

```text
8 → 0
```

因此，`operator[]`不仅可能读取，也可能改变容器大小。

### 3. find()只查找

```cpp
auto it = values.find(5);
```

- `find(5)`不插入键`5`。
- `it == values.end()`表示没找到。
- `it != values.end()`表示找到。
- 找到`map`元素后，`it->first`是键，`it->second`是值。

### 4. set保存唯一元素

```cpp
std::set<int> ids;
ids.insert(3);
ids.insert(1);
ids.insert(3);
```

最终只有两个元素：`1`和`3`。

- `set`没有单独的键和值，元素本身就是查找键。
- `set`不保存重复元素，并按照元素比较规则保持有序。
- `set`没有`operator[]`，不能使用`ids[0]`按数字位置访问。
- `ids.find(3)`按元素值`3`查找；找到后`*it == 3`。

### 5. map值也可以是容器

```cpp
std::map<std::string, std::vector<int>> index;
index["A"].push_back(10);
index["A"].push_back(20);
```

- 键类型：`std::string`。
- 值类型：`std::vector<int>`。
- `index["A"]`是一个`std::vector<int>`对象。
- 键不存在时，先创建空`vector<int>`，随后向其中加入整数。

### 6. 复合键

```cpp
std::map<std::pair<int, int>, int> grid_to_cluster;
auto key = std::make_pair(2, 3);
grid_to_cluster[key] = 7;
```

表达的键值关系是：

```text
(2, 3) → 7
```

键是完整的`std::pair<int, int>`对象，不是两个独立键。

### 7. unordered_map

```cpp
std::unordered_map<int, FailureCounterState> counters;
```

与`map`相同：

- 保存唯一键及其对应值；
- 支持`find()`和`operator[]`；
- 键不存在时，`operator[]`会创建默认值。

主要区别：

- `map`按键有序；
- `unordered_map`不保证遍历顺序，通常通过哈希进行查找。

### 8. 复制值与引用值

```cpp
auto state = counters[5];
state.count = 7;
```

`state`是容器中值对象的副本。修改副本不影响`counters[5]`。

```cpp
auto & state = counters[5];
state.count = 7;
```

`state`是容器内部值对象的引用别名。修改`state.count`会直接修改`counters[5].count`。

## 容器对比

| 容器 | 保存内容 | 查找依据 | 重复 | 顺序 |
|---|---|---|---|---|
| `vector<T>` | 多个`T` | 数字位置 | 允许 | 插入顺序 |
| `map<K,V>` | 键值对 | 键 | 键唯一 | 按键有序 |
| `unordered_map<K,V>` | 键值对 | 键 | 键唯一 | 不保证顺序 |
| `set<T>` | 唯一元素 | 元素值 | 不允许 | 按元素有序 |

## 考试和纠错记录

### 第一轮：92/100，未通过

主要错误：认为`std::set`可以使用`ids[0]`访问。

纠正：

- `set`没有`operator[]`；
- 按元素值查找，不按数字位置访问；
- `find(value)`返回查找位置，找到后通过`*it`取得该元素。

### 教学顺序错误

最初计划明确将迭代器细节放到Day 10，但后续考试直接要求判断迭代器类型和类似指针的操作方式。该部分在考试前没有完成讲解，因此相关扣分和一次`50/100`评分作废，不记录为学习者错误。

Day 09只补充了使用关联容器所需的最小知识：

```cpp
it == container.end();  // 没找到
it != container.end();  // 找到
*it;                    // set元素
it->first;              // map键
it->second;             // map值
```

迭代器为什么支持`*`、`->`和`++`，以及遍历规则，放到Day 10系统学习。

### 第二轮重测：100/100，通过

已正确判断：

- 同一`map`键重新赋值会更新旧值；
- `find()`不插入，`operator[]`可能插入；
- `set`去重且不能按索引访问；
- `auto state`复制容器值，`auto & state`引用容器值；
- `map<string, vector<int>>`中的值本身是一个完整容器。

## 仍需保持的判断顺序

看到关联容器时依次判断：

1. 完整容器类型；
2. 键类型和值类型，或`set`的元素类型；
3. 当前操作是查找、插入还是更新；
4. 键不存在时是否会创建默认值；
5. 取得的是副本还是引用；
6. 容器是否保证遍历顺序。

## 完成状态

- 第一轮：92/100，关键错误已纠正
- 未讲内容导致的评分：已作废
- 第二轮重测：100/100
- 代码修改：无
- 最终状态：✅ Day 09通过

## 下一步

Day 10：循环、迭代器和索引访问。重点学习不同循环的控制过程、`auto`在循环中的推导、`begin/end`、`*`和`->`、越界风险，以及复制元素与引用元素的区别。
