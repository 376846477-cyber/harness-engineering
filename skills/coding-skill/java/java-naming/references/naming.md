# Java命名规范详细参考

## 规则1.1 源文件编码格式

- 必须是UTF-8编码
- ASCII水平空格字符(0x20，即空格)是唯一允许出现的空白字符
- 制表符不用于缩进

## 规则1.2 标识符

- 所有标识符仅使用ASCII字母、数字、下划线 `_`
- 名称由正则表达式匹配 `\w{2,64}`，即长度2-64个字符
- 不应使用特殊前缀或后缀（如name_, mName, s_name, kName）

## 规则1.3 包名

- 全部小写
- 连续单词直接连写
- 顶级包名com，第二级为公司名
- 示例：`com.company.mobilecontrol.views`

## 规则1.4 类、枚举和接口

- **大驼峰命名** (UpperCamelCase)
- 测试类加Test后缀：`HashTest`, `UserServiceTest`
- 文件名为顶层类名.java
- 抽象类加Abstract或Base前缀：`AbstractUserService`, `BaseController`
- 接口根据特性加-able或-able后缀：`Runnable`, `Serializable`, `Readable`
- 异常加后缀Exception：`AccessException`

## 规则1.5 方法和常量

- **方法名**：小驼峰 (lowerCamelCase)
- **非常量字段**：小驼峰
- **静态常量**：全大写，下划线分割 `MAX_CONNECTIONS`
- **枚举值**：全大写，下划线分割 `STATUS_SUCCESS`
- **泛型类型变量**：单个大写字母 `E, T, K, V, R`

### 方法命名习惯

| 前缀 | 用途 |
|-----|------|
| get/getIs | 获取属性 |
| set | 设置属性 |
| is/has | 布尔判断 |
| init/destroy | 生命周期 |
| on | 回调方法 |
| 动词 | 动作操作 |

### 循环变量例外

i, j, k等循环变量允许单字符

## 命名检查清单

1. ✅ 标识符仅包含ASCII字母、数字、下划线
2. ✅ 长度在2-64字符之间
3. ✅ 类名大驼峰
4. ✅ 方法名小驼峰
5. ✅ 常量全大写下划线
6. ✅ 无特殊前缀后缀
7. ✅ 包名全小写
8. ✅ 测试类以Test结尾