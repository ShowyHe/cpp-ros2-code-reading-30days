# Day 12：文件读取、YAML 和错误处理

## 今日目标

- 区分 `std::ifstream`、`std::ofstream` 和 `std::fstream`。
- 理解文本模式、二进制模式、`>>`、`getline()`、`read()` 和 `write()`。
- 理解 `YAML::Node`、`node["key"]` 与 `as<T>()`。
- 判断 YAML 标量、列表和映射对应的 C++ 类型。
- 理解 `try / catch`、异常传播、`e.what()` 和 `return false`。
- 识别输出参数在失败前被部分修改的风险。
- 使用临时容器实现“全部成功后再提交结果”。

## 1. 文件流对象

```cpp
std::ifstream input(path);   // 读取
std::ofstream output(path);  // 写入
std::fstream file(path);     // 读写
```

判断顺序：类型、对象名、路径、打开模式。

```cpp
std::ifstream file(path, std::ios::binary);
```

`std::ios::binary`是打开模式参数，不是文件对象。

## 2. 检查文件是否打开

```cpp
std::ifstream file(path);

if (!file.is_open()) {
  std::cerr << "Cannot open file: " << path << std::endl;
  return false;
}
```

`!file.is_open()`表示文件没有成功打开。`return false`立即结束当前函数，后面的读取代码不执行。

只打印日志、不返回：

```cpp
if (!file.is_open()) {
  logError();
}

file >> value;
```

仍会继续执行`file >> value`。因此错误日志和控制流处理是两件事。

## 3. 文本读取与二进制读取

### 格式化文本读取

```cpp
std::string magic;
int width;
int height;

file >> magic >> width >> height;
```

`>>`跳过前导空白，并按照目标变量类型解析数据。

### 整行读取

```cpp
std::string line;
std::getline(file, line);
```

`getline()`读取到换行符为止。

### 原始字节读取

```cpp
buffer.resize(byte_count);
file.read(
  reinterpret_cast<char *>(buffer.data()),
  static_cast<std::streamsize>(buffer.size()));
```

`read()`将文件中的原始字节直接写进已分配好的内存。

## 4. 文件写入

```cpp
std::ofstream file(path);

if (!file.is_open()) {
  return false;
}

file << "resolution: " << resolution << '\n';
```

二进制写入：

```cpp
file.write(
  reinterpret_cast<const char *>(data.data()),
  static_cast<std::streamsize>(data.size()));
```

文件流对象离开作用域后会自动析构并关闭文件，通常不必手动`close()`。

## 5. YAML 基本结构

### 标量

```yaml
resolution: 0.05
enabled: true
name: map_1
```

可转换为`double`、`bool`和`std::string`。

### 列表

```yaml
origin: [1.0, 2.0, 0.3]
```

可转换为：

```cpp
std::vector<double>
```

### 映射和对象列表

```yaml
maps:
  - name: map_a
    file: map_a.yaml
  - name: map_b
    file: map_b.yaml
```

## 6. YAML::Node 与 as<T>()

```cpp
YAML::Node doc = YAML::LoadFile(path);
```

- `YAML::Node`：yaml-cpp定义的节点类型；
- `doc`：表示解析后的YAML结构；
- `doc["resolution"]`：取得对应字段节点；
- `as<double>()`：把节点转换成C++的`double`。

```cpp
double resolution = doc["resolution"].as<double>();
auto origin = doc["origin"].as<std::vector<double>>();
```

此时：

```text
resolution：double
origin：std::vector<double>
```

## 7. 默认值

```cpp
double threshold =
  config["switch_threshold"].as<double>(5.0);
```

- `double`：目标C++类型；
- `5.0`：节点不存在或无法使用时的回退值；
- `threshold`：新定义的C++变量；
- 该语句不会定义、修改或写回YAML字段。

## 8. try / catch 与异常

```cpp
bool loadValue(const std::string & path, int & value)
{
  try {
    YAML::Node config = YAML::LoadFile(path);
    value = config["value"].as<int>();
    return true;
  } catch (const std::exception & e) {
    std::cerr << e.what() << std::endl;
    return false;
  }
}
```

当YAML为：

```yaml
value: abc
```

执行流程：

```text
as<int>()无法转换
→ 抛出异常
→ return true不执行
→ 进入catch
→ e是异常对象的const引用
→ e.what()取得错误说明
→ return false
```

`catch`只处理对应`try`执行期间传播出来的匹配异常。

## 9. 异常传播

```cpp
void parse()
{
  YAML::Node node = YAML::LoadFile("bad.yaml");
}
```

如果`parse()`内部没有`catch`，异常会离开`parse()`并继续向调用者传播。调用者存在匹配的`catch`时才会捕获；若一直无人捕获，程序通常会终止。

## 10. 直接修改输出参数的半成品风险

```cpp
bool loadMaps(
  const YAML::Node & config,
  std::vector<MapRegion> & maps)
{
  try {
    maps.clear();

    for (const auto & node : config["maps"]) {
      MapRegion region;
      region.name = node["name"].as<std::string>();
      maps.push_back(region);
    }

    return true;
  } catch (const std::exception & e) {
    std::cerr << "Load maps failed: "
              << e.what() << std::endl;
    return false;
  }
}
```

假设原来`maps`有5个元素，前两个新节点成功，第三个失败：

```text
maps.clear()                 → 旧5个元素被删除
第1个push_back              → maps有1个新元素
第2个push_back              → maps有2个新元素
第3个as<std::string>()异常   → 进入catch
return false                 → maps仍留下2个半成品元素
```

函数返回失败，并不代表输出参数自动恢复原值。

## 11. 完整安全版本：临时容器，成功后一次提交

```cpp
bool loadMaps(
  const YAML::Node & config,
  std::vector<MapRegion> & maps)
{
  try {
    std::vector<MapRegion> loaded_maps;

    for (const auto & node : config["maps"]) {
      MapRegion region;
      region.name = node["name"].as<std::string>();
      region.file_path = node["file"].as<std::string>();
      loaded_maps.push_back(region);
    }

    // 只有上面的所有节点都解析成功，才会执行这一句。
    maps = std::move(loaded_maps);
    return true;
  } catch (const std::exception & e) {
    std::cerr << "Load maps failed: "
              << e.what() << std::endl;
    return false;
  }
}
```

第三个节点抛异常时：

```text
前两个结果只进入loaded_maps
→ 第三个节点抛异常
→ maps = std::move(loaded_maps)不会执行
→ 进入catch并返回false
→ 原来的maps保持不变
→ loaded_maps离开作用域并自动销毁
```

关键不是自动检查“应当有几个元素”，而是：

```text
全部解析成功 → 一次性提交到maps
任意一步失败 → maps完全不变
```

需要验证元素数量、必填字段或业务约束时，应在最终赋值前另行检查。

## 12. 路径与错误日志

相对路径通常要结合YAML文件所在目录解析。有效错误日志应尽可能包含：

- 失败操作；
- 实际路径或对象；
- 底层原因`e.what()`；
- 当前函数返回的失败结果。

日志用于告诉人原因，返回值用于告诉调用代码结果。

## 考试记录

### 第一轮：纠错后94/100

已掌握：

- 文件流类型与二进制打开模式；
- `YAML::Node`和`as<T>()`；
- 默认值不写回YAML；
- 类型转换失败后的异常流程；
- `maps.clear()`造成的半成品风险。

纠错点：最初未注意题目给出的`value: abc`，重新阅读后正确判断`as<int>()`抛异常、`return true`不执行并最终返回`false`。

### 第二轮：100/100

最初题目只给出临时容器片段，没有写出完整`try/catch`，导致异常去向不够明确。补充完整函数后，已正确判断：

- 不返回时错误日志后仍会继续执行；
- YAML列表转换后的完整类型和值；
- 默认值不会写回文件；
- 异常会跳过`try`中后续代码；
- 临时容器保护原输出参数，全部成功后才提交。

## 完成状态

- 第一轮：纠错后94/100
- 第二轮：100/100
- 代码修改：无
- 最终状态：✅ Day 12通过

## 下一步

Day 13：跨文件调用链。重点训练从调用点进入声明和定义，识别参数、返回值、成员变量副作用，并把入口函数、业务函数、工具函数和底层库连接成一条完整数据流。