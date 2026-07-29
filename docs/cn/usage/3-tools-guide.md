# 3. 内置工具使用说明

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 理解工具系统的工作原理和调用流程
> - 掌握所有常用工具的使用场景和参数
> - 学会配置工具权限和安全限制
> - 能够通过自然语言引导Agent正确调用工具完成复杂任务

Hermes Agent 内置了数十种功能强大的工具，覆盖文件操作、终端执行、网络搜索、浏览器自动化、代码执行等多个场景，Agent 会根据用户的需求自动调用合适的工具完成任务。

## 工具系统概述

**为什么需要工具系统？**
大语言模型本身的知识是截止到训练时间的，并且无法和真实世界交互，工具系统让Agent突破了这个限制，可以实时获取信息、执行操作、对接外部系统，大大扩展了Agent的能力边界。

### 工具调用流程
1. Agent 根据用户需求判断需要调用的工具
2. 生成符合工具 Schema 要求的调用参数
3. 执行工具调用，等待工具返回结果
4. 根据工具返回结果继续处理，或者直接返回给用户

### 工具 Schema 规范
所有工具都遵循 OpenAI Function Calling 格式，统一的 Schema 结构：
```json
{
  "name": "工具名称",
  "description": "工具功能说明",
  "parameters": {
    "type": "object",
    "properties": {
      "参数名1": {"type": "参数类型", "description": "参数说明"},
      "参数名2": {"type": "参数类型", "description": "参数说明", "enum": ["可选值1", "可选值2"]}
    },
    "required": ["必填参数名1", "必填参数名2"]
  }
}
```

### 返回值格式
所有工具统一返回 JSON 字符串：
- 成功返回：`{"success": true, "data": "结果数据", ...}`
- 失败返回：`{"error": "错误信息描述", "code": "错误码（可选）"}`

## 工具分类说明

### 1. Core 核心工具集
提供 Agent 核心能力：

#### todo_tool
功能：管理待办任务列表
参数：
- `action`：操作类型，可选值：add/remove/list/mark
- `content`：任务内容（add 时必填）
- `task_id`：任务 ID（remove/mark 时必填）
示例：
```
> 帮我添加一个待办任务：明天下午3点开项目会议
```
Agent 会自动调用：
```json
{"name": "todo_tool", "parameters": {"action": "add", "content": "明天下午3点开项目会议"}}
```

#### memory_tool
功能：管理长期记忆
参数：
- `action`：操作类型，可选值：add/remove/search/list
- `content`：记忆内容（add 时必填）
- `keyword`：搜索关键词（search 时必填）
示例：
```
> 记住我最喜欢的编程语言是 Python
```

#### session_search
功能：搜索会话历史
参数：
- `keyword`：搜索关键词
- `limit`：返回结果数量，默认10
示例：
```
> 搜索之前关于Python性能优化的对话
```

---

### 2. File 文件操作工具集
提供本地文件的读写、修改、查询能力：

#### file_read
功能：读取本地文件内容
参数：
- `path`：文件路径（必填）
- `offset`：起始行号，默认1
- `limit`：最大读取行数，默认2000
示例：
```
> 读取当前目录下的 README.md 文件
```

#### file_write
功能：写入内容到本地文件
参数：
- `path`：文件路径（必填）
- `content`：要写入的内容（必填）
- `append`：是否追加写入，默认false（覆盖写入）
示例：
```
> 帮我创建一个 hello.py 文件，内容是 print("Hello Hermes")
```

#### file_list
功能：列出目录下的文件和子目录
参数：
- `path`：目录路径，默认当前目录
- `recursive`：是否递归列出，默认false
- `pattern`：文件名过滤模式，可选
示例：
```
> 列出 src 目录下的所有 Python 文件
```

#### file_patch
功能：增量修改文件内容
参数：
- `path`：文件路径（必填）
- `old_str`：要替换的旧内容（必填）
- `new_str`：替换后的新内容（必填）
示例：
```
> 把 hello.py 中的 "Hello Hermes" 修改为 "Hello World"
```

---

### 3. Terminal 终端执行工具集
提供本地 shell 命令执行能力：

#### terminal_exec
功能：执行 shell 命令
参数：
- `command`：要执行的 shell 命令（必填）
- `cwd`：执行命令的工作目录，默认当前目录
- `timeout`：命令超时时间（秒），默认30
- `background`：是否后台运行，默认false
示例：
```
> 查看当前 Python 版本
```
Agent 会自动调用：
```json
{"name": "terminal_exec", "parameters": {"command": "python --version"}}
```

#### background_job
功能：管理后台运行的任务
参数：
- `action`：操作类型，可选值：list/output/kill/wait
- `job_id`：任务 ID（output/kill/wait 时必填）
示例：
```
> 查看所有后台运行的任务
```

---

### 4. Web 网络工具集
提供网页搜索、内容提取能力：

#### web_search
功能：网络搜索信息
参数：
- `query`：搜索关键词（必填）
- `num_results`：返回结果数量，默认10
- `time_range`：时间范围，可选值：day/week/month/year
示例：
```
> 搜索最近一周关于AI大模型的最新消息
```

#### web_extract
功能：提取指定网页的正文内容
参数：
- `url`：网页 URL（必填）
- `raw`：是否返回原始 HTML，默认false（返回净化后的正文）
示例：
```
> 提取 https://example.com 页面的内容
```

---

### 5. Browser 浏览器自动化工具集
提供真实浏览器自动化操作能力：

#### browser_navigate
功能：导航到指定 URL
参数：
- `url`：目标 URL（必填）
- `wait_for_load`：是否等待页面加载完成，默认true

#### browser_click
功能：点击页面上的元素
参数：
- `selector`：元素 CSS 选择器（必填）
- `wait_for_navigation`：是否等待点击后的页面跳转，默认false

#### browser_fill
功能：填写表单字段
参数：
- `selector`：输入框 CSS 选择器（必填）
- `value`：要填写的内容（必填）

示例：
```
> 帮我打开百度，搜索 Hermes Agent，然后提取搜索结果
```

---

### 6. Code 代码开发工具集
提供代码执行、评审、测试生成能力：

#### code_execution_tool
功能：在隔离沙箱中执行代码
参数：
- `language`：代码语言，支持 python/javascript/java/go 等（必填）
- `code`：要执行的代码内容（必填）
- `timeout`：执行超时时间（秒），默认10
示例：
```
> 帮我运行这段快速排序的Python代码，看看输出是否正确
```

#### code_review
功能：评审代码质量
参数：
- `code`：要评审的代码内容（必填）
- `language`：代码语言
- `focus`：评审重点，可选：performance/security/readability/all

---

### 7. MCP 工具集
支持 Model Context Protocol 协议，对接第三方 MCP 服务：

#### mcp_list_servers
功能：列出已配置的 MCP 服务器
#### mcp_tool_discovery
功能：发现指定 MCP 服务器提供的工具
#### mcp_call
功能：调用 MCP 服务器的工具
参数：
- `server`：MCP 服务器名称（必填）
- `tool`：工具名称（必填）
- `params`：工具参数（必填）

---

### 8. Delegate 子代理工具集
支持多代理协作，分派任务给子代理执行：

#### delegate_to_agent
功能：分派任务给指定类型的子代理
参数：
- `task`：任务描述（必填）
- `agent_type`：子代理类型，可选：code/research/write/design
- `return_result`：是否直接返回结果，默认true

#### subagent_spawn
功能：创建新的独立子代理实例
参数：
- `name`：子代理名称（必填）
- `system_prompt`：子代理的系统提示词（必填）

---

### 9. Media 多媒体工具集
提供音视频、图像处理能力：

#### vision_tool
功能：分析图片内容
参数：
- `image_url`：图片 URL 或本地路径（必填）
- `prompt`：分析指令（必填）

#### transcription_tool
功能：音频转文字
参数：
- `audio_path`：音频文件路径（必填）
- `language`：音频语言，可选

#### tts_tool
功能：文字转语音
参数：
- `text`：要转换的文字（必填）
- `voice`：音色名称，可选

#### image_generation
功能：生成图片
参数：
- `prompt`：图片生成提示词（必填）
- `size`：图片尺寸，默认 1024x1024

---

### 10. IoT 物联网工具集
对接智能家居和物联网设备：

#### homeassistant_tool
功能：控制 HomeAssistant 设备
参数：
- `action`：操作类型：get_state/call_service
- `entity_id`：设备实体 ID
- `service`：要调用的服务（call_service 时必填）
- `params`：服务参数

#### device_control
功能：控制其他物联网设备

---

### 11. System 系统管理工具集
提供 Agent 系统管理能力：
- checkpoint_manager：管理执行检查点
- budget_config：配置 token 使用预算
- credential_files：管理凭据文件

## 工具权限控制

### 启用/禁用工具集
在 `config.yaml` 中配置启用的工具集：
```yaml
tools:
  enabled_toolsets:
    - core
    - file
    - terminal
    - web
  disabled_tools:
    - browser_navigate  # 禁用特定工具
```

### 工具执行审批
高危工具（如 terminal_exec、file_write）默认需要用户审批才能执行，可以通过配置 `HERMES_EXEC_ASK=0` 关闭审批（不推荐生产环境使用）。

### 路径安全限制
文件操作工具默认限制只能访问当前工作目录及子目录，防止越权访问系统文件，可以通过配置 `HERMES_FILE_ALLOW_PATHS` 添加允许访问的路径。

---

## ✍️ 练习
1. 尝试让Agent读取你当前目录下的某个文件，验证file_read工具的调用
2. 让Agent执行一个简单的终端命令（比如 `ls -la`），体验工具审批流程
3. 配置禁用browser工具集，验证浏览器相关工具无法被调用
4. 尝试使用web_search工具搜索最近的AI新闻，体验网络搜索能力

---

## ❓ 常见问题
### Q：Agent不会调用我想要的工具怎么办？
A：你可以在提问时明确指定要使用的工具，比如"使用web_search搜索最近的AI新闻"，或者补充更多上下文让Agent正确判断需要的工具。

### Q：工具调用失败提示没有权限怎么办？
A：检查对应的工具集是否已经启用，或者是否在disabled_tools列表中被禁用，同时确认是否授予了对应的系统权限（比如文件读写权限、网络访问权限）。

### Q：可以自定义开发新工具吗？
A：可以，参考开发指南中的"添加新工具"章节，只需要几十行代码就可以开发自己的自定义工具。

### Q：工具调用的结果会占用Token吗？
A：会，工具返回的结果会作为上下文的一部分发送给模型，所以默认会自动对长结果进行压缩，减少Token消耗。

---

## 📖 导航
- 上一篇：[网关使用指南](./2-gateway-guide.md)
- 下一篇：[技能使用指南](./4-skills-guide.md)
- [返回目录](../SUMMARY.md)
