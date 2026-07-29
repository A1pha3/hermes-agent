# 3. 自定义扩展开发

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 了解 Hermes Agent 的扩展开发体系和设计思路
> - 独立完成工具、技能、平台三种类型的扩展开发
> - 掌握扩展的调试、测试、打包、发布全流程
> - 遵循开发规范，开发高质量可复用的扩展

## 扩展开发体系概述

Hermes Agent 采用**核心+扩展**的模块化设计，核心代码保持稳定，通过扩展满足定制化需求，避免对核心代码的侵入式修改：

| 扩展类型 | 适用场景 | 开发难度 | 代码侵入性 |
|----------|----------|----------|----------|
| **工具扩展** | 新增和外部系统交互的能力（比如调用内部API、操作数据库等） | 低 | 低，仅需新增文件和注册 |
| **技能扩展** | 新增垂直领域的工作流和能力（比如代码审计、文档生成等） | 中 | 无侵入，完全独立部署 |
| **平台扩展** | 新增支持的消息/IDE平台（比如新增企业微信、飞书、自定义IDE等） | 高 | 低，仅需新增适配器和注册 |

> 设计思路：
> 1. **无侵入优先**：优先选择技能扩展，不需要修改核心代码，升级不丢失
> 2. **模块化设计**：每个扩展独立，互不影响，方便测试和维护
> 3. **通用注册机制**：所有扩展都通过统一的注册机制接入，不需要修改核心逻辑
> 4. **隔离运行**：扩展运行在独立的上下文，不会影响核心服务稳定性

## 扩展开发准备

### 环境要求
- 已完成[开发环境搭建](../development/1-env-setup.md)
- 熟悉 Python 3.10+ 语法
- 了解 Hermes Agent 的基本架构和扩展点

### 通用开发流程
1. 按照扩展类型选择对应的模板初始化项目
2. 实现核心逻辑，遵循统一的接口规范
3. 编写单元测试，覆盖正常和异常场景
4. 本地测试验证功能正常
5. 打包发布，或者提交到内部扩展市场
6. 安装使用，收集反馈迭代优化

---

## 扩展开发准备

### 环境要求
- 已完成[开发环境搭建](../development/1-env-setup.md)
- 熟悉 Python 3.10+ 语法
- 了解 Hermes Agent 的基本架构

### 开发流程
1. 按照扩展类型选择对应的模板
2. 实现核心逻辑
3. 本地测试验证
4. 打包发布
5. 安装使用

---

## 1. 工具扩展开发

工具扩展用于新增和外部系统交互的能力，是最常用的扩展类型，比如调用内部API、操作数据库、对接内部服务等。

> 工具设计原则：
> 1. **单一职责**：一个工具只做一件事，功能明确
> 2. **幂等性**：重复调用不会产生副作用，避免误操作
> 3. **安全优先**：敏感操作需要参数校验和权限控制
> 4. **友好错误**：错误信息明确，模型可以理解并修正参数重试

### 步骤1：创建工具文件
在 `hermes_agent/tools/` 目录下创建你的工具文件，命名规范：`{分类}_{工具名}.py`，比如 `custom_weather.py`：

```python
import json
import os
import requests
from typing import Dict, Any
from tools.registry import registry

def check_requirements() -> bool:
    """检查工具运行依赖是否满足，返回False时工具会自动隐藏"""
    # 检查依赖库是否安装
    try:
        import requests
    except ImportError:
        return False
    # 检查必填环境变量是否配置
    if not os.getenv("WEATHER_API_KEY"):
        return False
    return True

def query_weather(city: str, unit: str = "celsius", task_id: str = None) -> str:
    """
    工具核心逻辑，必须返回JSON格式字符串
    :param city: 要查询的城市名
    :param unit: 温度单位，celsius摄氏度/fahrenheit华氏度
    :param task_id: 任务ID，系统自动传入，用于日志跟踪
    """
    api_key = os.getenv("WEATHER_API_KEY")
    
    # 参数校验
    if not city:
        return json.dumps({
            "success": False,
            "error": "城市名不能为空"
        }, ensure_ascii=False)
    
    try:
        # 调用外部API
        url = f"https://api.weatherapi.com/v1/current.json?key={api_key}&q={city}"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
        
        # 格式化返回结果
        result = {
            "success": True,
            "data": {
                "city": data["location"]["name"],
                "temperature": data["current"]["temp_c"] if unit == "celsius" else data["current"]["temp_f"],
                "unit": unit,
                "condition": data["current"]["condition"]["text"],
                "humidity": data["current"]["humidity"]
            },
            "task_id": task_id
        }
        return json.dumps(result, ensure_ascii=False)
    
    except Exception as e:
        # 捕获所有异常，返回结构化错误信息
        return json.dumps({
            "success": False,
            "error": f"查询天气失败: {str(e)}"
        }, ensure_ascii=False)

# 注册工具到中心
registry.register(
    name="query_weather", # 全局唯一的工具名，使用下划线分隔
    toolset="custom", # 所属工具集，用于批量开关
    schema={
        "name": "query_weather",
        "description": "查询指定城市的实时天气信息，包括温度、湿度、天气状况等",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "要查询的城市名称，支持中英文，例如：北京、Shanghai、New York"
                },
                "unit": {
                    "type": "string",
                    "description": "温度单位，可选值：celsius（摄氏度，默认）、fahrenheit（华氏度）",
                    "default": "celsius",
                    "enum": ["celsius", "fahrenheit"]
                }
            },
            "required": ["city"] # 必填参数列表
        }
    },
    handler=lambda args, **kw: query_weather(
        city=args.get("city"),
        unit=args.get("unit", "celsius"),
        task_id=kw.get("task_id")
    ),
    check_fn=check_requirements, # 依赖检查函数
    requires_env=["WEATHER_API_KEY"], # 依赖的环境变量，配置时会提示用户
    icon="☀️", # 工具图标，用于UI展示
)
```

### 步骤2：注册工具
在 `model_tools.py` 的 `_discover_tools()` 函数中添加你的工具导入（仅需导入，不需要其他操作，导入时会自动执行注册逻辑）：
```python
# 导入自定义工具
import tools.custom_weather
```

### 步骤3：启用工具
在配置文件中添加工具集到启用列表：
```yaml
tools:
  enabled_toolsets:
    - core
    - custom # 启用自定义工具集
```

### 工具调试技巧
1. 使用`hermes tools list`命令查看工具是否正常注册
2. 使用`hermes tools call query_weather '{"city": "北京"}'`命令直接调用工具测试，不需要对话
3. 开启`--debug`模式可以查看工具调用的详细参数和返回结果
4. 工具返回的JSON格式必须严格正确，否则模型无法解析

---

## 2. 技能扩展开发

技能扩展用于新增垂直领域的工作流和专业能力，不需要修改核心代码，是最推荐的扩展方式，适合实现比如代码审计、文档生成、数据处理、内部运维等定制化工作流。

> 技能优势：
> 1. 完全无侵入，不需要修改核心代码，升级不丢失
> 2. 热加载，修改技能代码不需要重启服务
> 3. 可以独立打包分发，方便内部共享
> 4. 支持按用户/角色授权使用不同技能

### 技能类型
技能分为两种类型，根据需求选择：
1. **声明式技能**：仅用Markdown编写提示词和工作流，不需要代码，适合简单的流程定制
2. **代码式技能**：使用Python编写复杂的逻辑和工具调用，适合复杂的工作流场景

### 步骤1：创建技能目录
在 `~/.hermes/skills/` 目录下创建你的技能目录，结构如下：
```
code_review_skill/
├── SKILL.md          # 技能定义文件，包含提示词、工作流（声明式技能只需要这个文件）
├── skill.json        # 技能元数据，必填
├── main.py           # 技能核心逻辑（代码式技能需要）
├── requirements.txt  # 技能依赖（代码式技能需要）
├── README.md         # 技能说明文档
└── examples/         # 使用示例
```

### 步骤2：编写技能元数据
`skill.json` 示例：
```json
{
  "name": "code_review", // 全局唯一技能名
  "version": "1.0.0",    // 语义化版本
  "description": "代码审计技能，自动审查Python代码的质量、安全、性能问题",
  "author": "技术部",
  "trigger_phrases": ["代码审查", "帮我审代码", "检查这段代码"], // 触发关键词，用户输入包含这些词会自动调用技能
  "requires_env": ["CODE_QUALITY_API_KEY"], // 依赖的环境变量
  "category": "development", // 技能分类，用于整理展示
  "icon": "🔍"
}
```

### 步骤3：实现技能逻辑
#### 声明式技能（不需要代码）
直接编写`SKILL.md`，包含技能的工作流、提示词、规则：
```markdown
# 代码审查技能
## 角色
你是资深的Python代码审查专家，严格遵循Google代码规范。

## 工作流程
1. 先检查代码的语法错误和明显bug
2. 再检查安全漏洞，比如SQL注入、XSS、命令注入等
3. 再检查性能问题，比如低效的循环、内存泄漏等
4. 最后给出优化建议，按严重程度分级

## 输出格式
### 审查结果
- 严重问题：[问题列表]
- 重要问题：[问题列表]
- 优化建议：[建议列表]

### 代码优化版本
```python
[优化后的代码]
```
```
#### 代码式技能（需要Python逻辑）
编写`main.py`，实现`handle_request`函数：
```python
from typing import Dict, Any
import os
from my_code_checker import check_code_quality

def handle_request(query: str, context: Dict[str, Any]) -> str:
    """
    处理技能请求
    :param query: 用户的查询内容
    :param context: 上下文，包含会话信息、用户信息、工具调用能力等
    :return: 技能执行结果，返回给用户
    """
    # 从上下文提取用户发送的代码
    code = context.get("user_input").get("code", "")
    api_key = os.getenv("CODE_QUALITY_API_KEY")
    
    # 调用内部代码检查服务
    result = check_code_quality(code, api_key)
    
    # 格式化返回结果
    return f"""### 代码审查结果
严重问题：{result.get('critical', 0)}个
重要问题：{result.get('major', 0)}个
优化建议：{result.get('suggestions', [])}

### 优化后代码
```python
{result.get('optimized_code')}
```
"""
```

### 步骤4：安装和测试技能
执行命令安装技能：
```bash
hermes skills install ~/.hermes/skills/code_review_skill
```
测试技能：
```bash
hermes chat -q "帮我审查这段代码 print('hello world')"
# 会自动触发代码审查技能
```

### 技能调试技巧
1. 使用`hermes skills list`查看已安装的技能列表
2. 使用`hermes skills debug code_review`查看技能的元数据和加载状态
3. 技能代码修改后会自动重载，不需要重启服务
4. 技能日志输出到`~/.hermes/logs/skills/{技能名}.log`，方便排查问题

---

## 3. 平台扩展开发

平台扩展用于新增支持的消息或IDE平台，让Hermes Agent可以在更多渠道提供服务，比如新增企业微信、钉钉、Slack、自定义IDE插件等。

> 平台适配器设计原则：
> 1. 遵循统一的抽象接口，上层网关不需要关心具体平台的实现差异
> 2. 所有平台相关的逻辑都封装在适配器内部，对外暴露统一的方法
> 3. 异常处理完善，平台API调用失败不会影响核心服务
> 4. 支持配置热加载，修改配置不需要重启服务

### 步骤1：创建平台适配器
在 `hermes_agent/gateway/platforms/` 目录下创建适配器文件，比如 `wecom.py`（企业微信适配器），继承`BasePlatform`基类并实现所有抽象方法：

```python
from typing import Dict, Any, List
import asyncio
import logging
from gateway.platforms.base import BasePlatform
from gateway.message import Message

logger = logging.getLogger(__name__)

class WecomPlatform(BasePlatform):
    """企业微信平台适配器"""
    
    def __init__(self, config: Dict[str, Any]):
        super().__init__(config)
        self.app_id = config.get("app_id")
        self.app_secret = config.get("app_secret")
        self.agent_id = config.get("agent_id")
        self.token = config.get("token")
        self.encoding_aes_key = config.get("encoding_aes_key")
        self.access_token = None
        self.access_token_expire = 0
        
        # 白名单配置
        self.allowed_users = config.get("allowed_users", [])
        self.allow_all = config.get("allow_all_users", False)

    async def connect(self):
        """连接到平台，初始化资源，启动监听"""
        logger.info("正在连接企业微信平台")
        try:
            # 1. 获取access_token
            await self._refresh_access_token()
            # 2. 启动webhook服务或者长连接监听消息
            await self._start_message_listener()
            logger.info("企业微信平台连接成功")
        except Exception as e:
            logger.error(f"企业微信连接失败: {e}", exc_info=True)
            raise

    async def send_message(self, user_id: str, message: str, **kwargs) -> bool:
        """
        发送消息给用户
        :param user_id: 用户ID
        :param message: 消息内容，支持Markdown格式
        :param kwargs: 附加参数，比如@用户、附件等
        :return: 发送是否成功
        """
        try:
            # 转换Markdown为企业微信支持的格式
            wecom_msg = self._convert_markdown_to_wecom_format(message)
            # 调用企业微信API发送消息
            await self._call_wecom_api("message/send", {
                "touser": user_id,
                "agentid": self.agent_id,
                "msgtype": "markdown",
                "markdown": {"content": wecom_msg}
            })
            return True
        except Exception as e:
            logger.error(f"发送消息给{user_id}失败: {e}")
            return False

    async def receive_messages(self) -> List[Message]:
        """接收消息，转换为内部统一的Message格式"""
        # 从消息队列中获取接收到的平台消息
        messages = await self._message_queue.get()
        # 转换为内部统一格式
        internal_messages = []
        for msg in messages:
            # 检查用户权限
            if not self._check_user_permission(msg.get("user_id")):
                logger.warning(f"用户{msg.get('user_id')}无访问权限，消息已忽略")
                continue
                
            internal_msg = Message(
                user_id=msg.get("user_id"),
                username=msg.get("username"),
                content=msg.get("content"),
                chat_id=msg.get("chat_id"),
                chat_type=msg.get("chat_type", "user"),
                platform=self.platform_name
            )
            internal_messages.append(internal_msg)
        return internal_messages

    async def disconnect(self):
        """断开连接，清理资源"""
        logger.info("正在断开企业微信平台连接")
        await self._stop_message_listener()
        logger.info("企业微信平台已断开连接")

    # 内部辅助方法
    async def _refresh_access_token(self):
        """刷新access_token，过期自动更新"""
        # 实现刷新逻辑
        pass
        
    def _convert_markdown_to_wecom_format(self, markdown: str) -> str:
        """转换通用Markdown为企业微信支持的格式"""
        # 企业微信的Markdown有部分语法差异，需要转换
        return markdown.replace("```python", "```").replace("**", "*")
        
    def _check_user_permission(self, user_id: str) -> bool:
        """检查用户是否有权限访问"""
        if self.allow_all:
            return True
        return user_id in self.allowed_users
```

### 步骤2：注册平台
在 `gateway/run.py` 的 `platform_registry` 中添加你的平台，让网关可以识别和加载：
```python
platform_registry = {
    "telegram": TelegramPlatform,
    "discord": DiscordPlatform,
    "feishu": FeishuPlatform,
    "wecom": WecomPlatform,  # 新增企业微信平台
}
```

### 步骤3：配置平台
在配置文件中添加平台的配置项：
```yaml
gateway:
  enabled_platforms:
    - wecom # 启用企业微信平台
  wecom:
    app_id: "你的企业微信app_id"
    app_secret: "你的app_secret"
    agent_id: 1000001
    token: "你的回调token"
    encoding_aes_key: "你的加密密钥"
    allowed_users: ["user1", "user2"] # 允许访问的用户列表
    allow_all_users: false # 是否允许所有用户访问
```

### 平台扩展调试技巧
1. 使用`hermes gateway test wecom`命令测试平台连接和配置是否正确
2. 平台相关日志输出到`~/.hermes/logs/gateway-wecom.log`
3. 开发过程中可以用平台的测试工具发送消息，验证接收逻辑是否正常
4. 发送消息时可以开启debug模式，查看API调用的详细参数和返回结果

---

## 扩展打包与发布

开发完成的扩展可以打包分发，方便团队内部共享或者提交到官方市场。

### 扩展结构规范
打包前确保扩展符合以下目录结构：
```
my-extension/
├── pyproject.toml       # 包配置文件，包含元数据和依赖
├── README.md            # 扩展说明文档
├── LICENSE              # 开源协议
├── src/
│   └── my_extension/    # 扩展代码
│       ├── __init__.py
│       └── ...
└── tests/               # 单元测试
    └── test_my_extension.py
```

### 打包为可安装包
使用 `poetry` 打包你的扩展（没有poetry的话先安装`pip install poetry`）：
1. 初始化包配置：
```bash
poetry init
```
按照提示填写包名、版本、作者、依赖等信息。

2. 构建包：
```bash
poetry build
```
会在`dist/`目录下生成`.whl`和`.tar.gz`两种格式的安装包。

3. 本地安装测试：
```bash
pip install dist/my_extension-1.0.0-py3-none-any.whl
```

### 发布方式
#### 1. 发布到官方技能市场
如果你想分享你的扩展给所有用户，可以提交到官方技能市场：
1. Fork 官方技能市场仓库
2. 添加你的扩展信息到列表，包含名称、描述、版本、仓库地址等
3. 编写详细的使用文档和示例
4. 提交 Pull Request，审核通过后会正式上线

#### 2. 内部私有发布
企业内部使用的扩展可以发布到内部PyPI仓库或者Git仓库：
1. 配置内部PyPI源：
```bash
poetry config repositories.my-internal https://pypi.mycompany.com/simple/
poetry config http-basic.my-internal username password
```
2. 发布到内部源：
```bash
poetry publish --repository my-internal
```
3. 员工安装：
```bash
pip install --index-url https://pypi.mycompany.com/simple/ my-extension
```

### 版本管理规范
扩展版本使用语义化版本号：`主版本号.次版本号.修订号`
- 主版本号：不兼容的API修改
- 次版本号：向下兼容的功能新增
- 修订号：向下兼容的问题修复
示例：`1.0.0`、`1.1.0`、`2.0.0`

---

## ✍️ 练习
1. 开发一个简单的天气查询工具扩展，调用公开的天气API，支持按城市查询
2. 开发一个代码审查的声明式技能，实现自动审查Python代码的功能
3. 尝试打包你开发的工具，发布到内部测试源，让同事安装使用
4. （可选）开发一个简单的企业微信/钉钉平台适配器，实现消息收发功能

---

## ❓ 常见问题
### Q：开发的扩展在升级 Hermes Agent 后会丢失吗？
A：
- **技能扩展**：完全不会丢失，保存在用户目录`~/.hermes/skills/`下，升级不影响
- **工具/平台扩展**：如果直接修改核心代码目录，升级时会被覆盖，建议：
  1. 将自定义扩展做成独立的包，安装到Python环境中，不需要修改核心代码
  2. 维护自己的fork分支，升级时合并上游更新
  3. 优先使用技能扩展满足需求，不需要修改核心代码

### Q：可以使用其他语言开发扩展吗？
A：核心扩展需要使用Python，但你可以通过两种方式对接其他语言：
1. 开发一个Python壳工具，内部调用其他语言开发的HTTP服务或者命令行工具
2. 将其他语言开发的服务封装成API，在技能或者工具中调用

### Q：如何调试扩展？
A：调试技巧：
1. 使用 `hermes --debug` 模式运行，查看详细的日志输出
2. 工具扩展可以用 `hermes tools call {工具名} {参数JSON}` 直接调用测试
3. 技能扩展可以查看技能日志 `~/.hermes/logs/skills/{技能名}.log`
4. 平台扩展可以用 `hermes gateway test {平台名}` 测试连接和消息收发
5. 可以在代码中加入`import pdb; pdb.set_trace()`断点调试

### Q：扩展开发有哪些安全注意事项？
A：安全开发规范：
1. 所有用户输入都要做校验和过滤，避免注入攻击
2. 敏感信息（API密钥、密码等）不要硬编码到代码中，通过环境变量配置
3. 工具执行的命令不要直接拼接用户输入，使用参数化方式
4. 文件操作要限制在允许的目录范围内，避免路径遍历漏洞
5. 对外发布的扩展要做安全审计，避免包含恶意代码

### Q：扩展可以调用其他扩展的功能吗？
A：可以，通过统一的注册中心获取其他工具的handler调用，或者通过技能调用API，建议尽量减少扩展之间的依赖，保持扩展的独立性。

---

## 📖 导航
- 上一篇：[生产环境部署](./2-production-deployment.md)
- 下一篇：[集成第三方系统](./4-third-party-integration.md)
- [返回目录](../SUMMARY.md)

---

## 📖 导航
- 上一篇：[生产环境部署](./2-production-deployment.md)
- 下一篇：[集成第三方系统](./4-third-party-integration.md)
- [返回目录](../SUMMARY.md)