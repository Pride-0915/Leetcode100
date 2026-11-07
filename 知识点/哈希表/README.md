# 哈希表与 C++ `unordered` 容器

> 用于复习哈希表原理、C++ 常用接口和力扣中的典型用法。

## 目录

- [一、哈希表是什么](#一哈希表是什么)
- [二、哈希冲突与处理方法](#二哈希冲突与处理方法)
- [三、C++ 常用哈希容器](#三-c-常用哈希容器)
- [四、`unordered_map` 常用操作](#四-unordered_map-常用操作)
- [五、性能与容量管理](#五性能与容量管理)
- [六、手写拉链法哈希表](#六手写拉链法哈希表)
- [七、哈希表题型：两数之和](#七哈希表题型两数之和)
- [八、常见错误与选择建议](#八常见错误与选择建议)

## 一、哈希表是什么

哈希表通过哈希函数把键（key）映射到存储位置，以空间换取较快的查找、插入和删除。

- 常见操作平均时间复杂度为 $O(1)$，最坏可能退化到 $O(n)$。
- 哈希函数将键映射到桶（bucket）；不同键可能映射到同一桶。
- 哈希表不保证元素按键或插入顺序遍历。

## 二、哈希冲突与处理方法

当不同键得到相同的哈希位置时，就发生了哈希冲突。常见处理方法如下：

| 方法 | 思路 |
| --- | --- |
| 拉链法（链地址法） | 每个桶保存多个元素，冲突元素放在同一桶中 |
| 开放寻址法 | 冲突时按探查规则继续寻找其他空位置 |

这些是哈希表的常见实现策略。**C++ 标准不规定 `std::unordered_map` 或 `std::unordered_set` 必须采用哪一种底层实现**，不同标准库实现可能不同。

## 三、C++ 常用哈希容器

使用前包含对应头文件：

```cpp
#include <unordered_map>
#include <unordered_set>
```

| 容器 | 保存内容 | 适用场景 |
| --- | --- | --- |
| `std::unordered_map<Key, Value>` | 键值对 | 根据键快速查找或更新值 |
| `std::unordered_set<Key>` | 唯一键 | 判断元素是否出现、去重 |
| `std::map<Key, Value>` | 按键排序的键值对 | 需要有序遍历或范围查询 |
| `std::set<Key>` | 按键排序的唯一键 | 需要有序集合 |

`unordered_map` 和 `unordered_set` 的查找、插入、删除平均为 $O(1)$，最坏为 $O(n)$；`map` 和 `set` 通常为 $O(\log n)$。

## 四、`unordered_map` 常用操作

```cpp
#include <iostream>
#include <string>
#include <unordered_map>

std::unordered_map<std::string, int> counts;

counts["apple"] = 2;                 // 不存在则插入，存在则覆盖
counts.insert({"banana", 3});        // 键已存在时不覆盖
counts.emplace("orange", 4);         // 尝试原地构造元素

auto it = counts.find("apple");
if (it != counts.end()) {
    std::cout << it->first << ": " << it->second << '\n';
}

counts.erase("banana");              // 按键删除
std::cout << counts.size() << '\n';
```

### `[]`、`at()` 与 `find()`

| 写法 | 键不存在时的行为 | 常见用途 |
| --- | --- | --- |
| `mp[key]` | 插入该键，并将值初始化为默认值 | 插入或更新；计数 |
| `mp.at(key)` | 抛出 `std::out_of_range` | 确认键存在后的访问 |
| `mp.find(key)` | 返回 `mp.end()`，不插入 | 安全检查是否存在 |

不要用 `mp[key]` 做纯粹的存在性检查，因为它会修改容器：

```cpp
if (mp.find(key) != mp.end()) {
    // key 已存在
}
```

### 遍历

```cpp
for (const auto& [key, value] : counts) { // C++17 结构化绑定
    std::cout << key << ": " << value << '\n';
}
```

遍历顺序未指定，不要依赖它来获得排序结果。需要按键有序时使用 `std::map`。

### 自定义类型作为键

键类型需要提供哈希函数和相等比较。下面通过自定义哈希器传给容器：

```cpp
#include <cstddef>
#include <functional>
#include <unordered_map>

struct Point {
    int x;
    int y;
};

struct PointHash {
    std::size_t operator()(const Point& point) const {
        const auto hashX = std::hash<int>{}(point.x);
        const auto hashY = std::hash<int>{}(point.y);
        return hashX ^ (hashY << 1);
    }
};

struct PointEqual {
    bool operator()(const Point& lhs, const Point& rhs) const {
        return lhs.x == rhs.x && lhs.y == rhs.y;
    }
};

std::unordered_map<Point, int, PointHash, PointEqual> pointValues;
```

哈希值相同不代表键相同；容器还会用相等比较确认键是否相同。对于力扣题，如果键结构简单，也可以把多个字段编码成字符串或整数，但要保证编码不会产生歧义。

## 五、性能与容量管理

| 接口 | 含义 |
| --- | --- |
| `size()` | 元素数量 |
| `bucket_count()` | 当前桶数量 |
| `load_factor()` | 当前负载因子，即元素数量除以桶数量 |
| `max_load_factor()` | 自动扩桶使用的最大负载因子阈值 |
| `reserve(n)` | 为大约 `n` 个元素预留空间，可能触发重哈希 |
| `rehash(n)` | 请求调整桶数量，并可能重新分配元素 |

元素增长导致负载因子超过阈值时，容器可能扩桶并重新分配元素，这称为重哈希（rehash）。已知大致元素数量时，提前调用 `reserve` 可以减少反复扩桶的开销。

注意：哈希表的 `reserve(n)` 是预留容纳元素的空间，不是插入 `n` 个元素；它与 `std::vector::reserve(n)` 的目标类似，但作用于不同容器。

## 六、手写拉链法哈希表

下面实现一个只处理非负整数键和值的教学版本。每个桶保存一个链表；键重复时更新值。容量必须大于 0，找不到键时 `get` 返回 `-1`，因此本例也约定 `-1` 不能作为有效值。

```cpp
#include <list>
#include <utility>
#include <vector>

class MyHashMap {
private:
    std::vector<std::list<std::pair<int, int>>> buckets;

    std::size_t index(int key) const {
        return static_cast<std::size_t>(key) % buckets.size();
    }

public:
    explicit MyHashMap(std::size_t bucketCount = 1000)
        : buckets(bucketCount) {}

    void put(int key, int value) {
        auto& bucket = buckets[index(key)];
        for (auto& entry : bucket) {
            if (entry.first == key) {
                entry.second = value;
                return;
            }
        }
        bucket.emplace_back(key, value);
    }

    int get(int key) const {
        const auto& bucket = buckets[index(key)];
        for (const auto& entry : bucket) {
            if (entry.first == key) {
                return entry.second;
            }
        }
        return -1;
    }

    void remove(int key) {
        auto& bucket = buckets[index(key)];
        for (auto it = bucket.begin(); it != bucket.end(); ++it) {
            if (it->first == key) {
                bucket.erase(it);
                return;
            }
        }
    }
};
```

哈希函数将键映射到桶下标。遍历桶时仍需比较完整的键，因为多个键可能落入同一桶。实际应用优先使用标准容器；手写版本主要用于理解冲突处理和基本操作。

## 七、哈希表题型：两数之和

逐个扫描数组，用哈希表保存已经访问过的数及其下标。对当前数 `nums[i]`，先查找 `target - nums[i]`；找到就返回答案，否则再保存当前数。

```cpp
#include <unordered_map>
#include <vector>

std::vector<int> twoSum(const std::vector<int>& nums, int target) {
    std::unordered_map<int, int> indexByValue;

    for (int i = 0; i < static_cast<int>(nums.size()); ++i) {
        const int complement = target - nums[i];
        const auto it = indexByValue.find(complement);
        if (it != indexByValue.end()) {
            return {it->second, i};
        }
        indexByValue[nums[i]] = i;
    }
    return {};
}
```

- 时间复杂度：平均 $O(n)$。
- 空间复杂度：$O(n)$。
- 先查找再插入，避免同一个元素被重复使用。

## 八、常见错误与选择建议

- **把 `mp[key]` 当成只读查找**：键不存在时会插入默认值；判断存在请用 `find`。
- **假设哈希容器有序**：`unordered_map` 的遍历顺序未指定；需要有序结果请用 `map` 或排序。
- **认为平均 $O(1)$ 是最坏保证**：大量冲突时操作可能退化到 $O(n)$。
- **手写哈希时忽略冲突**：取模只负责映射到桶，不保证每个键对应唯一桶，仍需在桶内比较键。
- **用特殊值表示“未找到”却没有约定**：例如返回 `-1` 时，需确认 `-1` 不会是合法结果；否则改用额外状态或 `std::optional`。
- **忘记自定义键的相等比较**：哈希函数负责定位，相等比较负责判断键是否相同，两者缺一不可。

选择思路：只需快速去重或存在性查询时用 `unordered_set`；需要键到值的映射时用 `unordered_map`；需要有序遍历或范围查询时用 `set` / `map`。