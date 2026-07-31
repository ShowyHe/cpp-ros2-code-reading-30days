# Day 07：第一周复盘与闭卷测试

## 复盘范围

- ROS2包结构、源码入口、可执行程序与运行时节点名
- 类型、变量、函数、返回值与作用域
- 声明、定义、命名空间与`::`
- `const`、值传递、引用传递与复制关系
- `class`、构造函数、初始化列表和访问权限
- 对象、原始指针、`unique_ptr`与`shared_ptr`

## 测试成绩

- 第一轮：80分
- 第二轮：89分
- `shared_ptr`定向补测：100分
- 最终状态：✅ 第一周复盘通过

## 已掌握内容

- 能区分源码文件、可执行程序和ROS2运行时节点名。
- 能识别函数声明、函数定义、返回类型、参数、局部变量和成员变量。
- 能解释`MapRegion::`表示成员归属，成员函数末尾`const`限制其修改普通成员变量。
- 能判断值传递、引用传递和赋值语句各自是否发生复制。
- 能识别构造函数、初始化列表以及`public`、`protected`、`private`权限。
- 能区分对象、原始指针和智能指针，并正确使用`.`和`->`。

## 主要错误与纠正

### 1. 参数不复制，但成员赋值会复制

```cpp
void setConfig(const SplitterConfig & config)
{
  config_ = config;
}
```

- 参数`config`是只读引用，传入时不复制`SplitterConfig`对象。
- `config_ = config`会把内容复制到成员变量`config_`。

### 2. `shared_ptr`对象数量与被管理对象数量

```cpp
auto a = std::make_shared<MapSplitter>();
auto b = a;
const std::shared_ptr<MapSplitter> & c = a;
```

- `MapSplitter`对象：1个。
- 真正的`shared_ptr`对象：2个，即`a`和`b`。
- `c`是`a`的只读引用，不是新的`shared_ptr`，不增加引用计数。
- `b = a`复制的是`shared_ptr`，不是复制`MapSplitter`对象，也不能只描述成复制裸地址。
- 可以通过`c->`修改其指向的`MapSplitter`对象，但不能修改`c`本身的绑定。

### 3. 类型名称要写准确

```cpp
auto node = std::make_shared<MapNode>();
```

对象类型是`MapNode`；`std::make_shared<MapNode>()`是创建方式，不是类型名称。

## 后续重点

第二周继续训练数据结构和跨文件调用。阅读容器代码时，重点区分：容器本身、容器元素、元素成员，以及复制与引用关系。
