---
name: c-security-input
description: C语言编码规范 - 输入校验与安全规范。适用于所有C语言代码开发场景：(1) 缓冲区溢出防护；(2) 命令注入防护；(3) 格式化字符串漏洞防护；(4) 任何外部输入处理。触发关键词：C语言、注入、缓冲区溢出、buffer overflow、安全、校验、validation、输入、input
---

# C语言输入校验与安全规范

基于SEI CERT C Coding Standard

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则1.1 | 禁止使用gets() |
| 规则1.2 | 禁止使用sprintf()，使用snprintf() |
| 规则1.3 | 禁止使用strcpy()，使用strncpy() |
| 规则1.4 | 禁止格式化字符串使用用户输入 |
| 规则1.5 | 禁止使用system()执行不可信命令 |
| 规则1.6 | 数组访问必须检查边界 |

## 缓冲区溢出防护

**错误示例**:
```c
char buffer[256];
gets(buffer);                    /* 危险！无长度限制 */
sprintf(buffer, "%s", input);    /* 危险！可能溢出 */
strcpy(buffer, input);           /* 危险！可能溢出 */
```

**正确示例**:
```c
char buffer[256];
fgets(buffer, sizeof(buffer), stdin);       /* 安全，指定长度 */
snprintf(buffer, sizeof(buffer), "%s", input); /* 安全，指定长度 */
strncpy(buffer, input, sizeof(buffer) - 1);    /* 安全，指定长度 */
buffer[sizeof(buffer) - 1] = '\0';             /* 确保终止 */
```

## 格式化字符串漏洞

**错误示例**:
```c
printf(user_input);    /* 危险！用户输入作为格式化字符串 */
```

**正确示例**:
```c
printf("%s", user_input);  /* 安全，固定格式化字符串 */
```

## 详细规范

见 [references/security-input.md](references/security-input.md)
