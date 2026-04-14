# 2. 核心模块分析

Hermes Agent 由多个职责明确的核心模块组成，每个模块独立实现，通过定义良好的接口交互。

## 2.1 Agent 模块（`run_agent.py` AIAgent 类）

### 职责
Agent 模块是整个系统的核心，负责：
- 对话核心循环的实现
- 工具调用的调度和协调
- 上下文生命周期的管理
- 多模型/多 Provider 的适配
- 迭代预算和 Token 用量控制

### 内部结构
```
AIAgent 类
├── 配置属性
│   ├── model_config：模型相关配置（名称、温度、最大Token等）
│   ├── tool_config：工具相关配置（启用的工具集、禁用的工具列表等）
│   ├── session_config：会话相关配置（最大迭代次数、自动压缩开关等）
│   └── platform_config：平台相关配置（平台标识、权限规则等）
├── 状态属性
│   ├── session_id：当前会话ID
│   ├── token_usage：当前会话的Token用量统计（输入/输出/缓存命中）
│   ├── iteration_count：当前会话的迭代次数
│   ├── compressor：上下文压缩器实例
│   └── cache_enabled：提示词缓存开关
└── 核心方法
    ├── run_conversation()：主对话循环，处理完整的对话流程
    ├── chat()：简化的同步API，直接返回对话结果
    ├── switch_model()：运行时动态切换模型或Provider
    └── _handle_tool_calls()：工具调用处理，判断并行执行逻辑
```

### 交互方式
- 依赖 `model_tools` 模块处理工具的调度和执行
- 依赖 `context_compressor` 模块处理上下文压缩
- 依赖 `prompt_caching` 模块处理提示词缓存
- 调用 `SessionDB` 存储层持久化会话数据
- 通过回调函数和入口层交互，返回进度和结果

## 2.2 CLI 模块（`cli.py`、`hermes_cli/` 目录）

### 职责
CLI 模块提供交互式终端使用体验，负责：
- 解析命令行参数和选项
- 实现交互式终端界面
- 管理用户配置和密钥
- 实现 Slash 命令系统
- 提供皮肤和主题自定义能力

### 内部结构
```
CLI 模块
├── HermesCLI 类：CLI 主控制器，管理整个终端生命周期
├── Slash 命令系统
│   ├── COMMAND_REGISTRY：统一的命令注册中心，所有命令在此注册
│   ├── CommandDef：命令定义结构，包含名称、描述、别名、参数、处理函数
│   ├── resolve_command()：命令解析和别名匹配
│   └── SlashCommandCompleter：自动补全实现
├── 皮肤引擎（`skin_engine.py`）
│   ├── SkinConfig：皮肤配置结构
│   ├── _BUILTIN_SKINS：内置皮肤定义
│   ├── load_skin()：加载皮肤配置
│   └── get_active_skin()：获取当前激活的皮肤
└── 配置系统
    ├── DEFAULT_CONFIG：默认配置模板
    ├── load_config()：加载用户配置
    ├── save_config()：保存配置修改
    └── load_env()：加载环境变量和密钥
```

### 交互方式
- 实例化 `AIAgent` 类处理用户对话请求
- 调用工具模块执行终端相关命令
- 读写配置文件和环境变量
- 调用皮肤引擎渲染终端界面

## 2.3 工具模块（`tools/` 目录、`model_tools.py`、`registry.py`）

### 职责
工具模块负责所有工具能力的实现、注册、调度，是 Agent 能力扩展的核心：
- 所有内置工具的实现
- 工具的自动注册和发现
- 工具集的动态开关和可用性检查
- 工具执行的权限控制和危险操作检测
- 工具调用的调度和执行

### 内部结构
```
工具模块
├── 注册中心（`registry.py`）
│   ├── Registry 单例类：管理所有工具的元数据
│   ├── register()：工具注册方法
│   ├── get_tool()：根据名称获取工具
│   └── list_tools()：获取所有可用工具列表
├── 工具编排层（`model_tools.py`）
│   ├── _discover_tools()：自动发现并导入所有工具
│   ├── get_tool_definitions()：获取可用工具的Schema列表
│   ├── handle_function_call()：调度执行工具调用
│   └── _is_parallel_safe()：判断工具是否可以并行执行
└── 工具实现层（`tools/*.py`）
    ├── 每个工具独立实现为一个文件
    ├── 文件导入时自动注册到注册中心
    └── 每个工具包含实现函数、schema、可用性检查函数
```

### 交互方式
- 工具在 import 时自动注册到 `Registry` 单例
- 核心层通过 `model_tools.get_tool_definitions()` 获取可用工具的 Schema，传给 LLM
- 核心层通过 `model_tools.handle_function_call()` 调度执行工具调用
- 工具执行结果返回给核心层，追加到上下文

## 2.4 技能模块（`agent/skill_commands.py`、`hermes_cli/skills_config.py`）

### 职责
技能模块负责自定义技能的管理、加载和执行：
- 技能的自动扫描和加载
- 技能的启用/禁用管理
- 技能提示词的注入
- 技能命令的解析和处理

### 内部结构
```
技能模块
├── 技能存储：`~/.hermes/skills/` 目录下的Markdown文件
├── 技能加载器
│   ├── scan_skills()：扫描技能目录，加载所有技能
│   ├── parse_skill_file()：解析SKILL.md文件，提取元数据和提示词
│   └── register_skill_command()：注册技能对应的Slash命令
└── 技能执行器
    ├── inject_skill_prompt()：将技能提示词注入到系统提示
    └── handle_skill_command()：处理技能命令触发
```

### 交互方式
- 技能在系统提示词组装时自动注入，不需要修改核心代码
- 技能执行走普通的对话流程，不需要特殊处理
- 技能命令通过Slash命令系统注册和处理

## 2.5 网关模块（`gateway/` 目录）

### 职责
网关模块负责对接各类消息平台，将 Hermes Agent 能力暴露给 IM 用户：
- 对接各类消息平台（Telegram、Discord、Slack、企业微信、微信等）
- 多用户会话隔离和管理
- 后台任务和完成通知管理
- 平台相关的权限控制

### 内部结构
```
网关模块
├── 平台适配器（`gateway/platforms/` 目录）
│   ├── 每个平台独立实现适配器
│   ├── 统一的接口规范：start()、stop()、send_message()、on_message()
│   └── 平台特定的消息格式转换、权限校验
├── 会话管理器
│   ├── SessionStore 类：管理用户/群组的会话状态
│   ├── 按用户ID/群组ID隔离会话
│   └── 会话生命周期管理（创建、过期、销毁）
└── 后台任务系统
    ├── JobManager 类：管理异步执行的后台任务
    ├── 任务状态跟踪
    └── 任务完成后的通知推送
```

### 交互方式
- 每个用户消息创建独立的 `AIAgent` 实例处理
- 会话状态存储到 `SessionDB` 存储层
- 工具执行结果通过平台适配器返回给用户
- 后台任务完成后主动推送通知给用户

## 2.6 存储模块（`hermes_state.py` SessionDB 类）

### 职责
存储模块负责所有数据的持久化存储：
- 会话历史的持久化存储
- 全文搜索支持
- 会话历史回溯和管理
- 配置数据的存储

### 内部结构
```
存储模块
├── SessionDB 类：SQLite数据库封装
│   ├── init_db()：初始化数据库表结构
│   ├── create_session()：创建新会话
│   ├── add_message()：添加消息到会话
│   ├── update_session_stats()：更新会话统计信息
│   ├── search_messages()：全文搜索消息
│   └── get_session_history()：获取会话历史
├── 数据库表结构
│   ├── sessions表：会话元数据
│   ├── messages表：会话消息数据
│   └── messages_fts表：FTS5全文搜索虚拟表
└── 迁移系统
    ├── _get_current_schema_version()：获取当前schema版本
    ├── _run_migrations()：执行增量迁移
    └── 向后兼容所有历史版本
```

### 交互方式
- 核心层在每个对话回合结束后异步写入会话数据
- 全文搜索功能通过 `/search` Slash 命令调用
- 会话回溯功能通过 `/session` Slash 命令调用
- 配置数据通过配置系统读写
