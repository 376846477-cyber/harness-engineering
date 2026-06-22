# 华为Java序列化安全规范详细参考

## 规则7.1 敏感数据序列化

- 禁止序列化未加密的敏感数据
- 敏感数据使用transient

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private String username;
    private String email;

    // ✅ 敏感数据不序列化
    private transient String password;
    private transient String creditCardNumber;

    // ✅ 或加密后序列化
    private transient String passwordHash;

    public void setPassword(String pwd) {
        this.passwordHash = hash(pwd);
    }
}
```

## 规则7.2 跨域数据保护

- 含敏感数据的对象跨信任域传递必须签名并加密

## 规则7.3 序列化绕过防护

- 防止通过序列化绕过安全管理器

## 规则7.4 反序列化安全（关键）

**❌ 禁止**:
```java
// ❌ 直接反序列化不可信数据
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject();
```

**✅ 必须使用白名单**:
```java
// ✅ 使用ObjectInputFilter白名单
ObjectInputFilter filter = info -> {
    Class<?> clazz = info.getSerialClass();
    if (clazz == null) return ObjectInputFilter.Status.ALLOWED;

    String name = clazz.getName();
    if (ALLOWED_CLASSES.contains(name)) {
        return ObjectInputFilter.Status.ALLOWED;
    }
    // 黑名单检查
    if (BLOCKED_CLASSES.contains(name)) {
        return ObjectInputFilter.Status.REJECTED;
    }
    return ObjectInputFilter.Status.REQUESTED;
};

ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(filter);
Object obj = ois.readObject();
```

## 规则7.5 内部类序列化

- 禁止序列化非静态内部类
- 非静态内部类会隐含外部类引用

```java
public class Outer implements Serializable {
    // ❌ 错误：非静态内部类
    private class Inner implements Serializable {}

    // ✅ 正确：静态内部类
    private static class Inner implements Serializable {}
}
```

## 规则8.1 安全检查方法

- 安全检查方法必须声明为private或final

```java
public class SecureClass {
    // ✅ 安全方法应该是private
    private boolean checkPermission() {
        // 安全检查逻辑
    }

    // ✅ 或final防止覆写
    public final boolean validate() {
        return checkPermission();
    }
}
```

## 规则8.2 自定义类加载器

- 必须调用超类的getPermission()

```java
public class CustomClassLoader extends ClassLoader {
    @Override
    protected Permission getPermission(Class<?> clazz) {
        return super.getPermission(clazz);
    }
}
```

## 序列化安全检查清单

1. ✅ 敏感数据使用transient
2. ✅ 反序列化使用白名单过滤
3. ✅ 静态内部类替代非静态
4. ✅ 安全检查方法private/final
5. ✅ 自定义类加载器调用getPermission
6. ✅ 跨域数据签名加密