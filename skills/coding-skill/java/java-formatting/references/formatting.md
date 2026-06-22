# 华为Java排版格式规范详细参考

## 规则3.1 大括号

- 左大括号`{`不换行
- 右大括号`}`独立一行
- 空块可以用`{}`简化

```java
// 正确
public void method() {
    if (condition) {
        doSomething();
    }
}

// 空块
if (condition) {}
```

## 规则3.2 缩进

- 使用4个空格缩进
- 制表符不用于缩进
- IDE需配置Tab为4空格

## 规则3.3 行宽

- 推荐不超过120个字符
- 超长需要换行

## 规则3.4 换行策略

### 方法参数换行
```java
public void method(String param1,
                   int param2,
                   long param3) {}
```

### 链式调用换行
```java
User user = userService.getUserById(id)
    .setStatus(UserStatus.ACTIVE)
    .save();
```

### 条件换行
```java
if (condition1
    && condition2
    || condition3) {}
```

## 规则3.5 空白字符

- 关键字后加空格：`if (x)`
- 运算符前后加空格：`a + b`, `a = b`
- 逗号后加空格：`method(a, b, c)`
- 冒号后加空格：`switch(x) { case Y: }`
- 不需要在`(`, `[`, `{`之前加空格

## 规则3.6 枚举格式

```java
public enum Status {
    PENDING,    // 逗号分隔
    PROCESSING,
    SUCCESS,    // 最后分号结束
    FAILED;

    public String getDesc() {
        // 方法
    }
}
```

## 规则3.7 switch语句

```java
switch (status) {
    case PENDING:
        handlePending();
        break;
    case SUCCESS:
        handleSuccess();
        break;
    default:
        handleUnknown();
        break;
}
```

## 规则3.8 注解位置

```java
// 类注解
@Entity
public class User {}

// 方法注解
@Override
public void doSomething() {}

// 字段注解
@Nullable
private String name;
```

## 格式检查清单

1. ✅ 左大括号不换行
2. ✅ 4空格缩进
3. ✅ 行宽不超过120
4. ✅ 方法参数逗号后换行
5. ✅ 关键字后有空格
6. ✅ 运算符前后有空格
7. ✅ 枚举值逗号分隔
8. ✅ case与break对齐