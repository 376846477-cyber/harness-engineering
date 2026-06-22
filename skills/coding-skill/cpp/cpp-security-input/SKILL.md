---
name: cpp-security-input
description: C++编码规范 - 输入校验与安全规范。适用于所有C++代码开发场景：(1) 缓冲区溢出防护；(2) SQL注入防护；(3) 命令注入防护；(4) 任何外部输入处理。触发关键词：C++、注入、SQL、缓冲区溢出、buffer overflow、安全、校验、validation、输入、input
---

# C++输入校验与安全规范

基于SEI CERT C++ Coding Standard

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 禁止使用不安全的C函数（gets、sprintf、strcpy） |
| 规则1.2 | 禁止缓冲区溢出 |
| 规则1.3 | 禁止直接拼接SQL |
| 规则1.4 | 禁止使用system()执行不可信命令 |
| 规则1.5 | 文件路径校验前必须先标准化 |

## 缓冲区溢出防护

**错误示例**:
```cpp
char buffer[256];
gets(buffer);                    // 危险！无长度限制
sprintf(buffer, "%s", input);    // 危险！可能溢出
strcpy(buffer, input);           // 危险！可能溢出
```

**正确示例**:
```cpp
std::string buffer;
std::getline(std::cin, buffer);              // 安全，自动扩展
snprintf(buffer, sizeof(buffer), "%s", input); // 指定长度
strncpy(buffer, input, sizeof(buffer) - 1);   // 指定长度
buffer[sizeof(buffer) - 1] = '\0';            // 确保终止
```

## 使用std::string替代C字符串

```cpp
// 优先使用std::string
std::string name = get_user_input();

// 使用std::string_view避免拷贝
void process(std::string_view input);
```

## 详细规范

见 [references/security-input.md](references/security-input.md)
