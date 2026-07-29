# 4. 集成第三方系统

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 掌握 Hermes Agent 集成第三方系统的三种方式和适用场景
> - 独立完成常见企业系统的集成（内部API、数据库、SSO、通知系统等）
> - 实现企业级的权限控制和操作审计，满足合规要求
> - 保障集成过程中的数据安全和系统稳定性

## 集成方式概述

Hermes Agent 采用开放式设计，支持三种方式集成第三方系统，可以根据场景复杂度选择合适的方式：

| 集成方式 | 适用场景 | 开发成本 | 优势 | 劣势 |
|----------|----------|----------|------|------|
| **工具集成** | 简单API调用、操作外部服务、查询数据等简单交互 | 低 | 开发快、无需修改核心代码、灵活度高 | 适合简单交互，复杂流程需要编写额外逻辑 |
| **技能集成** | 复杂工作流、多步骤交互、跨系统联动等场景 | 中 | 功能强大、可复用、无需修改核心代码、热加载 | 需要编写技能工作流逻辑 |
| **平台集成** | 统一身份认证、嵌入现有系统、深度整合企业架构等场景 | 高 | 体验一致、深度整合、符合企业IT规范 | 开发成本高、需要修改核心配置 |

> 集成设计原则：
> 1. **最小侵入**：优先选择工具/技能集成，尽量不修改核心代码
> 2. **安全优先**：所有集成都要做权限控制、数据脱敏、审计日志
> 3. **可监控**：集成的接口要有监控告警，出现问题及时发现
> 4. **可降级**：第三方系统故障时不影响核心服务正常运行

## 集成前期准备
在开始集成前需要做好以下准备：
1. 梳理需要集成的系统接口文档、权限要求、限流策略
2. 申请专用的服务账号，使用最小权限原则，避免使用个人账号
3. 准备测试环境，先在测试环境验证集成效果，再上生产
4. 配置网络策略，确保Hermes Agent服务器可以访问第三方系统
5. 准备环境变量存储敏感信息，不要硬编码到代码或配置文件

---

## 常见系统集成方案

### 1. 集成企业内部API
#### 适用场景
需要调用企业内部的业务API，比如查询订单、创建工单、查询用户信息、触发工作流等。

#### 实现步骤
1. 开发自定义工具，封装API调用逻辑，统一处理签名、认证、错误、重试
2. 配置API密钥、基础地址等敏感信息到环境变量
3. 配置工具Schema，让大模型理解工具的功能和参数
4. 测试工具调用，验证功能正常
5. 配置RBAC权限，控制哪些用户可以使用这个工具

#### 最佳实践示例代码
```python
import json
import os
import requests
from typing import Optional, Dict, Any
from tools.registry import registry
import logging

logger = logging.getLogger(__name__)

# 配置重试策略：最多重试3次，指数退避
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    retry=retry_if_exception_type((requests.exceptions.Timeout, requests.exceptions.ConnectionError))
)
def _send_request(method: str, url: str, headers: Dict, params: Optional[Dict] = None, body: Optional[Dict] = None):
    """发送HTTP请求，带重试机制"""
    response = requests.request(
        method=method,
        url=url,
        params=params,
        json=body,
        headers=headers,
        timeout=10
    )
    response.raise_for_status()
    return response.json()

def internal_api_call(
    endpoint: str, 
    method: str = "GET", 
    params: Optional[Dict] = None, 
    body: Optional[Dict] = None,
    task_id: Optional[str] = None
) -> str:
    """
    调用企业内部API，支持查询订单、创建工单等操作
    :param endpoint: API端点，比如 /api/v1/orders
    :param method: HTTP方法，支持GET/POST/PUT/DELETE，默认GET
    :param params: 查询参数，可选
    :param body: 请求体参数，可选
    :param task_id: 任务ID，用于日志跟踪
    """
    api_key = os.getenv("INTERNAL_API_KEY")
    base_url = os.getenv("INTERNAL_API_BASE", "https://api.example.com")
    
    if not api_key:
        return json.dumps({
            "success": False,
            "error": "内部API密钥未配置，请联系管理员"
        }, ensure_ascii=False)
    
    # 安全校验：禁止访问敏感端点
    sensitive_endpoints = ["/api/admin", "/api/user/password", "/api/delete"]
    for sensitive in sensitive_endpoints:
        if endpoint.startswith(sensitive):
            logger.warning(f"用户尝试访问敏感API端点：{endpoint}, 任务ID：{task_id}")
            return json.dumps({
                "success": False,
                "error": "无权限访问该API端点"
            }, ensure_ascii=False)
    
    headers = {
        "Authorization": f"Bearer {api_key}",
        "User-Agent": "Hermes-Agent/1.0",
        "X-Request-ID": task_id or ""
    }
    
    try:
        data = _send_request(method, f"{base_url}{endpoint}", headers, params, body)
        return json.dumps({
            "success": True,
            "data": data
        }, ensure_ascii=False)
    
    except requests.exceptions.HTTPError as e:
        logger.error(f"API调用失败，状态码：{e.response.status_code}, 错误：{e}")
        return json.dumps({
            "success": False,
            "error": f"API调用失败，状态码：{e.response.status_code}, 错误：{str(e)}"
        }, ensure_ascii=False)
    
    except Exception as e:
        logger.error(f"API调用异常：{e}", exc_info=True)
        return json.dumps({
            "success": False,
            "error": f"API调用异常：{str(e)}"
        }, ensure_ascii=False)

# 注册工具
registry.register(
    name="internal_api",
    toolset="enterprise",
    schema={
        "name": "internal_api",
        "description": "调用企业内部API，支持查询订单、创建工单、查询用户信息等操作。仅允许访问白名单内的端点。",
        "parameters": {
            "type": "object",
            "properties": {
                "endpoint": {
                    "type": "string", 
                    "description": "API端点路径，比如 /api/v1/orders、/api/v1/tickets/create"
                },
                "method": {
                    "type": "string", 
                    "description": "HTTP请求方法，支持GET/POST/PUT/DELETE，默认GET",
                    "default": "GET",
                    "enum": ["GET", "POST", "PUT", "DELETE"]
                },
                "params": {
                    "type": "object", 
                    "description": "URL查询参数，可选"
                },
                "body": {
                    "type": "object", 
                    "description": "POST/PUT请求的JSON请求体，可选"
                }
            },
            "required": ["endpoint"]
        }
    },
    handler=lambda args, **kw: internal_api_call(
        endpoint=args.get("endpoint"),
        method=args.get("method", "GET"),
        params=args.get("params"),
        body=args.get("body"),
        task_id=kw.get("task_id")
    ),
    check_fn=lambda: bool(os.getenv("INTERNAL_API_KEY")),
    requires_env=["INTERNAL_API_KEY", "INTERNAL_API_BASE"],
    category="enterprise"
)
```

#### 安全配置
1. 使用专用服务账号调用API，仅授予必要的权限
2. 配置API端点白名单，禁止访问敏感接口
3. 开启操作审计，所有API调用都会记录用户、参数、返回结果
4. 对返回结果中的敏感信息（手机号、身份证号、密码等）自动脱敏

---

### 2. 集成数据库
#### 适用场景
需要查询数据库中的数据，比如统计报表、查询用户信息、查询业务数据等，仅支持查询操作，禁止修改数据。

#### 实现步骤
1. 创建数据库只读账号，仅授予SELECT权限，禁止写入和修改权限
2. 开发数据库查询工具，封装SQL查询逻辑，内置安全检查
3. 配置数据库连接信息到环境变量（使用Secret管理敏感信息）
4. 配置查询限制，避免性能问题和数据泄露
5. 配置RBAC权限，仅允许授权用户使用数据库查询工具

#### 安全最佳实践
> 数据库是高风险系统，必须严格限制权限：
> 1. ✅ **只读账号**：必须使用只读账号连接数据库，禁止写入权限
> 2. ✅ **语句校验**：仅允许执行SELECT语句，禁止DDL（CREATE/DROP/ALTER）、DML（INSERT/UPDATE/DELETE）语句
> 3. ✅ **行数限制**：限制查询最多返回1000行，避免大量数据泄露
> 4. ✅ **超时限制**：查询超时时间设置为10秒，避免慢查询影响数据库性能
> 5. ✅ **敏感脱敏**：对返回结果中的敏感字段（手机号、身份证号、密码、地址等）自动脱敏
> 6. ✅ **禁止敏感表**：禁止访问用户表、权限表、配置表等敏感表
> 7. ✅ **审计日志**：所有查询操作都记录审计日志，包含用户、SQL、返回行数、执行时间等信息

#### 示例代码（MySQL）
```python
import json
import os
import re
from typing import Optional
import pymysql
from pymysql.converters import escape_string
from tools.registry import registry
import logging

logger = logging.getLogger(__name__)

# SQL安全检查正则
DANGEROUS_KEYWORDS = re.compile(
    r"\b(INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|TRUNCATE|REPLACE|GRANT|REVOKE|EXEC|EXECUTE)\b",
    re.IGNORECASE
)
SENSITIVE_TABLES = ["users", "user", "auth", "permission", "config", "secret"]
MAX_ROWS = 1000

def query_database(sql: str, limit: int = 100, task_id: Optional[str] = None) -> str:
    """
    查询企业内部MySQL数据库，仅支持SELECT查询
    :param sql: SELECT查询语句
    :param limit: 返回行数限制，最大1000，默认100
    """
    # 安全检查
    if DANGEROUS_KEYWORDS.search(sql):
        logger.warning(f"用户尝试执行危险SQL：{sql}, 任务ID：{task_id}")
        return json.dumps({
            "success": False,
            "error": "仅允许执行SELECT查询语句，禁止其他操作"
        }, ensure_ascii=False)
    
    if not sql.strip().upper().startswith("SELECT"):
        return json.dumps({
            "success": False,
            "error": "查询语句必须以SELECT开头"
        }, ensure_ascii=False)
    
    # 检查是否访问敏感表
    for table in SENSITIVE_TABLES:
        if re.search(rf"\b{table}\b", sql, re.IGNORECASE):
            return json.dumps({
                "success": False,
                "error": f"禁止访问敏感表：{table}"
            }, ensure_ascii=False)
    
    # 限制返回行数
    limit = min(limit, MAX_ROWS)
    
    # 读取数据库配置
    db_config = {
        "host": os.getenv("DB_HOST", "localhost"),
        "port": int(os.getenv("DB_PORT", 3306)),
        "user": os.getenv("DB_USER"),
        "password": os.getenv("DB_PASSWORD"),
        "database": os.getenv("DB_NAME"),
        "charset": "utf8mb4",
        "connect_timeout": 5,
        "read_timeout": 10
    }
    
    # 检查配置是否完整
    if not all([db_config["user"], db_config["password"], db_config["database"]]):
        return json.dumps({
            "success": False,
            "error": "数据库配置不完整，请联系管理员"
        }, ensure_ascii=False)
    
    connection = None
    try:
        connection = pymysql.connect(**db_config)
        with connection.cursor() as cursor:
            # 执行查询
            cursor.execute(sql)
            # 获取结果
            columns = [desc[0] for desc in cursor.description]
            rows = cursor.fetchmany(limit)
            
            # 敏感字段脱敏
            sensitive_columns = ["phone", "mobile", "email", "id_card", "password", "secret", "address"]
            processed_rows = []
            for row in rows:
                processed_row = list(row)
                for i, col in enumerate(columns):
                    col_lower = col.lower()
                    if any(sensitive in col_lower for sensitive in sensitive_columns):
                        processed_row[i] = "***" # 脱敏处理
                processed_rows.append(processed_row)
            
            return json.dumps({
                "success": True,
                "data": {
                    "columns": columns,
                    "rows": processed_rows,
                    "total": len(processed_rows)
                }
            }, ensure_ascii=False)
    
    except Exception as e:
        logger.error(f"数据库查询失败：{e}", exc_info=True)
        return json.dumps({
            "success": False,
            "error": f"查询失败：{str(e)}"
        }, ensure_ascii=False)
    finally:
        if connection:
            connection.close()

# 注册工具
registry.register(
    name="query_database",
    toolset="enterprise",
    schema={
        "name": "query_database",
        "description": "查询企业业务数据库，仅支持SELECT查询，可用于查询统计报表、业务数据等。",
        "parameters": {
            "type": "object",
            "properties": {
                "sql": {
                    "type": "string", 
                    "description": "SELECT查询语句，比如 SELECT * FROM orders WHERE create_time > '2024-01-01'"
                },
                "limit": {
                    "type": "integer", 
                    "description": "返回行数限制，最大1000，默认100",
                    "default": 100,
                    "minimum": 1,
                    "maximum": 1000
                }
            },
            "required": ["sql"]
        }
    },
    handler=lambda args, **kw: query_database(
        sql=args.get("sql"),
        limit=args.get("limit", 100),
        task_id=kw.get("task_id")
    ),
    check_fn=lambda: all([os.getenv("DB_HOST"), os.getenv("DB_USER"), os.getenv("DB_PASSWORD")]),
    requires_env=["DB_HOST", "DB_USER", "DB_PASSWORD", "DB_NAME"]
)
```

---

### 3. 集成企业单点登录（SSO）
#### 适用场景
企业需要统一身份认证，让用户使用企业现有账号登录Hermes Agent，不需要单独创建账号，符合企业安全规范。

#### 支持的认证方式
| 认证方式 | 适用场景 |
|----------|----------|
| **OAuth2/OIDC** | 通用标准协议，支持大部分身份提供商（Okta、Auth0、Azure AD、企业微信、钉钉等） |
| **LDAP/Active Directory** | 企业内部目录服务，适合传统企业架构 |
| **SAML2.0** | 企业级身份联合，支持跨域单点登录 |
| **企业内部账号集成** | 对接企业自有的账号体系 |

#### OAuth2/OIDC集成步骤（推荐）
1. 在企业身份提供商（IDP）中创建Hermes Agent应用，配置回调地址：`https://your-hermes-domain.com/api/auth/callback`
2. 获取Client ID和Client Secret
3. 在Hermes Agent配置文件中添加认证配置
4. 配置角色映射规则，将企业账号的角色同步到Hermes Agent的RBAC权限体系
5. 测试登录流程，验证权限正确

#### 配置示例
```yaml
auth:
  enabled: true
  type: oauth2
  oauth2:
    client_id: "your_client_id"
    client_secret: "your_client_secret"
    authorize_url: "https://idp.example.com/oauth2/authorize"
    token_url: "https://idp.example.com/oauth2/token"
    userinfo_url: "https://idp.example.com/oauth2/userinfo"
    logout_url: "https://idp.example.com/logout"
    scopes: ["openid", "profile", "email"]
    # 用户唯一标识字段
    user_id_field: "email"
    # 角色映射：IDP返回的角色名到Hermes角色的映射
    role_mapping:
      "admin": "admin"
      "developer": "developer"
      "staff": "user"
    # 默认角色：新用户默认分配的角色
    default_role: "user"
    # 允许登录的域名白名单，仅允许公司内部邮箱登录
    allowed_domains: ["company.com"]
```

#### LDAP集成配置示例
```yaml
auth:
  enabled: true
  type: ldap
  ldap:
    server: "ldap://ldap.example.com:389"
    bind_dn: "cn=admin,dc=example,dc=com"
    bind_password: "your_ldap_password"
    search_base: "ou=users,dc=example,dc=com"
    search_filter: "(uid=%s)"
    attributes:
      user_id: "uid"
      email: "mail"
      name: "cn"
      role: "title"
    role_mapping:
      "技术部管理员": "admin"
      "开发人员": "developer"
      "员工": "user"
```

---

### 4. 集成企业通知系统
#### 适用场景
需要将Hermes Agent的消息、任务状态、告警等推送到企业的通知系统，比如企业微信机器人、钉钉机器人、邮件、短信、飞书等，方便及时接收通知。

#### 支持的通知事件
| 事件类型 | 触发时机 |
|----------|----------|
| `message_received` | 收到用户消息时 |
| `message_sent` | 发送回复给用户时 |
| `task_completed` | 长时间运行的任务完成时 |
| `task_failed` | 任务执行失败时 |
| `tool_call` | 调用工具时（可配置高危操作通知） |
| `error_occurred` | 系统出现错误时 |
| `new_user_login` | 新用户首次登录时 |

#### 配置示例
```yaml
notifications:
  enabled: true
  # 全局通知开关，默认发送给管理员
  admin_notify:
    enabled: true
    events: ["error_occurred", "new_user_login", "task_failed"]
  # Webhook通知（支持企业微信、钉钉、飞书、自定义webhook等）
  webhooks:
    - name: "企业微信告警群"
      url: "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_key"
      type: "wecom" # 支持wecom/dingtalk/feishu/custom
      events: ["error_occurred", "task_failed"] # 仅发送错误和失败事件
      secret: "your_webhook_secret" # 用于签名验证
    - name: "任务通知群"
      url: "https://open.feishu.cn/open-apis/bot/v2/hook/your_hook_key"
      type: "feishu"
      events: ["task_completed"]
  # 邮件通知
  email:
    enabled: true
    smtp_server: "smtp.example.com"
    smtp_port: 587
    smtp_username: "notify@example.com"
    smtp_password: "your_email_password"
    recipients: ["admin@example.com", "dev@example.com"]
    events: ["error_occurred"]
```

### 5. 其他常见系统集成
#### 集成CI/CD系统
可以对接Jenkins、GitLab CI、GitHub Actions等CI/CD系统，实现：
- 触发构建任务
- 查询构建状态
- 查看构建日志
- 回滚版本
实现方式：开发对应的工具扩展，封装CI/CD系统的API调用。

#### 集成企业知识库
对接企业内部知识库、Confluence、语雀等，实现：
- 检索知识库内容
- 生成文档
- 回答基于知识库的问题
实现方式：开发知识库检索工具，或者使用RAG技术将知识库内容导入向量数据库，实现语义检索。

#### 集成监控告警系统
对接Prometheus、Grafana、Zabbix、企业内部监控系统等，实现：
- 查询监控指标
- 接收告警并自动排查问题
- 生成告警分析报告
实现方式：开发监控系统查询工具，配置告警webhook，收到告警时自动触发排查流程。

#### 集成代码仓库
对接GitLab、GitHub、Gitee等代码仓库，实现：
- 查询代码提交记录
- 代码审查
- 创建合并请求
- 查看Issue
实现方式：开发代码仓库工具，封装平台API。

---

## 企业级权限控制和安全方案

### 1. 基于角色的权限控制（RBAC）
通过RBAC实现细粒度的权限控制，不同角色分配不同的工具和功能权限，最小权限原则。

#### 配置示例
```yaml
rbac:
  enabled: true
  # 默认角色：新用户默认分配的角色
  default_role: "user"
  # 角色定义
  roles:
    admin:
      description: "超级管理员，拥有所有权限"
      permissions: ["*"]  # 所有权限
      allowed_tools: ["*"]
      allowed_skills: ["*"]
      allow_system_config: true # 允许修改系统配置
    developer:
      description: "开发人员，拥有代码、文件、终端权限"
      permissions: ["tools.*", "skills.*"]
      allowed_tools: ["tools.file.*", "tools.code.*", "tools.terminal.*"]
      allowed_skills: ["code*", "dev*"]
      allow_system_config: false
    ops:
      description: "运维人员，拥有服务器、监控、发布权限"
      permissions: ["tools.terminal.*", "tools.monitor.*", "tools.deploy.*"]
      allowed_tools: ["tools.terminal", "tools.monitor", "tools.deploy"]
    user:
      description: "普通用户，仅拥有基础查询权限"
      permissions: ["tools.web.search", "tools.knowledge.*"]
      allowed_tools: ["tools.web.search"]
      allow_system_config: false
  # 用户角色映射，支持通配符
  users:
    "admin@company.com": "admin"
    "dev*@company.com": "developer"
    "ops*@company.com": "ops"
    "*@company.com": "user"
  # 群组角色映射（对接企业组织架构时使用）
  groups:
    "技术部/开发组": "developer"
    "技术部/运维组": "ops"
    "产品部/*": "user"
```

#### 权限校验优先级
用户匹配规则从上到下优先级递减，找到第一个匹配的规则就停止：
1. 精确匹配的用户规则
2. 通配符匹配的用户规则
3. 精确匹配的群组规则
4. 通配符匹配的群组规则
5. 默认角色

### 2. 操作审计与合规
所有用户操作都会记录审计日志，满足等保合规要求。

#### 配置示例
```yaml
audit:
  enabled: true
  log_path: "~/.hermes/logs/audit.log"
  retention_days: 180 # 日志保留180天，满足等保要求
  # 记录的字段
  fields:
    - timestamp # 操作时间
    - user_id   # 用户ID
    - username  # 用户名
    - ip        # 登录IP
    - platform  # 登录平台
    - session_id # 会话ID
    - operation_type # 操作类型：message/tool_call/config_change/login
    - operation_detail # 操作详情，比如工具名、参数、配置变更内容
    - status    # 操作状态：success/failed
    - cost_time # 耗时
  # 日志输出方式，支持file、stdout、elasticsearch、syslog
  output: ["file", "elasticsearch"]
  elasticsearch:
    hosts: ["http://es-node1:9200", "http://es-node2:9200"]
    index: "hermes-audit-%{+yyyy.MM.dd}"
```

#### 审计日志查询
可以通过`hermes audit query`命令查询审计日志：
```bash
# 查询最近7天用户user1的所有工具调用
hermes audit query --user user1 --type tool_call --last 7d
# 查询所有失败的操作
hermes audit query --status failed
# 导出上个月的审计日志
hermes audit query --last 30d --format csv > audit-export.csv
```

### 3. 数据安全方案
#### 敏感信息保护
- 所有敏感信息（API密钥、数据库密码、认证信息等）都加密存储在环境变量或者Secret管理系统中，不会明文存储在配置文件或者数据库中
- 输出内容自动检测敏感信息，自动脱敏：手机号、身份证号、银行卡号、密码、密钥、地址等
- 禁止将敏感信息返回给用户，或者写入日志

#### 数据加密
- 会话数据、用户数据存储时自动加密
- 传输过程全部使用HTTPS加密
- 静态数据定期备份，备份数据加密存储

#### 数据泄露防护
- 配置敏感信息检测规则，发现用户尝试获取敏感数据时自动拦截
- 大文件导出需要审批
- 对外输出内容自动检测是否包含敏感信息，发现后自动拦截并告警

---

## ✍️ 练习
1. 集成你公司的内部API，实现查询员工信息或者订单信息的功能，注意做好安全校验
2. 配置RBAC权限控制，实现开发人员可以使用代码和终端工具，普通用户只能使用查询工具
3. 配置企业微信通知，当系统出现错误时自动发送告警到企业微信告警群
4. （可选）对接你们公司的SSO认证系统，实现单点登录

---

## ❓ 常见问题
### Q：集成内部API时遇到网络限制怎么办？
A：解决方案：
1. 在企业内部部署Hermes Agent，和内部系统在同一个网络区域
2. 配置HTTP/HTTPS代理，通过代理访问内部网络
3. 申请网络策略开通，允许Hermes Agent服务器访问内部API地址
4. 使用API网关做中转，统一访问内部接口

### Q：如何保障集成的安全性，避免数据泄露？
A：安全保障措施：
1. 所有敏感信息使用环境变量或者Secret系统管理，不要硬编码
2. 最小权限原则，给API账号和数据库账号授予最小必要的权限
3. 开启操作审计，所有操作都可以追溯
4. 敏感信息自动脱敏，避免泄露
5. 配置API调用频率限制，防止恶意调用或者误操作导致系统压力过大
6. 定期审计权限，及时回收不需要的权限

### Q：可以集成LDAP认证吗？
A：可以，Hermes Agent原生支持LDAP、OAuth2/OIDC、SAML2.0等多种认证方式，也支持对接企业自有的账号体系。

### Q：第三方系统故障会影响Hermes Agent的正常使用吗？
A：不会，设计上已经做了解耦：
1. 工具调用超时时间设置合理，不会因为第三方系统超时导致整个服务卡住
2. 第三方系统故障时会返回清晰的错误信息，不会影响其他功能的正常使用
3. 支持降级配置，第三方系统不可用时可以临时禁用对应的工具
4. 所有外部调用都有异常捕获，不会抛出未处理的异常导致服务崩溃

### Q：如何监控第三方集成的可用性？
A：监控方案：
1. 配置健康检查任务，定期调用第三方系统的探活接口，检测可用性
2. 配置指标监控，统计API调用成功率、平均耗时、错误率等指标
3. 配置告警规则，当错误率超过阈值或者耗时过长时自动通知管理员
4. 在审计日志中统计所有第三方调用的结果，定期分析可用性

---

## 📖 导航
- 上一篇：[自定义扩展开发](./3-custom-extension.md)
- 下一篇：[企业级部署最佳实践](./5-enterprise-best-practices.md)
- [返回目录](../SUMMARY.md)

---

## 📖 导航
- 上一篇：[自定义扩展开发](./3-custom-extension.md)
- 下一篇：[企业级部署最佳实践](./5-enterprise-best-practices.md)
- [返回目录](../SUMMARY.md)