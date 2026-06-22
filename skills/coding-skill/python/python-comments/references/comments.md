# Python注释规范详细参考（PEP 257）

## 1. 模块Docstring

模块开头应包含docstring，说明模块用途、作者、版本等。

```python
"""用户认证模块。

提供用户登录、注销、权限验证等功能。依赖config模块获取认证配置。

Example:
    >>> from auth import authenticate
    >>> user = authenticate('admin', 'password')

Note:
    此模块需要数据库连接，使用前确保已初始化数据库。

Author: Zhang San
Version: 2.1.0
Since: 2024-01-15
"""

import hashlib
from typing import Optional
```

## 2. 类Docstring

类docstring应说明类的职责、属性、用法示例。

```python
class UserSession:
    """用户会话管理类。

    管理用户登录状态、会话超时、权限缓存。线程安全。

    Attributes:
        user_id (int): 用户唯一标识符。
        token (str): 会话令牌。
        expires_at (datetime): 会话过期时间。
        permissions (set): 用户权限集合。

    Example:
        >>> session = UserSession(user_id=1)
        >>> session.refresh()
        >>> if session.is_valid():
        ...     session.logout()

    Warning:
        会话令牌敏感，请勿在日志中输出。
    """

    def __init__(self, user_id: int) -> None:
        """初始化用户会话。

        Args:
            user_id: 用户唯一标识符，必须为正整数。

        Raises:
            ValueError: user_id非正整数时抛出。
        """
        if user_id <= 0:
            raise ValueError("user_id必须为正整数")
        self.user_id = user_id
        self.token = self._generate_token()
        self.permissions = set()
```

## 3. 函数Docstring

### 3.1 单行Docstring

适用于简单、自解释的函数。

```python
def is_even(n: int) -> bool:
    """判断整数是否为偶数。"""
    return n % 2 == 0


def get_current_timestamp() -> float:
    """返回当前Unix时间戳（秒）。"""
    return time.time()
```

### 3.2 多行Docstring（推荐格式）

对于复杂函数，使用多行docstring说明参数、返回值、异常。

```python
def calculate_discount(
    price: float,
    discount_rate: float,
    min_price: float = 0.0,
) -> float:
    """计算折后价格。

    根据折扣率计算商品折后价格，结果不低于最低限价。

    Args:
        price: 商品原价，必须大于等于0。
        discount_rate: 折扣率，范围0.0到1.0（如0.8表示八折）。
        min_price: 折后最低限价，默认为0。

    Returns:
        折后价格，保证不低于min_price。

    Raises:
        ValueError: price为负数或discount_rate不在有效范围时抛出。

    Example:
        >>> calculate_discount(100.0, 0.8)
        80.0
        >>> calculate_discount(100.0, 0.1, min_price=50.0)
        50.0
    """
    if price < 0:
        raise ValueError("price不能为负数")
    if not 0.0 <= discount_rate <= 1.0:
        raise ValueError("discount_rate必须在0.0到1.0之间")

    discounted = price * discount_rate
    return max(discounted, min_price)
```

## 4. 参数和返回值描述规范

### 4.1 Google风格（推荐）

```python
def fetch_users(
    department: str,
    include_inactive: bool = False,
    limit: Optional[int] = None,
) -> List[User]:
    """获取部门用户列表。

    Args:
        department: 部门名称，如'Engineering'、'Sales'。
        include_inactive: 是否包含已停用用户，默认False。
        limit: 返回用户数量上限，None表示不限制。

    Returns:
        用户对象列表，按入职时间升序排列。

    Raises:
        DepartmentNotFoundError: 部门不存在时抛出。
        DatabaseError: 数据库连接失败时抛出。
    """
    pass
```

### 4.2 NumPy风格

```python
def merge_dicts(base: dict, updates: dict) -> dict:
    """
    合并两个字典。

    Parameters
    ----------
    base : dict
        基础字典，作为合并起点。
    updates : dict
        更新字典，键值会覆盖base中的同名键。

    Returns
    -------
    dict
        合并后的新字典，不修改原字典。

    Examples
    --------
    >>> merge_dicts({'a': 1}, {'b': 2})
    {'a': 1, 'b': 2}
    """
    pass
```

## 5. 类型注解配合

Python 3.5+推荐使用类型注解，docstring中可简化参数类型描述。

```python
from typing import Dict, List, Optional, Union

def process_config(
    config: Dict[str, Union[str, int, bool]],
    env: str = "production",
    overrides: Optional[Dict[str, str]] = None,
) -> Dict[str, Any]:
    """处理配置并应用环境变量覆盖。

    Args:
        config: 配置字典，键为配置项名称。
        env: 目标环境名称。
        overrides: 环境变量覆盖项，键需与config中存在。

    Returns:
        处理后的完整配置字典。

    Note:
        类型注解已说明参数类型，docstring中无需重复。
    """
    pass
```

## 6. 行内注释规范

```python
# 正确示例：解释复杂逻辑的原因
result = []
for item in items:
    # 过滤掉已取消的订单，这些不计入统计
    if item.status != 'cancelled':
        result.append(item)

# 正确示例：提供业务背景说明
threshold = 0.85  # 根据历史数据分析，0.85为最优阈值

# 错误示例：描述显而易见的事
count = count + 1  # 计数器加1（无意义）

# 错误示例：注释与代码矛盾
value = 100  # 设置值为50（矛盾，会误导读者）
```

### 行内注释格式规则

- 与代码至少间隔2个空格
- 注释符号`#`后保留1个空格
- 简短明了，不超过72字符
- 解释"为什么"而非"是什么"

## 7. TODO/FIXME规范

```python
# TODO(author): 简短描述 - 可选的关联issue/PR编号
# TODO(zhangsan): 添加批量导入功能 - #123

# FIXME(author): 问题简述
# FIXME(lisi): 并发情况下可能产生竞态条件

# HACK(author): 临时方案说明
# HACK(wangwu): 临时绕过API限制，等待v2.0版本修复

# XXX: 需要关注的代码段，可能有问题
# XXX: 此处逻辑依赖外部状态，重构时需注意

# NOTE: 重要说明，提醒后续维护者
# NOTE: 此函数被多个模块调用，修改需谨慎

# DEPRECATED: 已废弃说明，推荐替代方案
# DEPRECATED: use calculate_v2() instead, will remove in v3.0
def calculate_legacy(data):
    pass
```

## 8. Sphinx格式（reStructuredText）

适用于需要生成HTML文档的项目。

```python
def send_email(
    to: str,
    subject: str,
    body: str,
    cc: Optional[List[str]] = None,
    attachments: Optional[List[str]] = None,
) -> bool:
    """发送邮件。

    通过SMTP服务发送邮件，支持抄送和附件。

    :param to: 收件人邮箱地址。
    :type to: str
    :param subject: 邮件主题。
    :type subject: str
    :param body: 邮件正文（纯文本）。
    :type body: str
    :param cc: 抄送列表，可选。
    :type cc: Optional[List[str]]
    :param attachments: 附件文件路径列表，可选。
    :type attachments: Optional[List[str]]
    :return: 发送成功返回True，失败返回False。
    :rtype: bool
    :raises SMTPException: SMTP连接或发送失败时抛出。
    :raises FileNotFoundError: 附件文件不存在时抛出。

    .. code-block:: python

        success = send_email(
            to="user@example.com",
            subject="Welcome",
            body="Hello!",
        )

    .. warning::
        不要在邮件正文中包含敏感信息如密码。
    """
    pass
```

## 9. 检查清单

### Docstring检查清单

- [ ] 模块顶部有docstring说明模块用途
- [ ] 公开类有docstring说明职责和属性
- [ ] 公开函数有docstring说明功能
- [ ] 复杂函数docstring包含Args、Returns、Raises
- [ ] Docstring首行为总结句，以句号结尾
- [ ] 多行docstring首行与后续内容间有空行
- [ ] 使用三双引号`"""`而非三单引号`'''`
- [ ] 参数描述格式统一（团队内保持一致）
- [ ] 有使用示例（Example或Examples）

### 行内注释检查清单

- [ ] 注释解释"为什么"而非重复代码
- [ ] 不描述显而易见的事实
- [ ] 注释与代码保持同步
- [ ] 风格统一：`# 注释内容`
- [ ] 长度不超过72字符
- [ ] TODO/FIXME包含作者信息

### 类型注解检查清单

- [ ] 参数有类型注解
- [ ] 返回值有类型注解
- [ ] 复杂类型使用typing模块
- [ ] docstring不再重复类型说明

### 禁止事项

- [ ] 无注释的复杂业务逻辑
- [ ] 过时注释（与代码不一致）
- [ ] 废弃代码仅注释不删除
- [ ] 中英文混杂的注释
- [ ] 注释中包含敏感信息

## 10. 常用工具

| 工具 | 用途 | 安装命令 |
|------|------|----------|
| pydocstyle | Docstring风格检查 | `pip install pydocstyle` |
| pydoc | 生成文档 | Python内置 |
| Sphinx | 文档生成系统 | `pip install sphinx` |
| darglint | Docstring参数检查 | `pip install darglint` |
| interrogate | Docstring覆盖率检查 | `pip install interrogate` |

### pydocstyle配置示例（.pydocstyle.ini）

```ini
[pydocstyle]
convention = google
add-ignore = D100,D104
max-line-length = 120
```

## 11. 参考资源

- PEP 257: Docstring Conventions
- PEP 484: Type Hints
- Google Python Style Guide - Docstrings
- NumPy Docstring Standard
- Sphinx Documentation
