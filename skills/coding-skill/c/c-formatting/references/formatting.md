# C语言排版格式规范详细参考

## 一、缩进规范

### 1.1 基本规则
- 统一使用4个空格缩进，禁止使用制表符
- 每级缩进增加4个空格
- 预处理指令不缩进，即使位于缩进块中

### 1.2 正确示例

```c
int calculate_sum(int *arr, int size)
{
    int sum = 0;
    for (int i = 0; i < size; i++)
    {
        sum += arr[i];
    }
    return sum;
}

#ifdef DEBUG
void debug_print(const char *msg)
{
    printf("[DEBUG] %s\n", msg);
}
#endif
```

### 1.3 错误示例

```c
int calculate_sum(int *arr, int size)
{
	int sum = 0;  // 错误：使用了制表符
  for (int i = 0; i < size; i++)  // 错误：使用了2空格
    {
        sum += arr[i];
    }
    return sum;
}
```

## 二、行宽规范

### 2.1 基本规则
- 单行代码不超过120字符
- 字符串字面量可适当放宽
- 长表达式应合理拆分

### 2.2 长行拆分规则
- 在运算符后断行，运算符在新行开头
- 函数参数较多时，每个参数独占一行
- 条件表达式过长时，在逻辑运算符处断行

### 2.3 正确示例

```c
// 长函数声明拆分
int process_user_request(struct user_context *ctx,
                         const char *request_type,
                         const char *request_data,
                         size_t data_length);

// 长条件表达式拆分
if (user_is_authenticated(ctx) &&
    user_has_permission(ctx, PERM_WRITE) &&
    resource_is_available(resource_id))
{
    process_request(ctx, request);
}

// 长字符串可以保持在一行
const char *error_message = "Error: Failed to allocate memory for user context structure. Please check available system resources.";
```

### 2.4 错误示例

```c
// 错误：超过120字符未拆分
int process_user_request(struct user_context *ctx, const char *request_type, const char *request_data, size_t data_length);

// 错误：在运算符前断行
if (user_is_authenticated(ctx)
    && user_has_permission(ctx, PERM_WRITE))
{
    process_request(ctx);
}
```

## 三、include排序规范

### 3.1 排序规则
1. 对应头文件（如foo.c对应的foo.h）放在第一位
2. C标准库头文件
3. 系统头文件（POSIX等）
4. 本项目头文件

### 3.2 分组要求
- 每组头文件之间空一行
- 组内按字母顺序排列
- 使用尖括号`<>`包含系统头文件
- 使用双引号`""`包含项目头文件

### 3.3 正确示例

```c
// user_manager.c
#include "user_manager.h"    // 对应头文件

#include <stdio.h>           // C标准库
#include <stdlib.h>
#include <string.h>

#include <pthread.h>         // 系统头文件
#include <unistd.h>

#include "config.h"          // 项目头文件
#include "database.h"
#include "logger.h"
```

### 3.4 错误示例

```c
// 错误：顺序混乱
#include <stdio.h>
#include "user_manager.h"
#include "logger.h"
#include <stdlib.h>
#include <pthread.h>
#include "config.h"
```

## 四、大括号风格

### 4.1 Allman风格规则
- 开括号`{`独占一行，与控制语句对齐
- 闭括号`}`独占一行，与开括号对齐
- 所有控制语句（if、for、while、switch、do）都使用大括号

### 4.2 正确示例

```c
int find_max(int *arr, int size)
{
    int max = arr[0];
    
    for (int i = 1; i < size; i++)
    {
        if (arr[i] > max)
        {
            max = arr[i];
        }
    }
    
    return max;
}

void handle_event(event_t *event)
{
    switch (event->type)
    {
        case EVENT_CLICK:
            process_click(event);
            break;
        
        case EVENT_KEYPRESS:
            process_keypress(event);
            break;
        
        default:
            log_unknown_event(event);
            break;
    }
}
```

### 4.3 错误示例

```c
// 错误：K&R风格
int find_max(int *arr, int size) {
    int max = arr[0];
    for (int i = 1; i < size; i++) {
        if (arr[i] > max) {
            max = arr[i];
        }
    }
    return max;
}

// 错误：省略大括号
if (value > 0)
    process_value(value);
```

## 五、空行规范

### 5.1 基本规则
- 函数之间使用2个空行
- 函数内逻辑块之间使用1个空行
- 变量声明与代码之间使用1个空行
- 文件末尾保留1个空行

### 5.2 正确示例

```c
int initialize_system(config_t *config)
{
    int result = 0;
    char *buffer = NULL;
    FILE *fp = NULL;
    
    // 验证配置参数
    if (config == NULL)
    {
        return ERROR_INVALID_PARAM;
    }
    
    // 分配缓冲区
    buffer = (char *)malloc(config->buffer_size);
    if (buffer == NULL)
    {
        return ERROR_NO_MEMORY;
    }
    
    // 初始化完成
    return SUCCESS;
}


void cleanup_system(void)
{
    // 清理资源
    release_memory();
    close_connections();
}
```

## 六、空格规范

### 6.1 基本规则
- 二元运算符两侧加空格：`a + b`、`x == y`
- 逗号后加空格：`func(a, b, c)`
- 关键字后加空格：`if (x)`、`while (y)`、`for (i = 0; i < n; i++)`
- 括号内侧不加空格
- 分号前不加空格，for语句分号后加空格

### 6.2 正确示例

```c
int calculate(int a, int b, int c)
{
    int result = (a + b) * c;
    
    if (result > 100)
    {
        result = result / 2;
    }
    
    for (int i = 0; i < 10; i++)
    {
        result += i;
    }
    
    return result;
}
```

### 6.3 错误示例

```c
// 错误：运算符两侧无空格
int result=(a+b)*c;

// 错误：逗号后无空格
int calculate(int a,int b,int c)

// 错误：括号内有空格
if ( result > 100 )

// 错误：关键字后无空格
if(x > 0)
```

## 七、指针声明风格

### 7.1 基本规则
- 星号`*`靠近变量名，而非类型名
- 多个指针声明时，每个变量独占一行

### 7.2 正确示例

```c
int *ptr;
char *name;
const char *message;

// 多个指针
int *array;
int *current;
int *end;
```

### 7.3 错误示例

```c
// 错误：星号靠近类型
int* ptr;
char* name;

// 错误：多个指针在同一行
int *array, *current, *end;

// 错误：混合声明
int *ptr, value;  // ptr是指针，value是int，容易混淆
```

## 八、switch语句格式

### 8.1 基本规则
- case标签与switch对齐，不额外缩进
- 每个case体缩进4空格
- 必须包含default分支
- case后空一格再写值

### 8.2 正确示例

```c
status_t process_command(command_t cmd, void *data)
{
    switch (cmd)
    {
        case CMD_INIT:
            initialize_module(data);
            break;
        
        case CMD_START:
            start_processing();
            break;
        
        case CMD_STOP:
            stop_processing();
            cleanup_resources();
            break;
        
        case CMD_RESET:
            reset_state();
            break;
        
        default:
            log_error("Unknown command: %d", cmd);
            return STATUS_ERROR;
    }
    
    return STATUS_OK;
}
```

## 九、格式检查清单

### 9.1 缩进检查
- [ ] 是否统一使用4空格缩进
- [ ] 是否禁止使用制表符
- [ ] 预处理指令是否从行首开始

### 9.2 行宽检查
- [ ] 单行是否不超过120字符
- [ ] 长行是否合理拆分
- [ ] 运算符是否在新行开头

### 9.3 include检查
- [ ] 对应头文件是否放在第一位
- [ ] 头文件是否按组分类排序
- [ ] 组间是否有空行

### 9.4 大括号检查
- [ ] 是否使用Allman风格
- [ ] 所有控制语句是否都使用大括号
- [ ] 大括号是否独占一行

### 9.5 空行检查
- [ ] 函数间是否有2空行
- [ ] 逻辑块间是否有1空行
- [ ] 变量声明与代码间是否有空行

### 9.6 空格检查
- [ ] 运算符两侧是否有空格
- [ ] 逗号后是否有空格
- [ ] 关键字后是否有空格
- [ ] 括号内侧是否无空格

### 9.7 指针检查
- [ ] 星号是否靠近变量名
- [ ] 多个指针是否分行声明

### 9.8 switch检查
- [ ] case标签是否与switch对齐
- [ ] 是否包含default分支
- [ ] 每个case是否有break或return

## 十、工具推荐

### 10.1 自动格式化工具
- **indent**：GNU indent，支持多种风格配置
- **clang-format**：LLVM工具，配置灵活
- **uncrustify**：支持多种语言，可定制性强

### 10.2 检查工具
- **cpplint**：Google风格检查
- **cppcheck**：静态分析，包含格式检查
- **clang-tidy**：LLVM静态分析工具

### 10.3 .clang-format配置示例

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
UseTab: Never
BreakBeforeBraces: Allman
ColumnLimit: 120
SortIncludes: true
IncludeBlocks: Preserve
```
