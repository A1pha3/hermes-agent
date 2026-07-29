# 4. 工具参数参考

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 了解所有内置工具的功能和参数
> - 正确调用工具完成各种任务
> - 理解工具的使用限制和注意事项

---

## 📖 导航
- 上一篇：[命令参考](./3-command-reference.md)
- 下一篇：[变更日志](./5-changelog.md)
- [返回目录](../SUMMARY.md)

---

## 概述

Hermes Agent 的内置工具按功能分为多个工具集，默认启用 core、file、web、code 工具集，其他工具集可以根据需要手动启用。所有工具调用的返回结果都是 JSON 格式。

---

## core 工具集（核心工具）

### `get_current_time` - 获取当前时间
**功能**：获取当前系统时间，支持指定时区。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `timezone` | string | 否 | Asia/Shanghai | 时区，比如 America/New_York |

**示例**：
```json
{
  "name": "get_current_time",
  "parameters": {
    "timezone": "Asia/Shanghai"
  }
}
```

**返回示例**：
```json
{
  "success": true,
  "data": {
    "timestamp": 1717248000,
    "datetime": "2024-06-01 12:00:00",
    "timezone": "Asia/Shanghai"
  }
}
```

---

### `calculate` - 数学计算
**功能**：执行数学表达式计算，支持四则运算、函数、单位转换等。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `expression` | string | 是 | 数学表达式，比如 "2 + 3 * 4"、"100 USD to CNY" |

**示例**：
```json
{
  "name": "calculate",
  "parameters": {
    "expression": "sqrt(16) + 2^3"
  }
}
```

**返回示例**：
```json
{
  "success": true,
  "data": {
    "expression": "sqrt(16) + 2^3",
    "result": 12
  }
}
```

---

## file 工具集（文件操作）

### `list_dir` - 列出目录内容
**功能**：列出指定目录下的文件和文件夹。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `path` | string | 否 | . | 目录路径 |
| `show_hidden` | boolean | 否 | false | 是否显示隐藏文件 |
| `recursive` | boolean | 否 | false | 是否递归列出子目录 |
| `max_depth` | integer | 否 | 3 | 递归最大深度 |

**示例**：
```json
{
  "name": "list_dir",
  "parameters": {
    "path": "/home/user/project",
    "show_hidden": false,
    "recursive": true,
    "max_depth": 2
  }
}
```

---

### `read_file` - 读取文件内容
**功能**：读取指定文件的内容，支持文本文件和代码文件。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `path` | string | 是 | 文件路径 |
| `offset` | integer | 否 | 起始行号，从1开始 |
| `limit` | integer | 否 | 读取的最大行数，默认读取全部 |

**示例**：
```json
{
  "name": "read_file",
  "parameters": {
    "path": "/home/user/project/main.py",
    "offset": 1,
    "limit": 50
  }
}
```

**注意事项**：
- 不支持读取二进制文件（图片、视频、可执行文件等）
- 大文件会自动截断，最多返回10000行内容

---

### `write_file` - 写入文件
**功能**：写入内容到文件，支持覆盖和追加模式。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `path` | string | 是 | | 文件路径 |
| `content` | string | 是 | | 要写入的内容 |
| `append` | boolean | 否 | false | 是否追加到文件末尾，false为覆盖 |
| `create_dir` | boolean | 否 | true | 上级目录不存在时是否自动创建 |

**示例**：
```json
{
  "name": "write_file",
  "parameters": {
    "path": "/home/user/project/test.py",
    "content": "print('Hello World')\n",
    "append": false
  }
}
```

**注意事项**：
- 覆盖文件前会自动备份到 `~/.hermes/backup` 目录
- 禁止写入系统关键目录（/etc、/root 等）

---

### `delete_file` - 删除文件或目录
**功能**：删除指定的文件或目录，默认需要用户审批。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `path` | string | 是 | 文件或目录路径 |
| `recursive` | boolean | 否 | 是否递归删除目录 |

**注意事项**：
- 默认需要用户手动审批才能执行
- 禁止删除系统关键文件和目录

---

### `search_file` - 搜索文件内容
**功能**：在指定目录下搜索包含关键词的文件，支持正则表达式。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `query` | string | 是 | | 搜索关键词，支持正则表达式 |
| `path` | string | 否 | . | 搜索路径 |
| `include` | string | 否 | | 包含的文件模式，比如 "*.py" |
| `exclude` | string | 否 | | 排除的文件模式，比如 "node_modules/*" |
| `case_sensitive` | boolean | 否 | false | 是否区分大小写 |
| `max_results` | integer | 否 | 50 | 最大返回结果数 |

**示例**：
```json
{
  "name": "search_file",
  "parameters": {
    "query": "def main",
    "path": "/home/user/project",
    "include": "*.py",
    "exclude": "venv/*"
  }
}
```

---

## web 工具集（网络操作）

### `web_search` - 网络搜索
**功能**：搜索网络获取最新信息，支持自定义搜索引擎。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `query` | string | 是 | | 搜索关键词 |
| `num_results` | integer | 否 | 10 | 返回结果数量，最多50 |
| `time_range` | string | 否 | | 时间范围：day/week/month/year |
| `language` | string | 否 | zh-CN | 搜索语言 |

**示例**：
```json
{
  "name": "web_search",
  "parameters": {
    "query": "Python 3.12 新特性",
    "num_results": 5,
    "time_range": "year"
  }
}
```

---

### `web_request` - 发送HTTP请求
**功能**：发送HTTP请求调用外部API或获取网页内容。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `url` | string | 是 | | 请求URL |
| `method` | string | 否 | GET | 请求方法：GET/POST/PUT/DELETE/HEAD |
| `headers` | object | 否 | | 请求头 |
| `params` | object | 否 | | 查询参数 |
| `json` | object | 否 | | POST JSON数据 |
| `data` | string | 否 | | POST表单数据 |
| `timeout` | integer | 否 | 10 | 请求超时时间（秒） |
| `verify_ssl` | boolean | 否 | true | 是否验证SSL证书 |

**示例**：
```json
{
  "name": "web_request",
  "parameters": {
    "url": "https://api.example.com/users",
    "method": "GET",
    "headers": {
      "Authorization": "Bearer token"
    },
    "params": {
      "page": 1,
      "page_size": 10
    }
  }
}
```

**注意事项**：
- 默认禁止访问内部网络地址，可以在配置中调整允许的域名列表
- 自动过滤敏感请求头（比如 Cookie、Authorization 等）不会出现在日志中

---

### `browser_navigate` - 浏览器自动化
**功能**：使用真实浏览器访问网页，支持动态内容渲染、截图、表单填写等。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `url` | string | 是 | 要访问的URL |
| `action` | string | 否 | 执行的操作：get/screenshot/click/fill/submit |
| `selector` | string | 否 | CSS选择器，用于定位元素 |
| `value` | string | 否 | 输入框填写的值 |
| `wait_for` | string | 否 | 等待元素出现的选择器 |
| `timeout` | integer | 否 | 操作超时时间（秒） |

**示例**：
```json
{
  "name": "browser_navigate",
  "parameters": {
    "url": "https://example.com/login",
    "action": "fill",
    "selector": "#username",
    "value": "user@example.com"
  }
}
```

---

## code 工具集（代码操作）

### `run_code` - 执行代码
**功能**：在沙箱环境中执行代码，支持多种编程语言。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `language` | string | 是 | | 编程语言：python/javascript/bash/java/go等 |
| `code` | string | 是 | | 要执行的代码 |
| `args` | array | 否 | | 命令行参数 |
| `timeout` | integer | 否 | 30 | 执行超时时间（秒） |
| `cwd` | string | 否 | . | 工作目录 |

**示例**：
```json
{
  "name": "run_code",
  "parameters": {
    "language": "python",
    "code": "print('Hello World')\nprint(1 + 2)",
    "timeout": 10
  }
}
```

**注意事项**：
- 代码在隔离的沙箱环境中执行，不会影响宿主系统
- 默认禁止网络访问和文件系统写入，可以在配置中调整权限
- 执行结果最多返回1000行内容

---

### `lint_code` - 代码检查
**功能**：静态检查代码错误和风格问题。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `language` | string | 是 | 编程语言 |
| `code` | string | 是 | 要检查的代码 |
| `config` | object | 否 | 检查配置 |

---

### `format_code` - 代码格式化
**功能**：格式化代码，统一风格。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `language` | string | 是 | 编程语言 |
| `code` | string | 是 | 要格式化的代码 |
| `style` | string | 否 | 格式化风格：pep8/google/airbnb等 |

---

## terminal 工具集（终端操作）

### `run_command` - 执行终端命令
**功能**：在终端执行命令，默认需要用户审批。

**参数**：
| 参数名 | 类型 | 必填 | 默认值 | 说明 |
|--------|------|------|--------|------|
| `command` | string | 是 | | 要执行的命令 |
| `args` | array | 否 | | 命令参数 |
| `cwd` | string | 否 | . | 工作目录 |
| `timeout` | integer | 否 | 60 | 执行超时时间（秒） |
| `background` | boolean | 否 | false | 是否后台运行 |
| `notify_on_complete` | boolean | 否 | false | 后台运行完成后是否通知 |

**示例**：
```json
{
  "name": "run_command",
  "parameters": {
    "command": "ls",
    "args": ["-la"],
    "cwd": "/home/user/project"
  }
}
```

**注意事项**：
- 默认需要用户手动审批才能执行危险命令
- 后台运行的命令可以通过 `/jobs` 命令查看状态

---

### `get_command_output` - 获取后台命令输出
**功能**：获取后台运行命令的输出内容。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `job_id` | string | 是 | 后台任务ID |

---

## delegate 工具集（子代理）

### `delegate_task` - 委托任务给子代理
**功能**：将复杂任务拆分成多个子任务，委托给子代理并行执行。

**参数**：
| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| `task` | string | 是 | 子任务描述 |
| `model` | string | 否 | 子代理使用的模型 |
| `context` | string | 否 | 任务上下文信息 |
| `expected_output` | string | 否 | 期望的输出格式 |

---

## 工具权限说明

| 工具 | 默认权限 | 是否需要审批 |
|------|----------|--------------|
| `get_current_time`、`calculate` | 启用 | 不需要 |
| `list_dir`、`read_file`、`search_file` | 启用 | 不需要 |
| `write_file` | 启用 | 不需要（备份后写入） |
| `delete_file`、`run_command` | 启用 | 需要审批 |
| `web_search`、`web_request` | 启用 | 不需要（受域名限制） |
| `run_code` | 启用 | 不需要（沙箱隔离） |
| `browser_navigate` | 禁用 | 需要审批 |

---

## ✍️ 练习
1. 练习使用 `read_file` 和 `write_file` 工具读写本地文件
2. 使用 `web_search` 工具搜索最新的技术资讯
3. 使用 `run_code` 工具执行一段简单的 Python 代码

---

## ❓ 常见问题
### Q：工具调用的结果会被保存到会话历史吗？
A：是的，工具调用的参数和返回结果都会保存到会话历史中。

### Q：可以自定义工具吗？
A：可以，参考 [新增内置工具](../development/3-add-tool.md) 文档开发自定义工具。

### Q：工具调用超时怎么办？
A：可以在配置中调整工具的默认超时时间，或者调用时指定 `timeout` 参数。

---

## 📖 导航
- 上一篇：[命令参考](./3-command-reference.md)
- 下一篇：[变更日志](./5-changelog.md)
- [返回目录](../SUMMARY.md)