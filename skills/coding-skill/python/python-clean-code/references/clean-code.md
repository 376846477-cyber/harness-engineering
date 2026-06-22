# Clean Code总览 - Python编码规范

## 概述

Clean Code是指在满足功能正确的前提下，把具有以下**七个核心特征**的代码称为Clean Code。这些特征构成了高质量Python代码的基础框架。

### 七个核心特征总览

| 特征 | 定义 | 优先级 | 核心目标 |
|-----|------|-------|---------|
| **可读** | 易于阅读和理解 | 高 | 降低认知负担 |
| **可维护** | 可修改、可扩展、可复用 | 高 | 降低变更成本 |
| **安全** | 防御恶意威胁 | 高 | 保护数据安全 |
| **可靠** | 正常运行、容错自愈 | 中 | 保证服务稳定 |
| **可测试** | 易于发现和定位故障 | 中 | 提升质量信心 |
| **高效** | 资源利用率高 | 中 | 优化运行效率 |
| **可移植** | 跨平台运行 | 低 | 扩大适用范围 |

### 特征优先级与权衡

**推荐优先级**：功能正确 > 可读 > 可维护 > 安全 > 可靠 > 可测试 > 高效 > 可移植

**权衡原则**：
- 可读性优先于性能优化：过早优化是万恶之源
- 安全性不可妥协：宁可牺牲性能也要保证安全
- 可测试性服务于可靠性和安全性
- 可移植性按需考虑，非必需时降低优先级

---

## 1. 可读性（Readability）

**定义**：代码可读是指代码易于阅读和理解的能力，是Clean Code的首要特征。

**核心价值**：降低认知负担，让代码自解释，减少沟通成本。

### Pythonic原则

Pythonic代码遵循PEP 8和Python之禅，具有以下特点：

**错误示例**:
```python
names = []
for user in users:
    if user.active:
        names.append(user.name)
```

**正确示例**:
```python
names = [user.name for user in users if user.active]
```

**Python之禅（The Zen of Python）**：
- 优美胜于丑陋（Beautiful is better than ugly）
- 明了胜于晦涩（Explicit is better than implicit）
- 简单胜于复杂（Simple is better than complex）
- 可读性很重要（Readability counts）

### 关键实践

1. **利用名称传递信息**：好名字自注释
   - 使用描述性命名，避免缩写和单字母变量（循环变量除外）
   - 遵循snake_case命名约定
   
2. **利用格式传递信息**：缩进、空行、风格一致
   - 使用4空格缩进，不混用Tab
   - 合理使用空行分隔逻辑块
   
3. **注释作为代码的补充**：解释Why而非What
   - 使用Docstring描述模块、类、函数
   - 避免冗余注释，保持注释与代码同步
   
4. **减少嵌套层级**：使用早返回、卫语句

   **错误示例**:
   ```python
   def process(user):
       if user:
           if user.active:
               return do_something(user)
       return None
   ```
   
   **正确示例**:
   ```python
   def process(user):
       if not user:
           return None
       if not user.active:
           return None
       return do_something(user)
   ```
   
5. **变量作用域尽量小**：就近定义、就近使用

### 检查清单
- [ ] 命名是否清晰表达意图？
- [ ] 代码格式是否符合PEP 8？
- [ ] 嵌套层级是否超过3层？
- [ ] 是否需要注释才能理解代码？

**相关Skill**：`python-readability` | `python-naming` | `python-formatting` | `python-comments`

---

## 2. 可维护性（Maintainability）

**定义**：可维护是指软件可修改、可扩展和可复用的能力，是降低长期维护成本的关键。

**核心价值**：隔离变化、降低耦合、提升复用，使代码能够适应需求变化。

### SOLID原则在Python中的应用

- **S - 单一职责**：一个类只做一件事
- **O - 开闭原则**：对扩展开放，对修改关闭
- **L - 里氏替换**：子类可替换父类
- **I - 接口隔离**：接口要小而专一
- **D - 依赖倒置**：依赖抽象而非实现

### 关键实践

1. **通过抽象隔离变化**：使用ABC、Protocol、多态
   ```python
   from abc import ABC, abstractmethod
   
   class DataProcessor(ABC):
       @abstractmethod
       def process(self, data):
           pass
   ```
   
2. **缩小依赖范围**：接口最少化、参数最少化
   - 函数参数不超过5个，必要时使用dataclass
   
3. **向着稳定的方向依赖**：依赖抽象而非具体实现
   
4. **引入中间层降低耦合**：Facade模式、适配器模式
   
5. **抽取公共部分**：DRY原则（Don't Repeat Yourself）

### 检查清单
- [ ] 修改是否影响局部化？
- [ ] 是否遵循DRY原则？
- [ ] 依赖关系是否清晰？
- [ ] 是否易于扩展新功能？

**相关Skill**：`python-maintainability`

---

## 3. 安全性（Security）

**定义**：代码安全性是指软件对恶意威胁的防护能力，是生产环境的基本要求。

**核心价值**：保护数据安全，防止攻击，维护用户信任。

### 安全三要素

| 要素 | 说明 |
|-----|------|
| **机密性** | 敏感数据不被泄露 |
| **完整性** | 数据不被篡改 |
| **可用性** | 服务正常运行 |

### 关键实践

1. **减少攻击面**
   - 最小权限原则
   - 禁用不必要功能
   
2. **输入校验及转义**
   - 永远不信任外部输入
   - 使用白名单验证
   
3. **敏感数据保护**
   - 不硬编码密钥和密码
   - 使用环境变量或密钥管理服务
   
4. **防御性编程**
   - 类型检查、边界检查
   - 异常处理要有意义

### 安全检查清单
- [ ] 是否有SQL注入风险？
- [ ] 是否有命令注入风险？
- [ ] 敏感数据是否加密存储？
- [ ] 是否硬编码了密钥或密码？

**相关Skill**：`python-security-input` | `python-security-crypto` | `python-security-serialization`

---

## 4. 可靠性（Reliability）

**定义**：可靠性是指系统正常运行、容错自愈的能力，是生产稳定性的保障。

**核心价值**：减少故障时间，提升用户体验，降低运维成本。

### 关键实践

1. **操作防呆设计**
   - 类型提示 + 静态检查
   - 参数校验
   
2. **系统过载保护**
   - 限流、熔断、降级
   - 超时设置
   
3. **防御式编程**
   - 预期异常处理
   - 边界条件检查
   
4. **资源管理**
   ```python
   # 使用with语句管理资源
   with open('file.txt', 'r') as f:
       content = f.read()
   ```
   
5. **核心流程依赖最小化**
   - 关键路径减少外部依赖
   - 有降级处理策略

### 检查清单
- [ ] 是否有容错机制？
- [ ] 是否有自愈机制？
- [ ] 资源是否正确释放？
- [ ] 异常是否妥善处理？

**相关Skill**：`python-reliability` | `python-exceptions`

---

## 5. 可测试性（Testability）

**定义**：可测试是指代码易于进行单元测试、集成测试，易于发现和定位故障。

**核心价值**：提升代码质量信心，快速发现回归问题，支持持续重构。

### 关键实践

1. **高内聚低耦合**
   - 单一职责
   - 依赖注入
   
2. **减少全局状态**
   - 避免全局变量
   - 避免单例滥用
   
3. **支持依赖注入**
   ```python
   # 推荐：依赖注入
   class UserService:
       def __init__(self, db: Database):
           self.db = db
   ```
   
4. **结果和过程可观测**
   - 日志记录
   - 指标监控
   
5. **异常日志适当记录**
   - 结构化日志
   - 包含上下文信息

### 检查清单
- [ ] 是否可以单独测试每个函数？
- [ ] 是否支持mock外部依赖？
- [ ] 是否有清晰的测试边界？
- [ ] 测试是否覆盖边界条件？

**相关Skill**：`python-testability`

---

## 6. 高效性（Performance）

**定义**：高效性是指代码在资源利用（CPU、内存、IO）方面的效率。

**核心价值**：优化运行效率，降低资源成本，提升用户体验。

### 性能优化原则

1. **测量后优化**：使用profiler定位瓶颈
2. **避免过早优化**：可读性优先
3. **选择正确的算法**：时间复杂度很重要

### 关键实践

1. **选择高效的算法和数据结构**
   - list vs set vs dict的选择
   - 时间复杂度考量
   
2. **合理使用生成器和迭代器**
   ```python
   # 节省内存
   def read_large_file(file_path):
       with open(file_path) as f:
           for line in f:
               yield line.strip()
   ```
   
3. **减少冗余或无效计算**
   - 缓存计算结果
   - 避免循环内重复操作
   
4. **合理使用异步编程**
   - IO密集型使用asyncio
   - CPU密集型使用multiprocessing
   
5. **IO优化**
   - 批量操作
   - 缓冲写入

### 检查清单
- [ ] 是否有不必要的循环？
- [ ] 是否有内存泄漏？
- [ ] 大数据是否使用生成器？
- [ ] IO操作是否批量处理？

**相关Skill**：`python-performance` | `python-concurrency`

---

## 7. 可移植性（Portability）

**定义**：可移植性是指代码在不同平台（操作系统、Python版本）上运行的能力。

**核心价值**：扩大代码适用范围，减少平台依赖问题。

### 关键实践

1. **使用pathlib处理路径**
   ```python
   # 推荐
   from pathlib import Path
   config_path = Path('config') / 'settings.json'
   ```
   
2. **明确指定字符集**
   ```python
   with open('file.txt', 'r', encoding='utf-8') as f:
       content = f.read()
   ```
   
3. **使用标准库和跨平台库**
   - 优先使用os.path、pathlib
   - 避免平台特定API
   
4. **时区和locale处理**
   - 使用datetime和pytz
   - 明确时区转换
   
5. **Python版本兼容性**
   - 明确支持的版本范围
   - 使用typing替代typing_extensions（如适用）

### 检查清单
- [ ] 是否依赖平台特定的API？
- [ ] 文件路径处理是否跨平台？
- [ ] 是否明确指定了字符编码？
- [ ] 是否考虑时区问题？

**相关Skill**：`python-portability`

---

## 规范速查表

| 场景 | 使用Skill | 核心要点 |
|-----|----------|---------|
| 命名检查 | `python-naming` | snake_case、描述性命名 |
| 格式排版 | `python-formatting` | PEP 8、4空格缩进 |
| 注释规范 | `python-comments` | Docstring、解释Why |
| 异常处理 | `python-exceptions` | 具体异常、有意义的处理 |
| 并发编程 | `python-concurrency` | GIL、asyncio、线程安全 |
| 输入安全 | `python-security-input` | 白名单校验、防注入 |
| 加密安全 | `python-security-crypto` | 密钥管理、安全算法 |
| 序列化安全 | `python-security-serialization` | pickle风险、YAML安全 |
| 可读性 | `python-readability` | Pythonic、减少嵌套 |
| 可维护性 | `python-maintainability` | SOLID、低耦合 |
| 可靠性 | `python-reliability` | 容错、资源管理 |
| 可测试性 | `python-testability` | 依赖注入、mock |
| 高效性 | `python-performance` | 算法、生成器、异步 |
| 可移植性 | `python-portability` | pathlib、UTF-8 |

---

## 完整检查清单

### 通用检查
- [ ] 代码是否易于阅读和理解？
- [ ] 修改是否影响局部化？
- [ ] 是否有安全漏洞？
- [ ] 是否有容错和自愈机制？
- [ ] 代码是否易于测试？
- [ ] 是否有不必要的性能浪费？
- [ ] 是否依赖平台特定的API？

### 可读性检查
- [ ] 命名是否清晰表达意图？
- [ ] 是否符合PEP 8规范？
- [ ] 嵌套是否超过3层？

### 安全性检查
- [ ] 输入是否经过校验？
- [ ] 是否存在注入风险？
- [ ] 敏感数据是否保护？

### 可靠性检查
- [ ] 异常是否妥善处理？
- [ ] 资源是否正确释放？

### 性能检查
- [ ] 是否有明显的性能瓶颈？
- [ ] 大数据是否使用生成器？

---

## 总结

Clean Code的七个特征相互关联、相互支撑：

- **可读性**是基础，降低认知负担
- **可维护性**是目标，降低长期成本
- **安全性**是底线，保护数据和用户
- **可靠性**是保障，确保稳定运行
- **可测试性**是手段，提升质量信心
- **高效性**是优化，合理利用资源
- **可移植性**是扩展，扩大适用范围

遵循这些特征，编写Clean Code，是每位Python开发者的专业追求。

---

## 自动化工具推荐

### 代码质量检查工具

| 工具 | 用途 | 安装 |
|-----|------|------|
| **ruff** | 快速lint+格式化（替代flake8+isort+black） | `pip install ruff` |
| **mypy** | 静态类型检查 | `pip install mypy` |
| **pylint** | 深度代码分析 | `pip install pylint` |
| **bandit** | 安全漏洞扫描 | `pip install bandit` |
| **pytest** | 单元测试框架 | `pip install pytest` |
| **pytest-cov** | 测试覆盖率 | `pip install pytest-cov` |

### pre-commit配置

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.8
    hooks:
      - id: bandit
        args: ["-c", "pyproject.toml"]
```

### CI集成建议

```yaml
# GitHub Actions示例
name: Code Quality
on: [push, pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install ruff mypy bandit pytest pytest-cov
      - run: ruff check .
      - run: ruff format --check .
      - run: mypy .
      - run: bandit -r .
      - run: pytest --cov=src --cov-fail-under=80
```
