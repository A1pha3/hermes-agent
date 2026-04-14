# 4. 基础配置

## 配置文件位置

主配置文件位于：
```
~/.hermes/config.yaml
```

环境变量（API 密钥等敏感信息）位于：
```
~/.hermes/.env
```

## 核心配置项

### 模型配置
```yaml
model:
  default: anthropic/claude-opus-4.6
  max_tokens: 4096
  temperature: 0.7
```
- `default`：默认使用的模型
- `max_tokens`：单次响应最大 Token 数
- `temperature`：模型温度，0 到 1 之间，值越高输出越随机

### 显示配置
```yaml
display:
  skin: default
  show_tool_progress: true
  background_process_notifications: all
  prompt_symbol: ">"
```
- `skin`：使用的主题皮肤
- `show_tool_progress`：是否显示工具调用进度
- `background_process_notifications`：后台进程通知级别：all/result/error/off
- `prompt_symbol`：自定义提示符

### 工具配置
```yaml
tools:
  enabled_toolsets:
    - core
    - file
    - web
    - terminal
  disabled_tools: []
```
- `enabled_toolsets`：启用的工具集列表
- `disabled_tools`：禁用的特定工具列表

### 技能配置
```yaml
skills:
  enabled_skills:
    - alpha-loop
    - cn-doc-writer
  auto_update_skills: true
```
- `enabled_skills`：启用的技能列表
- `auto_update_skills`：是否自动更新技能

### 会话配置
```yaml
session:
  save_history: true
  max_history_length: 100
  auto_compress_context: true
```
- `save_history`：是否保存会话历史
- `max_history_length`：最大历史消息数
- `auto_compress_context`：是否自动压缩上下文

## 环境变量配置

在 `~/.hermes/.env` 中配置 API 密钥等敏感信息：
```bash
# Anthropic
ANTHROPIC_API_KEY=your_anthropic_api_key

# OpenAI
OPENAI_API_KEY=your_openai_api_key

# Browserbase (浏览器自动化)
BROWSERBASE_API_KEY=your_browserbase_api_key

# Firecrawl (网页抓取)
FIRECRAWL_API_KEY=your_firecrawl_api_key
```

## 修改配置的方式

### 方式一：直接编辑配置文件
直接编辑 `~/.hermes/config.yaml` 和 `~/.hermes/.env` 文件，修改后重启 Hermes 生效。

### 方式二：使用配置命令
在 CLI 中使用 `/set` 命令修改配置：
```
/set display.skin ares
```
修改会自动保存到配置文件，立即生效。

### 方式三：重新运行配置向导
```bash
hermes setup
```
重新运行交互式配置向导，重新配置所有选项。

## 多配置集（Profile）配置

Hermes 支持多配置集隔离配置，使用 `-p` 参数指定 Profile：
```bash
hermes -p work
```
每个 Profile 有独立的配置、密钥、会话存储，位于 `~/.hermes/profiles/work/` 目录。

查看所有 Profile：
```bash
hermes profile list
```

创建新 Profile：
```bash
hermes profile create personal
```

删除 Profile：
```bash
hermes profile delete old
```
