# 5. 新增平台适配器

平台适配器是 Hermes Agent 对接各种消息平台的扩展层，通过新增适配器可以让 Hermes Agent 在飞书、企业微信、钉钉、Teams 等各种消息平台上运行。本文档介绍如何开发新的平台适配器。

## 概述

消息网关通过统一的适配器接口对接不同的消息平台，每个平台适配器实现统一的接口规范，上层逻辑不需要关心平台差异。新增平台适配器需要完成：适配器实现、全链路注册、测试三个步骤。

支持的平台类型：
- 即时通讯平台：飞书、企业微信、钉钉、Teams、Matrix 等
- 客服系统：智齿、美洽、环信等
- 自定义系统：自研的消息服务、客服系统等
- 通知渠道：邮件、短信、Webhook 等

## 适配器接口规范

所有平台适配器必须继承 `BasePlatformAdapter` 基类，实现指定的方法。

### 必须实现的方法
| 方法 | 用途 | 返回值要求 |
|------|------|------------|
| `__init__(self, config)` | 解析平台配置，初始化内部状态 | 必须调用 `super().__init__(config, Platform.YOUR_PLATFORM)` |
| `connect() -> bool` | 连接平台服务，启动消息监听器 | 连接成功返回 True，失败返回 False |
| `disconnect()` | 停止监听器，关闭连接，清理资源 | 无返回值 |
| `send(chat_id, text, **kwargs) -> SendResult` | 发送文本消息给指定用户/群组 | 返回 `SendResult` 对象，包含是否成功、消息ID等信息 |
| `send_typing(chat_id)` | 发送"输入中"指示器，提升用户体验 | 无返回值，不支持的平台可以空实现 |
| `send_image(chat_id, image_url, caption) -> SendResult` | 发送图片消息 | 返回 `SendResult` 对象，不支持的平台可以抛出 `NotImplementedError` |
| `get_chat_info(chat_id) -> dict` | 获取聊天对象的信息 | 返回字典包含 `name`、`type`、`chat_id` 字段 |

### 必须实现的辅助函数
```python
def check_<platform>_requirements() -> bool:
    """检查平台依赖是否满足，返回True表示可以使用该平台"""
    # 检查依赖的环境变量是否配置
    # 检查依赖的Python库是否安装
    return True
```

## 完整开发示例

以下是新增飞书平台适配器的完整示例：

### 1. 创建适配器文件
在 `gateway/platforms/feishu.py` 创建适配器实现：
```python
"""飞书平台适配器"""
import os
import json
from typing import Optional
import requests
from lark_oapi.api.im.v1 import *
from lark_oapi import Client, AuthType, LogLevel
from gateway.config import Platform
from gateway.base_adapter import BasePlatformAdapter, SendResult

class FeishuAdapter(BasePlatformAdapter):
    def __init__(self, config):
        super().__init__(config, Platform.FEISHU)
        # 从配置中获取飞书凭证
        self.app_id = os.getenv("FEISHU_APP_ID")
        self.app_secret = os.getenv("FEISHU_APP_SECRET")
        self.verification_token = os.getenv("FEISHU_VERIFICATION_TOKEN")
        self.encrypt_key = os.getenv("FEISHU_ENCRYPT_KEY")
        # 初始化飞书客户端
        self.client = Client.builder() \
            .app_id(self.app_id) \
            .app_secret(self.app_secret) \
            .auth_type(AuthType.INTERNAL_APP) \
            .log_level(LogLevel.ERROR) \
            .build()
        # 存储用户open_id和chat_id映射
        self.user_chat_map = {}

    def connect(self) -> bool:
        """启动webhook服务监听飞书消息"""
        try:
            # 启动HTTP服务接收飞书事件回调
            self.start_webhook_server(port=7867)
            self.logger.info("飞书适配器连接成功，webhook服务启动在端口7867")
            return True
        except Exception as e:
            self.logger.error(f"飞书适配器连接失败: {e}", exc_info=True)
            return False

    def disconnect(self):
        """停止webhook服务，清理资源"""
        self.stop_webhook_server()
        self.logger.info("飞书适配器已断开连接")

    def send(self, chat_id: str, text: str, **kwargs) -> SendResult:
        """发送文本消息"""
        try:
            # 构造飞书消息
            content = json.dumps({"text": text})
            request = CreateMessageRequest.builder() \
                .receive_id_type("open_id") \
                .receive_id(chat_id) \
                .msg_type("text") \
                .content(content) \
                .build()
            # 发送消息
            response = self.client.im.v1.message.create(request)
            if response.success():
                return SendResult(
                    success=True,
                    message_id=response.data.message_id,
                    raw_response=response.raw.content
                )
            else:
                self.logger.error(f"发送飞书消息失败: {response.msg}")
                return SendResult(success=False, error=response.msg)
        except Exception as e:
            self.logger.error(f"发送飞书消息异常: {e}", exc_info=True)
            return SendResult(success=False, error=str(e))

    def send_typing(self, chat_id: str):
        """发送输入中状态"""
        try:
            # 飞书暂不支持输入中状态，空实现
            pass
        except Exception as e:
            self.logger.debug(f"发送输入中状态失败: {e}")

    def send_image(self, chat_id: str, image_url: str, caption: str = "") -> SendResult:
        """发送图片消息"""
        try:
            # 先上传图片到飞书服务器
            image_key = self._upload_image(image_url)
            # 构造图片消息
            content = json.dumps({
                "image_key": image_key
            })
            request = CreateMessageRequest.builder() \
                .receive_id_type("open_id") \
                .receive_id(chat_id) \
                .msg_type("image") \
                .content(content) \
                .build()
            response = self.client.im.v1.message.create(request)
            if response.success():
                return SendResult(
                    success=True,
                    message_id=response.data.message_id,
                    raw_response=response.raw.content
                )
            else:
                return SendResult(success=False, error=response.msg)
        except Exception as e:
            return SendResult(success=False, error=str(e))

    def get_chat_info(self, chat_id: str) -> dict:
        """获取聊天信息"""
        try:
            request = GetChatRequest.builder() \
                .chat_id(chat_id) \
                .build()
            response = self.client.im.v1.chat.get(request)
            if response.success():
                return {
                    "name": response.data.name,
                    "type": response.data.chat_type,
                    "chat_id": chat_id
                }
            return {"name": "未知", "type": "unknown", "chat_id": chat_id}
        except Exception as e:
            return {"name": "未知", "type": "unknown", "chat_id": chat_id}

    def _upload_image(self, image_url: str) -> str:
        """上传图片到飞书服务器，返回image_key"""
        # 实现图片上传逻辑
        pass

    def _handle_event(self, event):
        """处理飞书回调事件"""
        # 验证事件签名
        if not self._verify_signature(event):
            return
        # 处理消息事件
        if event.get("header", {}).get("event_type") == "im.message.receive_v1":
            message = event.get("event", {}).get("message", {})
            sender = event.get("event", {}).get("sender", {})
            chat_id = sender.get("sender_id", {}).get("open_id")
            content = json.loads(message.get("content", "{}")).get("text", "")
            # 过滤自身消息
            if message.get("sender", {}).get("sender_type") == "app":
                return
            # 构造来源信息
            source = self.build_source(
                user_id=chat_id,
                username=sender.get("sender_id", {}).get("name", "未知用户"),
                chat_id=chat_id,
                chat_type="user"
            )
            # 分发消息到网关处理
            self.handle_message(content, source)

def check_feishu_requirements() -> bool:
    """检查飞书平台依赖是否满足"""
    # 检查依赖的环境变量是否配置
    required_env = ["FEISHU_APP_ID", "FEISHU_APP_SECRET", "FEISHU_VERIFICATION_TOKEN"]
    for env in required_env:
        if not os.getenv(env):
            return False
    # 检查依赖的Python库是否安装
    try:
        import lark_oapi
    except ImportError:
        return False
    return True
```

## 全链路注册

适配器实现完成后，需要在多个文件中注册，完成全链路集成：

### 1. 添加平台枚举
编辑 `gateway/config.py`，在 `Platform` 枚举中添加新平台：
```python
class Platform(StrEnum):
    # ... 现有平台 ...
    FEISHU = "feishu"
```

添加环境变量加载逻辑：
```python
# 在_configure_platforms方法中添加
if os.getenv("FEISHU_APP_ID") and os.getenv("FEISHU_APP_SECRET"):
    config.enabled_platforms.append(Platform.FEISHU)
```

### 2. 注册适配器工厂
编辑 `gateway/run.py`，在 `_create_adapter` 方法中添加适配器创建逻辑：
```python
def _create_adapter(platform: Platform, config: GatewayConfig) -> BasePlatformAdapter:
    if platform == Platform.FEISHU:
        from gateway.platforms.feishu import FeishuAdapter
        return FeishuAdapter(config)
    # ... 现有其他平台的逻辑 ...
```

### 3. 添加权限控制配置
编辑 `gateway/run.py`，在权限控制映射中添加平台对应的环境变量：
```python
platform_env_map = {
    # ... 现有平台 ...
    Platform.FEISHU: "FEISHU_ALLOWED_USERS",
}

platform_allow_all_map = {
    # ... 现有平台 ...
    Platform.FEISHU: "FEISHU_ALLOW_ALL_USERS",
}
```

### 4. 添加平台特性提示
编辑 `agent/prompt_builder.py`，在 `PLATFORM_HINTS` 中添加平台特性说明：
```python
PLATFORM_HINTS = {
    # ... 现有平台 ...
    Platform.FEISHU: "当前平台是飞书，支持发送文本、图片、卡片消息，不支持文件上传。",
}
```

### 5. 添加到工具集
编辑 `toolsets.py`，添加平台专属工具集，并加入 `hermes-gateway` 复合工具集：
```python
# 新增飞书工具集
FEISHU_TOOLS = Toolset(
    name="feishu",
    description="飞书平台专属工具",
    tools=[
        # 飞书专属工具
    ]
)

# 添加到网关工具集
HERMES_GATEWAY = Toolset(
    name="hermes-gateway",
    description="网关模式默认工具集",
    includes=[
        # ... 现有工具集 ...
        FEISHU_TOOLS,
    ]
)
```

### 6. 添加到设置向导
编辑 `hermes_cli/gateway.py`，添加飞书平台的设置向导，引导用户配置环境变量。

### 7. 添加敏感信息脱敏
编辑 `agent/redact.py`，添加飞书相关的敏感标识符脱敏规则，避免日志泄露 `FEISHU_APP_SECRET` 等敏感信息。

## 测试验证

### 1. 依赖检查
运行健康检查，确认平台依赖检测正常：
```bash
hermes doctor
```
飞书平台应该显示已配置。

### 2. 单元测试
编写单元测试覆盖适配器的所有方法：
```python
# tests/gateway/platforms/test_feishu.py
from unittest.mock import patch, Mock
from gateway.platforms.feishu import FeishuAdapter, check_feishu_requirements

def test_check_requirements():
    with patch.dict("os.environ", {
        "FEISHU_APP_ID": "test_id",
        "FEISHU_APP_SECRET": "test_secret",
        "FEISHU_VERIFICATION_TOKEN": "test_token"
    }):
        assert check_feishu_requirements() == True

def test_send_message():
    adapter = FeishuAdapter({})
    # Mock飞书客户端返回成功
    with patch.object(adapter.client.im.v1.message, 'create') as mock_create:
        mock_response = Mock()
        mock_response.success.return_value = True
        mock_response.data.message_id = "msg_123"
        mock_create.return_value = mock_response
        
        result = adapter.send("user_123", "Hello Feishu")
        assert result.success == True
        assert result.message_id == "msg_123"
```

运行测试：
```bash
pytest tests/gateway/platforms/test_feishu.py -v
```

### 3. 集成测试
配置真实的飞书应用凭证，启动网关测试：
```bash
hermes gateway start --platforms feishu
```
发送消息给飞书机器人，确认能够正常接收和回复。

### 4. 集成点检查
运行以下命令确认所有集成点都已添加：
```bash
grep -r "feishu" gateway/ tools/ agent/ cron/ hermes_cli/ toolsets.py \
  --include="*.py" -l | sort -u
```
确保所有需要修改的文件都已添加飞书平台的支持。

## 最佳实践

### 可靠性设计
- 实现指数退避的重连机制，网络异常时自动重连
- 消息发送失败时自动重试，最多重试3次
- 所有异常都要捕获，避免适配器崩溃导致整个网关退出
- 完善的日志记录，方便排查问题

### 安全设计
- 验证所有回调事件的签名，防止伪造请求
- 敏感信息不要输出到日志，使用脱敏处理
- 严格遵循平台的权限控制，只申请必要的权限
- 用户消息过滤，防止恶意内容攻击

### 性能设计
- 消息处理采用异步方式，避免阻塞
- 合理设置超时时间，防止长时间等待
- 批量处理消息，提高吞吐量
- 缓存用户信息，减少重复请求平台API

### 兼容性设计
- 兼容平台API的多个版本，避免API升级导致适配器失效
- 不支持的功能优雅降级，不要抛出异常
- 适配不同类型的聊天场景（单聊、群聊、频道）
- 支持不同格式的消息内容（文本、图片、卡片、文件）
