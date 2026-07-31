# Day 06：指针、对象和智能指针

## 今日目标

- 区分对象变量与指针变量。
- 判断指针指向哪个对象。
- 区分对象访问成员使用 `.`，指针访问成员使用 `->`。
- 理解空指针 `nullptr` 及使用前检查。
- 理解 `std::unique_ptr` 与 `std::shared_ptr` 的基本所有权关系。
- 判断函数参数是否复制 `MapSplitter` 对象、地址或智能指针。

## 学习内容

### 对象与原始指针

```cpp
MapSplitter splitter;
MapSplitter * ptr = &splitter;
```

- `splitter` 是 `MapSplitter` 对象。
- `ptr` 是保存 `MapSplitter` 对象地址的原始指针。
- `&splitter` 表示取得 `splitter` 的地址。
- 对象使用 `.`，指针使用 `->`。

### 空指针

```cpp
MapSplitter * ptr = nullptr;
```

`nullptr` 表示当前没有指向有效对象。解引用或通过空指针调用成员可能导致程序崩溃，使用前应先检查。

### unique_ptr

```cpp
auto splitter = std::make_unique<MapSplitter>();
```

- 创建一个 `MapSplitter` 对象。
- `splitter` 的类型是 `std::unique_ptr<MapSplitter>`。
- 通常只有一个所有者，不能直接复制。
- 唯一拥有者销毁时，其管理的对象自动销毁。

### shared_ptr

```cpp
auto first = std::make_shared<MapSplitter>();
auto second = first;
```

- 只创建一个 `MapSplitter` 对象。
- 创建两个共同管理该对象的 `shared_ptr`。
- 复制的是 `shared_ptr`，不是 `MapSplitter` 对象。
- 最后一个拥有者销毁后，被管理对象自动销毁。

## 参数辨认

```cpp
void func1(MapSplitter splitter);
void func2(MapSplitter & splitter);
void func3(MapSplitter * splitter);
void func4(std::shared_ptr<MapSplitter> splitter);
void func5(const std::shared_ptr<MapSplitter> & splitter);
```

- `func1`：复制 `MapSplitter` 对象。
- `func2`：不复制对象，参数是原对象的引用。
- `func3`：复制地址值，不复制对象；参数可以为空。
- `func4`：复制 `shared_ptr`，共享所有权；不复制对象。
- `func5`：既不复制对象，也不复制 `shared_ptr`；参数本身不可重新指向，但仍可通过它修改非 const 的 `MapSplitter` 对象。

## 测试记录

- 第一轮：88分，通过。
- 第二轮：92分，通过。
- 最终状态：✅ Day 06通过。

## 主要错误与纠正

1. 原始指针按值传参时，复制的是地址值，不是 `MapSplitter` 对象。
2. `shared_ptr` 按值传参或赋值时，复制的是智能指针并增加共享所有权，不只是简单描述为“复制地址”。
3. `unique_ptr` 不能直接复制，是因为不能出现两个拥有同一对象所有权的 `unique_ptr`，不是因为任意两个指针都不能指向同一地址。
4. `ptr = nullptr` 只修改指针本身，不会自动修改或删除它原来指向的栈对象。
