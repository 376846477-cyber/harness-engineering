---
name: java-security-serialization
description: Java编码规范 - 序列化安全规范。适用于所有Java代码开发场景：(1) 序列化/反序列化代码；(2) Serializable类设计；(3) ObjectInputStream使用；(4) 敏感数据保护。触发关键词：Java、序列化、serializable、deserialize、ObjectInputStream、transient
---

# Java序列化安全规范

基于Java语言安全编程规范V3.2第7章"序列化和反序列化"和第8章"平台安全"

## 安全红线（必须遵守）

| 规则 | 要求 |
|-----|-----|
| 规则7.1 | 禁止序列化未加密的敏感数据 |
| 规则7.4 | 禁止直接将不可信数据进行反序列化 |
| 规则7.5 | 禁止序列化非静态的内部类 |
| 规则8.1 | 安全检查方法必须private或final |

## 序列化安全示例

### 敏感数据保护

**❌ 错误示例**:
```java
public class User implements Serializable {
    private String password;  // 敏感数据被序列化
    private String creditCard;
}
```

**✅ 正确示例**:
```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    // 使用transient保护敏感字段
    private transient String password;
    private transient String creditCard;

    // 加密后序列化
    public void setPassword(String pwd) {
        this.password = encrypt(pwd);
    }
}
```

### 反序列化安全

**✅ 正确示例**:
```java
// 使用白名单过滤
ObjectInputFilter filter = info -> {
    if (info.getSerialClass().isArray()) return ObjectInputFilter.Status.REJECTED;
    Class<?> clazz = info.getSerialClass();
    if (!ALLOWED_CLASSES.contains(clazz.getName())) {
        return ObjectInputFilter.Status.REJECTED;
    }
    return ObjectInputFilter.Status.ALLOWED;
};

ObjectInputStream in = new ObjectInputStream(bytes);
in.setObjectInputFilter(filter);
Object obj = in.readObject();
```

### 内部类序列化

**❌ 错误示例**:
```java
public class Outer implements Serializable {
    private class Inner implements Serializable {
        // 非静态内部类会携带外部引用
    }
}
```

**✅ 正确示例**:
```java
public class Outer implements Serializable {
    private static class Inner implements Serializable {
        // 静态内部类，不携带外部引用
    }
}
```

## 详细序列化安全规范

见 [references/security-serialization.md](references/security-serialization.md)