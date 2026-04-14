# 5. 多实例配置（Profile 系统）

Hermes Agent 支持多 Profile（多实例）功能，每个 Profile 是完全独立的运行环境，拥有自己的配置、凭据、会话数据、技能和工具设置，满足多场景、多环境、多用户的使用需求。

## 概述

Profile 系统的核心设计目标是实现完全的环境隔离，不同 Profile 之间互不干扰，适合以下场景：
- 区分开发/测试/生产环境
- 不同用户使用独立的配置和数据
- 同时使用不同的模型提供商和 API 密钥
- 隔离不同的工作场景（个人使用/工作使用/项目专用）

## 底层实现原理

### 核心隔离机制
1. **独立目录结构**：每个 Profile 对应独立的 `HERMES_HOME` 目录：
   - 默认 Profile：`~/.hermes/`
   - 自定义 Profile：`~/.hermes/profiles/[profile名称]/`
   
2. **启动注入逻辑**：在所有模块加载前，`_apply_profile_override()` 函数优先解析命令行的 `--profile/-p` 参数，或者读取 `~/.hermes/active_profile` 文件的默认 Profile，设置 `HERMES_HOME` 环境变量为对应 Profile 的目录。

3. **统一路径访问**：所有代码中涉及 Hermes 目录的操作都通过 `get_hermes_home()` 函数获取路径，确保所有配置、数据、凭据都指向当前 Profile 的目录，实现完全隔离。

## 常用 Profile 命令

### 查看所有 Profile
```bash
hermes profile list
```
输出示例：
```
  default  (active)
  work
  personal
  project-x
```

### 创建新的 Profile
```bash
hermes profile create [profile名称]
```
示例：
```bash
hermes profile create work
```
创建成功后会自动生成 `~/.hermes/profiles/work/` 目录，包含独立的配置文件模板。

### 使用指定 Profile 运行
```bash
hermes --profile [profile名称] [命令]
# 短参数版本
hermes -p [profile名称] [命令]
```
示例：
```bash
# 用 work Profile 启动 CLI
hermes -p work

# 用 personal Profile 运行配置向导
hermes -p personal setup

# 用 project-x Profile 启动网关
hermes -p project-x gateway start
```

### 切换默认 Profile
```bash
hermes profile use [profile名称]
```
示例：
```bash
hermes profile use work
```
设置后，后续执行 `hermes` 命令时如果不指定 `-p` 参数，会默认使用 `work` Profile。

### 查看当前激活的 Profile
```bash
hermes profile show
```
输出当前激活的 Profile 名称和目录路径。

### 删除 Profile
```bash
hermes profile delete [profile名称]
```
示例：
```bash
hermes profile delete old-project
```
删除后会删除该 Profile 对应的所有目录和数据，无法恢复，请谨慎操作。

## 隔离边界说明

每个 Profile 之间在以下维度完全隔离：

| 隔离维度 | 说明 |
|----------|------|
| **配置隔离** | 每个 Profile 有独立的 `config.yaml` 和 `.env` 文件，模型配置、工具启用、权限设置、皮肤主题完全独立，可以为不同 Profile 配置不同的默认模型、启用不同的工具集。 |
| **数据隔离** | 独立的会话历史数据库、长期记忆存储、技能安装目录、缓存文件，不同 Profile 的对话历史、记忆、技能不会互相访问。 |
| **凭据隔离** | 独立的 API 密钥、平台登录凭据、OAuth 授权信息，不会跨 Profile 泄露，适合不同团队使用不同的服务账号。 |
| **运行隔离** | 不同 Profile 的 Agent 实例完全独立，不会共享上下文、工具状态、后台进程，支持同时运行多个 Profile 的网关服务。 |
| **平台隔离** | 每个 Profile 可以独立配置不同的消息平台、权限规则、接入凭据，支持在同一台服务器上同时部署开发、测试、生产三套网关环境。 |
| **资源隔离** | 每个 Profile 可以独立设置模型配额、工具调用限制、执行资源限制，避免不同场景的资源使用互相影响。 |

## 使用场景示例

### 场景1：区分工作和个人使用
```bash
# 创建工作和个人两个 Profile
hermes profile create work
hermes profile create personal

# 配置工作 Profile：使用公司的 API 密钥，启用企业相关技能
hermes -p work setup

# 配置个人 Profile：使用个人 API 密钥，启用生活相关技能
hermes -p personal setup

# 日常工作使用
hermes -p work

# 个人使用
hermes -p personal
```

### 场景2：多环境部署网关
```bash
# 创建开发、测试、生产三个 Profile
hermes profile create dev
hermes profile create test
hermes profile create prod

# 分别配置三个环境的平台凭据、权限规则
hermes -p dev setup
hermes -p test setup
hermes -p prod setup

# 分别启动三个环境的网关服务
hermes -p dev gateway install
hermes -p test gateway install
hermes -p prod gateway install
```
三个环境的网关完全独立，互不影响。

### 场景3：项目专用环境
为每个重要项目创建独立的 Profile，保存项目相关的会话历史、记忆、配置：
```bash
hermes profile create project-ai-assistant
hermes -p project-ai-assistant
```
所有和该项目相关的对话、文档、代码都会保存在独立的目录下，不会和其他项目混淆。

## 常见问题

**Q：Profile 数据存储在哪里？**
A：默认 Profile 存储在 `~/.hermes/`，自定义 Profile 存储在 `~/.hermes/profiles/[profile名称]/`。

**Q：可以在不同 Profile 之间共享数据吗？**
A：默认完全隔离，不支持直接共享，如果需要共享可以手动复制对应目录下的文件。

**Q：删除 Profile 会影响其他 Profile 吗？**
A：不会，每个 Profile 是独立的目录，删除操作只会删除对应 Profile 的数据。

**Q：最多支持多少个 Profile？**
A：没有数量限制，可以根据需要创建任意多个 Profile。

**Q：如何备份 Profile？**
A：直接备份对应的 Profile 目录即可，恢复时将备份的目录放到 `~/.hermes/profiles/` 下即可使用。
