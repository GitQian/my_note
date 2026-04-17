# C++ 文件 I/O 学习笔记

C++ 通过标准库 `<fstream>` 提供了强大的文件操作能力。文件操作的核心是三个流类：`ifstream`（输入）、`ofstream`（输出）和 `fstream`（输入输出）。

---

## 1. 文件的打开与关闭

### 1.1 打开模式 (Open Modes)
在 `open()` 或构造函数中，可以通过位或运算符 `|` 组合多种模式：

| 模式标志 | 描述 |
| :--- | :--- |
| `ios::in` | 为读取而打开文件 |
| `ios::out` | 为写入而打开文件（会覆盖原有内容） |
| `ios::app` | 追加模式，所有写入都在文件末尾进行 |
| `ios::ate` | 打开后立即定位到文件末尾 |
| `ios::trunc` | 如果文件已存在，先清空内容（`ios::out` 的默认行为） |
| `ios::binary` | 以二进制模式打开（不处理换行符转换） |

### 1.2 安全检查
打开文件后，必须检查是否成功：
```cpp
if (!infile.is_open()) {
    std::cerr << "文件打开失败！" << std::endl;
}
```

---

## 2. 文件的写入 (Output)

使用 `ofstream` 对象，像使用 `cout` 一样使用 `<<` 运算符：
```cpp
std::ofstream outfile("test.txt", ios::out | ios::app);
outfile << "Hello World" << std::endl;
outfile.close();
```

---

## 3. 文件的读取 (Input) —— 多种方式对比

### 3.1 块读取 (read)
适合二进制文件或固定长度读取。
```cpp
char buffer[10];
infile.read(buffer, sizeof(buffer));
int count = infile.gcount(); // 获取实际读取的字节数
```

### 3.2 逐行读取 (getline)
- **C 风格**：`infile.getline(char_buf, size);`
- **C++ 风格（推荐）**：`std::getline(infile, str_obj);` —— 自动处理内存，更安全。

### 3.3 逐单词读取 (operator>>)
```cpp
string word;
while (infile >> word) { // 遇到空格、换行符停止
    cout << word << endl;
}
```

### 3.4 逐字符读取 (get)
```cpp
char c;
while ((c = infile.get()) != EOF) { ... }
```

### 3.5 一次性读取全文件 (Buffer Iterator)
最高效的一种写法，将整个文件流直接导入字符串：
```cpp
std::string content((std::istreambuf_iterator<char>(infile)), std::istreambuf_iterator<char>());
```

---

## 4. 文件指针定位 (Seeking)

C++ 允许随机访问文件内容。

- **`seekg` / `tellg`**：用于输入流（get）。
- **`seekp` / `tellp`**：用于输出流（put）。

### 4.1 常用操作：获取文件大小
```cpp
infile.seekg(0, ios::end);   // 定位到末尾
int size = infile.tellg();   // 获取当前偏移量（即大小）
infile.seekg(0, ios::beg);   // 重新回到开头
```

### 4.2 方向参数
- `ios::beg`：相对于文件开头。
- `ios::cur`：相对于当前位置。
- `ios::end`：相对于文件末尾。

---

## 5. 补充重要知识点

### 5.1 文本模式 vs 二进制模式
- **文本模式**：在 Windows 下，写入 `\n` 会被自动转换为 `\r\n`，读取时反之。
- **二进制模式**：不进行任何转换，读写内容与内存完全一致。处理图片、音视频或跨平台数据时必须使用 `ios::binary`。

### 5.2 流的状态检查
- `infile.eof()`：是否到达文件末尾。
- `infile.fail()`：是否发生逻辑错误（如类型不匹配）。
- `infile.bad()`：是否发生严重错误（如硬件故障）。
- `infile.clear()`：**重要！** 当读取到文件末尾后再想重新定位读取，必须先调用 `clear()` 重置状态位。

### 5.3 缓冲机制
文件操作通常是有缓冲的。调用 `outfile.flush()` 可以强制将缓冲区内容写入磁盘，而不必等到 `close()`。

---

## 6. 总结建议
1.  **优先使用 `std::string` 和 `std::getline`** 进行文本处理，避免字符数组溢出。
2.  **始终检查 `is_open()`**，防止程序因无效的文件指针崩溃。
3.  **使用 RAII**：在局部作用域定义流对象，利用析构函数自动调用 `close()`，即使发生异常也能保证资源释放。
