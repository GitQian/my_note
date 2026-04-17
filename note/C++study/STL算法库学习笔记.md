# C++ STL 算法库 (`<algorithm>`) 学习笔记

## 1. 核心思想：不要自己写 `for` 循环
C++ 之父 Bjarne Stroustrup 曾建议：**尽可能使用标准算法库，而不是手写循环。**

使用 STL 算法的优势：
1.  **效率高**：算法库经过编译器高度优化（有些甚至能利用 SIMD 指令）。
2.  **表达力强**：`std::find_if` 比一段嵌套的 `if-for` 逻辑更清晰地表达了“我在找什么”。
3.  **减少 Bug**：手写循环容易出现“差一错误”(off-by-one errors) 或迭代器失效问题。

---

## 2. 算法大类
STL 算法通常分为以下四大类：

### 2.1 非修改性操作 (Non-modifying)
这类算法只读取容器，不改变其内容。
*   `std::all_of` / `any_of` / `none_of`：检查条件。
*   `std::count_if`：满足条件的个数。
*   `std::find_if`：查找满足条件的第一个元素。
*   `std::mismatch`：比较两个序列。

### 2.2 修改性操作 (Modifying)
这类算法会改变容器内容。
*   `std::transform`：映射 (Map) 操作。
*   `std::copy` / `move`：拷贝或移动。
*   `std::replace`：替换元素。
*   `std::fill`：填充序列。

### 2.3 排序与分区 (Sorting & Partitioning)
*   `std::sort`：不稳定排序。
*   `std::stable_sort`：稳定排序。
*   `std::partition`：按条件分成两部分（如奇数放左边，偶数放右边）。
*   `std::nth_element`：找到第 N 大的数（效率极高，O(n)）。

### 2.4 数值算法 (`<numeric>`)
*   `std::accumulate`：累加或规约 (Reduce)。
*   `std::adjacent_difference`：计算相邻元素的差。
*   `std::partial_sum`：计算前缀和。

---

## 3. 灵魂伴侣：Lambda 表达式 (C++11)
算法库通常需要一个“谓词”(Predicate) 来决定行为。
```cpp
// 查找第一个大于 10 的数
auto it = std::find_if(vec.begin(), vec.end(), [](int n) { 
    return n > 10; 
});
```
*   `[]`：捕获列表。
*   `(int n)`：参数。
*   `{ return n > 10; }`：逻辑。

---

## 4. 必知必会：Erase-Remove 惯用法
由于 STL 算法设计为**与容器类型解耦**，它们只能通过迭代器操作，而**无法改变容器的大小**（因为迭代器不知道底层是 `vector` 还是数组）。

```cpp
// 错误示范：仅仅 remove 不会改变 vector 大小，只会把元素移到末尾
std::remove(v.begin(), v.end(), 2); 

// 正确写法：
v.erase(std::remove(v.begin(), v.end(), 2), v.end());
```
在 C++20 中，这被简化为 `std::erase(v, 2)`。

---

## 5. 性能提示
*   对于已排序的序列，使用 `std::binary_search` 或 `std::lower_bound` 效率更高（O(log n) vs O(n)）。
*   如果只需要找最大值，用 `std::max_element` 即可，没必要全量排序。
