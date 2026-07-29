# 1. 配置项完整参考

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 了解所有可用的配置项
> - 根据需求调整配置
> - 优化配置提升性能和体验

---

## 📖 导航
- 上一篇：[企业级部署最佳实践](../advanced/5-enterprise-best-practices.md)
- 下一篇：[API 参考](./2-api-reference.md)
- [返回目录](../SUMMARY.md)

---

## 配置文件说明

配置文件默认位于 `~/.hermes/config.yaml`，支持标准YAML格式，修改配置后**需要重启Hermes Agent**生效。你也可以通过 `hermes --config /path/to/custom.yaml` 命令指定自定义配置文件路径。

### 配置优先级（从高到低）
1. 命令行参数（比如 `hermes --model anthropic/claude-3-opus`）
2. 环境变量（格式：`HERMES_` + 配置路径大写，下划线分隔，比如 `HERMES_MODEL_DEFAULT` 对应 `model.default`）
3. 自定义配置文件（通过 `--config` 指定）
4. 默认配置文件（`~/.hermes/config.yaml`）
5. 系统内置默认值

### 配置验证
修改配置后可以先运行 `hermes config validate` 命令验证配置格式是否正确，避免启动失败。

---

## 核心配置项

### model（模型配置）
```yaml
model:
  default: "anthropic/claude-3.5-sonnet"  # 默认使用的模型，默认值：anthropic/claude-3-sonnet
  fallback_models: ["openai/gpt-4o", "ollama/llama3"]  # 主模型不可用时自动切换的降级模型列表，默认空
  temperature: 0.7  # 默认温度参数，0~1之间，值越高随机性越强，默认值：0.7
  max_tokens: 4096  # 最大生成Token数，根据模型上下文窗口调整，默认值：4096
  timeout: 60  # API请求超时时间（秒），默认值：60
  proxy: ""  # 模型API代理地址，比如 http://127.0.0.1:7890，默认空
  max_retries: 3 # API请求失败重试次数，默认值：3
  retry_delay: 2 # 重试等待时间（秒），指数退避基础值，默认值：2
```
> 配置生效：修改后重启生效

### provider（模型提供商配置）
```yaml
provider:
  anthropic:
    api_key: "${ANTHROPIC_API_KEY}"  # API密钥，支持环境变量注入
    api_base: "https://api.anthropic.com"  # API地址，可配置反向代理地址
    version: "2023-06-01"  # API版本，无需修改
  openai:
    api_key: "${OPENAI_API_KEY}"
    api_base: "https://api.openai.com/v1"
    organization: "" # OpenAI组织ID，可选
  ollama:
    api_base: "http://localhost:11434" # 本地Ollama服务地址
  qwen: # 通义千问
    api_key: "${QWEN_API_KEY}"
    api_base: "https://dashscope.aliyuncs.com/compatible-mode/v1"
  deepseek: # 深度求索
    api_key: "${DEEPSEEK_API_KEY}"
    api_base: "https://api.deepseek.com/v1"
```
> 配置生效：修改后重启生效，API密钥也可以配置在 `~/.hermes/.env` 文件中

### model_routing（模型路由配置）
根据任务类型自动选择最优模型，平衡成本和效果
```yaml
model_routing:
  enabled: true # 是否启用自动路由，默认值：false
  default_model: "anthropic/claude-3.5-sonnet" # 默认使用的模型
  rules:
    # 代码相关任务使用能力更强的模型
    - task_type: "code"
      model: "anthropic/claude-3-opus"
      keywords: ["代码", "编程", "debug", "开发"] # 触发关键词
    # 简单聊天使用低成本小模型
    - task_type: "chat"
      model: "anthropic/claude-3-haiku"
      keywords: ["聊天", "问答", "查询", "翻译"]
    # 图像处理使用多模态模型
    - task_type: "vision"
      model: "anthropic/claude-3.5-sonnet"
      keywords: ["图片", "图像", "截图", "OCR"]
  fallback_models: ["openai/gpt-4o", "ollama/llama3:70b"] # 主模型失败时降级列表
```
> 配置生效：修改后重启生效

### display（显示配置）
```yaml
display:
  skin: "default"  # 皮肤主题：default/ares/mono/slate，默认值：default
  lang: "zh-CN"  # 界面语言：zh-CN/en-US，默认跟随系统语言
  show_spinner: true  # 是否显示加载动画，默认值：true
  show_tool_calls: true  # 是否显示工具调用过程，false会隐藏工具调用细节，默认值：true
  show_reasoning: false  # 是否显示模型思考过程（思维链），默认值：false
  response_box: true  # 是否用框包裹返回结果，默认值：true
  prompt_symbol: ">"  # 输入提示符，支持自定义emoji比如 "🦊"，默认值：">"
  show_welcome_banner: true # 是否显示启动欢迎横幅，默认值：true
```
> 配置生效：修改后重启生效，部分显示配置CLI端可以通过 `/set` 命令动态修改无需重启

### log（日志配置）
```yaml
log:
  level: "info" # 日志级别：debug/info/warn/error，默认值：info
  path: "~/.hermes/logs/hermes.log" # 日志文件路径，默认值：~/.hermes/logs/hermes.log
  max_size: 100MB # 单个日志文件最大大小，默认值：100MB
  max_backups: 7 # 保留的日志备份数，默认值：7
  max_age: 30 # 日志保留天数，默认值：30
  compress: true # 是否压缩旧日志，默认值：true
  audit_log_enabled: true # 是否启用审计日志，默认值：true
  audit_log_path: "~/.hermes/logs/audit.log" # 审计日志路径
```
> 配置生效：修改后重启生效

### tools（工具配置）
```yaml
tools:
  enabled_toolsets: ["core", "file", "web", "code", "terminal"]  # 启用的工具集，默认值：["core", "file", "web", "code"]
  disabled_tools: []  # 禁用的特定工具列表，比如 ["terminal_exec"]，默认空
  auto_parallel: true # 自动并行执行无副作用的工具，提升速度，默认值：true
  cache_enabled: true # 工具结果缓存，相同参数重复调用直接返回缓存，默认值：true
  cache_ttl: 300 # 缓存有效期（秒），默认值：300
  approval:
    enabled: true  # 是否启用危险操作审批，默认值：true
    dangerous_commands: ["rm -rf /", "mkfs", "dd if=/dev", "format"]  # 高危命令列表，匹配到会触发审批
    auto_allow_safe: true # 自动允许安全的只读命令，不需要审批，默认值：true
  file:
    allowed_paths: ["~", "/tmp"]  # 允许操作的文件路径，支持通配符
    blocked_paths: ["/etc", "/root", "/sys", "/proc"]  # 禁止操作的敏感路径
    allow_write: true # 是否允许文件写入操作，设为false则只读，默认值：true
  web:
    allowed_domains: ["*"]  # 允许访问的域名，*表示全部允许
    blocked_domains: ["internal.example.com", "*.corp.com"]  # 禁止访问的内部域名
    timeout: 10 # 请求超时时间（秒），默认值：10
  terminal:
    allow_root: false # 是否允许以root身份执行命令，默认值：false
    allowed_commands: ["ls", "cat", "cd", "pwd", "python"] # 允许的命令列表，空表示全部允许
    blocked_commands: ["rm", "dd", "mkfs"] # 禁止的命令列表
  mcp: # MCP（Model Context Protocol）工具配置
    enabled: true # 是否启用MCP工具，默认值：true
    servers: # MCP服务器列表
      - name: "local-tools"
        command: "node"
        args: ["~/.mcp/servers/local-tools/dist/index.js"]
```
> 配置生效：修改后重启生效

### skills（技能配置）
```yaml
skills:
  enabled_skills: ["*"]  # 启用的技能列表，*表示全部启用
  disabled_skills: []  # 禁用的技能列表
  marketplace_url: "https://marketplace.hermes-agent.dev"  # 技能市场地址
  auto_update: true  # 是否自动更新技能
```

### session（会话配置）
```yaml
session:
  storage: "sqlite"  # 存储引擎：sqlite/postgresql
  sqlite_path: "~/.hermes/sessions.db"  # SQLite数据库路径
  postgresql:
    host: "localhost"
    port: 5432
    database: "hermes"
    user: "hermes"
    password: "${DB_PASSWORD}"
  auto_save: true  # 是否自动保存会话
  retention_days: 90  # 会话保留天数，0表示永久保留
  full_text_search: true  # 是否启用全文搜索
```

### gateway（网关配置）
```yaml
gateway:
  enabled: false  # 是否启用网关服务
  host: "0.0.0.0"  # 监听地址
  port: 8080  # 监听端口
  enabled_platforms: []  # 启用的平台列表
  telegram:
    bot_token: "${TELEGRAM_BOT_TOKEN}"
    allowed_users: ["*"]
  discord:
    bot_token: "${DISCORD_BOT_TOKEN}"
    allowed_servers: ["*"]
  webhook_secret: "${WEBHOOK_SECRET}"  # Webhook验证密钥
```

### context（上下文配置）
```yaml
context:
  compression:
    enabled: true  # 是否启用上下文压缩
    threshold: 2000  # 超过多少Token开始压缩
    ratio: 0.5  # 压缩比例
    preserve_keywords: []  # 压缩时保留的关键字
  caching:
    enabled: true  # 是否启用提示词缓存
    ttl: 86400  # 缓存过期时间（秒）
    provider: "anthropic"  # 缓存提供商
```

### auth（认证配置）
```yaml
auth:
  enabled: false  # 是否启用认证
  type: "oauth2"  # 认证类型：oauth2/ldap/basic
  oauth2:
    client_id: "${OAUTH_CLIENT_ID}"
    client_secret: "${OAUTH_CLIENT_SECRET}"
    authorize_url: "https://idp.example.com/oauth2/authorize"
    token_url: "https://idp.example.com/oauth2/token"
    userinfo_url: "https://idp.example.com/oauth2/userinfo"
    scopes: ["openid", "profile", "email"]
  rbac:
    enabled: false  # 是否启用RBAC权限控制
    roles: {}  # 角色定义
    users: {}  # 用户角色映射
```

### audit（审计配置）
```yaml
audit:
  enabled: false  # 是否启用审计日志
  log_path: "~/.hermes/logs/audit.log"  # 审计日志路径
  retention_days: 90  # 日志保留天数
  log_actions: ["tool_call", "skill_use", "config_change", "login"]  # 需要记录的操作类型
```

### notifications（通知配置）
```yaml
notifications:
  enabled: false  # 是否启用通知
  webhooks: []  # Webhook通知列表
  email:
    enabled: false
    smtp_host: "smtp.example.com"
    smtp_port: 587
    smtp_user: "${SMTP_USER}"
    smtp_password: "${SMTP_PASSWORD}"
    to: ["admin@example.com"]
```

---

## 配置示例

### 个人开发者最小配置
```yaml
model:
  default: "anthropic/claude-3-sonnet"
provider:
  anthropic:
    api_key: "sk-xxx"
```

### 本地Ollama私有化部署配置
完全本地化运行，不需要联网
```yaml
model:
  default: "ollama/qwen2:72b"
provider:
  ollama:
    api_base: "http://localhost:11434"
tools:
  enabled_toolsets: ["core", "file", "code", "terminal"] # 只启用不需要联网的工具
context:
  caching:
    enabled: true
    provider: "local" # 使用本地缓存
```

### 飞书网关配置
对接企业飞书机器人
```yaml
gateway:
  enabled: true
  port: 8080
  enabled_platforms: ["feishu"]
  feishu:
    app_id: "${FEISHU_APP_ID}"
    app_secret: "${FEISHU_APP_SECRET}"
    verification_token: "${FEISHU_VERIFICATION_TOKEN}"
    encrypt_key: "${FEISHU_ENCRYPT_KEY}"
    allowed_users: ["*@company.com"] # 只允许企业内部用户使用
```

### LDAP认证配置
对接企业内部LDAP账号系统
```yaml
auth:
  enabled: true
  type: "ldap"
  ldap:
    host: "ldap.company.com"
    port: 389
    use_ssl: true
    bind_dn: "cn=admin,dc=company,dc=com"
    bind_password: "${LDAP_BIND_PASSWORD}"
    search_base: "ou=users,dc=company,dc=com"
    search_filter: "(uid=%s)"
    attributes:
      user_id: "uid"
      email: "mail"
      name: "cn"
      role: "title"
  rbac:
    enabled: true
    role_mapping:
      "技术部开发": "developer"
      "运维工程师": "admin"
      "*": "user"
```

### 企业级部署配置
```yaml
model:
  default: "anthropic/claude-3.5-sonnet"
  fallback_models: ["openai/gpt-4o", "ollama/qwen2:72b"]
provider:
  anthropic:
    api_key: "${ANTHROPIC_API_KEY}"
    proxy: "http://proxy.company.com:7890"
  openai:
    api_key: "${OPENAI_API_KEY}"
session:
  storage: "postgresql"
  postgresql:
    host: "postgres.hermes.svc.cluster.local"
    port: 5432
    database: "hermes"
    user: "hermes"
    password: "${DB_PASSWORD}"
    ssl_mode: "require"
gateway:
  enabled: true
  port: 8080
  enabled_platforms: ["feishu", "web"]
  feishu:
    app_id: "${FEISHU_APP_ID}"
    app_secret: "${FEISHU_APP_SECRET}"
auth:
  enabled: true
  type: "oauth2"
  oauth2:
    client_id: "${OAUTH_CLIENT_ID}"
    client_secret: "${OAUTH_CLIENT_SECRET}"
    authorize_url: "https://sso.company.com/oauth2/authorize"
    token_url: "https://sso.company.com/oauth2/token"
    userinfo_url: "https://sso.company.com/oauth2/userinfo"
    allowed_domains: ["company.com"]
  rbac:
    enabled: true
audit:
  enabled: true
  log_path: "/var/log/hermes/audit.log"
  retention_days: 180 # 审计日志保留180天，满足等保要求
log:
  level: "warn"
  path: "/var/log/hermes/hermes.log"
```

---

## 环境变量列表

所有带 `${变量名}` 的配置项都可以通过环境变量设置，也可以写在 `~/.hermes/.env` 文件中自动加载。常用环境变量：

| 环境变量名 | 说明 |
|------------|------|
| `HERMES_HOME` | Hermes数据目录，默认值：`~/.hermes` |
| `ANTHROPIC_API_KEY` | Anthropic Claude系列模型API密钥 |
| `OPENAI_API_KEY` | OpenAI GPT系列模型API密钥 |
| `QWEN_API_KEY` | 阿里云通义千问API密钥 |
| `DEEPSEEK_API_KEY` | 深度求索DeepSeek API密钥 |
| `TELEGRAM_BOT_TOKEN` | Telegram机器人令牌 |
| `DISCORD_BOT_TOKEN` | Discord机器人令牌 |
| `FEISHU_APP_ID` / `FEISHU_APP_SECRET` | 飞书机器人凭证 |
| `WECOM_APP_ID` / `WECOM_SECRET` | 企业微信机器人凭证 |
| `DB_PASSWORD` | PostgreSQL数据库密码 |
| `LDAP_BIND_PASSWORD` | LDAP绑定密码 |
| `OAUTH_CLIENT_SECRET` | Oauth2认证密钥 |
| `WEBHOOK_SECRET` | Webhook验证密钥 |
| `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` | 代理服务器地址 |
| `NO_PROXY` | 不需要走代理的域名列表 |

---

## ✍️ 练习
1. 根据你的使用场景，调整配置项优化使用体验
2. 使用环境变量配置敏感信息，避免明文写在配置文件中

---

## ❓ 常见问题
### Q：修改配置后需要重启吗？
A：大部分配置修改后需要重启才能生效，部分CLI显示类配置可以通过 `/set` 命令动态修改无需重启，比如 `display.prompt_symbol`。

### Q：配置文件中的 `${变量名}` 是什么意思？
A：表示从环境变量或者 `~/.hermes/.env` 文件读取对应的值，适合存储API密钥、密码等敏感信息，避免明文写在配置文件中导致泄露。

### Q：可以使用多个配置文件吗？
A：可以，使用 `hermes --config /path/to/config.yaml` 指定自定义配置文件路径，自定义配置会和默认配置合并。

### Q：配置文件语法错误启动失败怎么办？
A：运行 `hermes config validate` 命令可以检查配置文件语法错误，会给出具体的错误位置和原因。

### Q：如何恢复默认配置？
A：删除 `~/.hermes/config.yaml` 文件，重启Hermes Agent会自动生成默认配置文件。

### Q：不同平台的配置可以分开吗？
A：可以，通过环境变量或者启动参数指定不同的配置文件，比如 `hermes --config ~/.hermes/config.work.yaml` 启动工作场景配置，`hermes --config ~/.hermes/config.personal.yaml` 启动个人场景配置。

### Q：配置文件中的路径支持相对路径吗？
A：支持相对路径，相对路径是相对于当前运行目录，建议使用绝对路径或者 `~` 开头的用户目录路径，避免不同运行目录导致路径错误。

---

## 📖 导航
- 上一篇：[企业级部署最佳实践](../advanced/5-enterprise-best-practices.md)
- 下一篇：[API 参考](./2-api-reference.md)
- [返回目录](../SUMMARY.md)