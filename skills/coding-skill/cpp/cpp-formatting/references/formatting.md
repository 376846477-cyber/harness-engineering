# C++排版格式规范详细参考

## 1. 缩进规范

### 基本规则
- 使用4空格缩进，禁止使用制表符（Tab）
- 访问说明符（public/protected/private）与class关键字对齐，不额外缩进
- 命名空间内容可选择不缩进（节省水平空间）

### 示例

```cpp
// 正确：4空格缩进
class Example {
public:
    void method() {
        if (condition) {
            doSomething();
        }
    }

private:
    int value;
};

// 命名空间内容不缩进（可选风格）
namespace mylib {

class Example {
    // ...
};

} // namespace mylib
```

## 2. 行宽规范

### 基本规则
- 最大行宽：120字符
- 函数声明和定义可适当放宽至140字符
- 字符串字面量可不受限制

### 换行规则
```cpp
// 长参数列表换行，与第一个参数对齐
ReturnType ClassName::longMethodName(Type1 param1,
                                      Type2 param2,
                                      Type3 param3) {
    // 函数体
}

// 表达式换行，运算符放行首
if (veryLongCondition1
    || veryLongCondition2
    && veryLongCondition3) {
    // ...
}
```

## 3. Include排序规范

### 排序顺序（从上到下）
1. 对应头文件（本模块的头文件）
2. C标准库头文件
3. C++标准库头文件
4. 第三方库头文件
5. 本项目其他头文件

### 示例

```cpp
// example.cpp 的include顺序
#include "example.h"              // 1. 对应头文件

#include <stdio.h>                // 2. C标准库
#include <stdlib.h>

#include <algorithm>              // 3. C++标准库
#include <string>
#include <vector>

#include <boost/asio.hpp>         // 4. 第三方库
#include <gtest/gtest.h>

#include "project/common.h"       // 5. 本项目其他头文件
#include "project/utils/helper.h"
```

### 分组规则
- 每组之间空一行
- 同组内按字母顺序排列
- 使用 `<>` 引用系统/第三方头文件
- 使用 `""` 引用项目头文件

## 4. 大括号风格

### 推荐风格：K&R风格（Stroustrup变体）

```cpp
// 类定义、函数定义：开括号另起一行
class Example
{
public:
    Example();
    ~Example();
};

void function()
{
    // 函数体
}

// 控制语句：开括号在同一行
if (condition) {
    doSomething();
} else {
    doOther();
}

for (int i = 0; i < n; ++i) {
    process(i);
}

while (running) {
    update();
}

// 单行语句也必须使用大括号
if (x > 0) {
    return x;  // 即使单行也要大括号
}
```

## 5. 空行规范

### 文件级别
- 文件开头注释前空一行
- include区块之间空一行
- 类/函数之间空两行

### 函数内部
- 逻辑段落之间空一行
- 变量声明与代码之间空一行
- 文件末尾保留一个空行

```cpp
#include "example.h"

#include <string>


class Example {
public:
    Example();
    ~Example();
    
    void process();
    

private:
    std::string name_;
    int value_;
};


Example::Example()
    : name_("default")
    , value_(0)
{
    // 初始化代码
    
    setup();
}

void Example::process()
{
    // 第一步：验证输入
    
    validate();
    
    // 第二步：处理数据
    
    transform();
    
    // 第三步：输出结果
    
    output();
}

```

## 6. 空格规范

### 运算符
```cpp
// 二元运算符两侧加空格
int result = a + b * c;
bool flag = (x > 0) && (y < 100);

// 一元运算符不加空格
int neg = -value;
bool notFlag = !flag;
int* ptr = &variable;
int ref = *ptr;

// 逗号后加空格
void func(int a, int b, int c);
int arr[] = {1, 2, 3, 4, 5};
```

### 括号
```cpp
// 函数名与开括号之间不加空格
func(arg1, arg2);
object.method();

// 关键字与开括号之间加空格
if (condition) { }
for (int i = 0; i < n; ++i) { }
while (running) { }
switch (value) { }

// 括号内侧不加空格
int result = (a + b) * c;  // 正确
int wrong = ( a + b ) * c; // 错误
```

### 模板
```cpp
// 模板参数列表中逗号后加空格
template<typename T, typename U>
class Container { };

// 模板实例化不加空格
std::vector<int> vec;
std::map<std::string, int> dict;
```

## 7. 指针和引用风格

### 基本规则
- `*` 和 `&` 紧靠类型名，与变量名之间加空格
- 理由：类型修饰符属于类型，不属于变量

```cpp
// 正确：指针符号靠近类型
int* ptr;
const char* name;
std::unique_ptr<int> smartPtr;

// 正确：引用符号靠近类型
int& ref = variable;
const std::string& name = getName();

// 错误：符号靠近变量名
int *ptr;        // 不推荐
int &ref;        // 不推荐

// 多个指针声明
int* ptr1;
int* ptr2;
// 或使用typedef/using简化
using IntPtr = int*;
IntPtr ptr1, ptr2;
```

### 函数参数中的指针/引用
```cpp
// 输入参数：const引用
void process(const std::string& input);

// 输出参数：非const指针
void getResult(int* outValue);

// 输入输出参数：非const引用
void update(std::vector<int>& data);
```

## 8. 模板格式

### 模板声明
```cpp
// 单参数：一行
template<typename T>
class Container { };

// 多参数：每个参数一行
template<
    typename Key,
    typename Value,
    typename Hash = std::hash<Key>
>
class HashMap { };

// 模板参数命名
template<typename T>           // 类型参数
template<int N>                // 非类型参数
template<template<typename> class C>  // 模板模板参数
```

### 模板特化
```cpp
template<>
class Container<std::string> {
    // 特化实现
};
```

## 9. Lambda表达式格式

### 基本格式
```cpp
// 简短lambda：单行
auto add = [](int a, int b) { return a + b; };

// 较长lambda：多行
auto processor = [](const std::string& input) {
    std::string result = input;
    transform(result.begin(), result.end(), result.begin(), ::toupper);
    return result;
};

// 带捕获列表
auto func = [this, &context, value = std::move(data)]() {
    processData(context, value);
};
```

### 在算法中使用
```cpp
std::sort(vec.begin(), vec.end(),
    [](const Item& a, const Item& b) {
        return a.priority > b.priority;
    });

std::transform(input.begin(), input.end(),
    std::back_inserter(output),
    [](const std::string& s) {
        return s.length();
    });
```

## 10. 构造函数初始化列表格式

### 基本格式
```cpp
// 短列表：单行
Example::Example(int value) : value_(value), name_("default") { }

// 长列表：每个成员一行，冒号或逗号开头
ClassName::ClassName(Type1 param1, Type2 param2)
    : member1_(param1)
    , member2_(param2)
    , member3_(defaultValue)
    , member4_(computeValue())
{
    // 构造函数体
}

// 基类初始化
Derived::Derived(int value)
    : Base(value)
    , member_(value)
{
}
```

### 初始化顺序
```cpp
// 必须按声明顺序初始化
class Example {
public:
    Example(int a, int b)
        : a_(a)      // 先声明
        , b_(b)      // 后声明
    { }

private:
    int a_;  // 先声明
    int b_;  // 后声明
};
```

## 11. 检查清单

### 缩进检查
- [ ] 所有代码使用4空格缩进
- [ ] 无制表符混用
- [ ] 访问说明符正确对齐

### 行宽检查
- [ ] 无超过120字符的行（字符串除外）
- [ ] 长参数列表正确换行

### Include检查
- [ ] include顺序正确（对应头文件→C标准库→C++标准库→第三方→本项目）
- [ ] 使用正确的引号类型（<>或""）
- [ ] 组间有空行

### 大括号检查
- [ ] 风格统一（K&R或Allman）
- [ ] 单行语句也使用大括号
- [ ] 无空函数体（使用 = delete 或 = default）

### 空行检查
- [ ] 类之间有两空行
- [ ] 函数之间有一空行
- [ ] 逻辑段落间有空行
- [ ] 文件末尾有一空行

### 空格检查
- [ ] 二元运算符两侧有空格
- [ ] 关键字与括号间有空格
- [ ] 函数名与括号间无空格
- [ ] 逗号后有空格

### 指针/引用检查
- [ ] `*` 和 `&` 紧靠类型名
- [ ] 声明中符号与变量名之间有空格

### 模板检查
- [ ] 模板参数格式正确
- [ ] 多参数模板换行对齐

### Lambda检查
- [ ] 简短lambda单行
- [ ] 复杂lambda多行格式正确
- [ ] 捕获列表格式正确

### 构造函数检查
- [ ] 初始化列表格式正确
- [ ] 初始化顺序与声明顺序一致

## 12. 工具配置

### clang-format配置

```yaml
# .clang-format
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 120
UseTab: Never
BreakBeforeBraces: Attach
PointerAlignment: Left
SortIncludes: true
IncludeBlocks: Regroup
```

### 集成到编辑器
- VS Code: C/C++扩展自动格式化
- CLion: 内置clang-format支持
- Vim: vim-clang-format插件

### CI/CD检查
```bash
# 检查格式
clang-format --dry-run --Werror src/*.cpp

# 自动格式化
clang-format -i src/*.cpp
```
