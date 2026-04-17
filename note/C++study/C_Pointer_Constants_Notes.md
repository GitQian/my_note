# C语言学习笔记：常量指针与指针常量

在C语言中，`const` 关键字与指针结合使用时，位置不同其含义也大不相同。这对于初学者来说往往是一个难点。

## 1. 极简记忆法

利用中文的词性（形容词+名词）来理解：

*   **常量指针**（常量的指针）：指向的是“常量”，所以**不能改内容**。
*   **指针常量**（指针是个常量）：指针本身是“常量”，所以**不能改指向**。

---

## 2. 核心概念速记

可以通过“从右往左读”的方法来记忆：

1.  **常量指针 (Pointer to Constant)**：`const int *p;`
    *   读作：`p` 是一个指针，指向一个 `const int`。
    *   **限制**：不能通过 `p` 修改它指向的内容（`*p = val` 是错的），但 `p` 本身可以指向别的地址（`p = &other` 是对的）。

2.  **指针常量 (Constant Pointer)**：`int * const p;`
    *   读作：`p` 是一个 `const` 指针，指向 `int`。
    *   **限制**：`p` 指向的地址不能改（`p = &other` 是错的），但可以通过 `p` 修改它指向的内容（`*p = val` 是对的）。

3.  **指向常量的指针常量 (Constant Pointer to Constant)**：`const int * const p;`
    *   读作：`p` 是一个 `const` 指针，指向 `const int`。
    *   **限制**：内容和地址都不能改。

---

## 2. 示例代码 (`example.c`)

```c
#include <stdio.h>

int main() {
    int a = 10;
    int b = 20;

    // 1. 常量指针 (Pointer to Constant)
    const int *p1 = &a;
    printf("--- 常量指针 (const int *p1) ---\n");
    printf("初始值 a: %d, *p1: %d\n", a, *p1);
    
    // *p1 = 100; // 错误！
    p1 = &b;      // 正确：指针可以改向
    printf("修改指向后 *p1 (指向 b): %d\n\n", *p1);


    // 2. 指针常量 (Constant Pointer)
    int * const p2 = &a;
    printf("--- 指针常量 (int * const p2) ---\n");
    printf("初始值 a: %d, *p2: %d\n", a, *p2);

    *p2 = 100;    // 正确：内容可以改
    printf("修改值后 a: %d, *p2: %d\n", a, *p2);
    
    // p2 = &b;   // 错误！


    // 3. 只读指针 (Constant Pointer to Constant)
    const int * const p3 = &a;
    printf("--- 指向常量的指针常量 (const int * const p3) ---\n");
    printf("只读访问 *p3: %d\n", *p3);

    return 0;
}
```

---

## 3. 程序运行输出

```text
--- 常量指针 (const int *p1) ---
初始值 a: 10, *p1: 10
修改指向后 *p1 (指向 b): 20

--- 指针常量 (int * const p2) ---
初始值 a: 10, *p2: 10
修改值后 a: 100, *p2: 100
指针 p2 的地址固定为: 0x7fff9267f1b8

--- 指向常量的指针常量 (const int * const p3) ---
只读访问 *p3: 100
```

## 4. 总结对比表

| 类型 | 声明方式 | 能否修改内容 (`*p`) | 能否修改指向 (`p`) |
| :--- | :--- | :---: | :---: |
| **常量指针** | `const int *p` | ❌ 否 | ✅ 是 |
| **指针常量** | `int * const p` | ✅ 是 | ❌ 否 |
| **指向常量的指针常量** | `const int * const p` | ❌ 否 | ❌ 否 |
