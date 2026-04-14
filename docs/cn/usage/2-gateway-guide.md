# 2. 消息网关使用

Hermes Agent 消息网关支持将 AI 助手部署到多种主流消息平台，为团队和用户提供统一的 AI 服务入口。

## 概述

消息网关是 Hermes Agent 的核心组件之一，负责对接各种消息平台，将用户消息转发给 Agent 处理，再将响应返回给用户。网关支持多平台同时接入，不同平台的会话完全隔离，权限独立控制。

## 支持的平台列表

目前支持 17+ 消息和服务平台：

| 分类 | 平台名称 | 说明 |
|------|----------|------|
| 即时通讯 | Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Mattermost | 主流聊天平台 |
| 企业协作 | 飞书（Feishu）、企业微信（WeCom）、微信（Weixin）、钉钉（DingTalk） | 国内企业常用平台 |
| 通知类 | SMS、Email、Webhook | 消息通知类服务 |
| 服务类 | API Server、HomeAssistant、BlueBubbles | 智能家居和服务集成 |

## 通用配置流程

所有平台的配置遵循统一的流程：

### Step 1：配置平台凭据
在 `~/.hermes/.env` 文件中添加对应平台的 API 密钥、访问令牌等凭据。不同平台需要的凭据不同，具体参考各平台的配置示例。

### Step 2：启用平台
在 `~/.hermes/config.yaml` 中设置对应平台的 `enabled: true`。

### Step 3：配置访问权限
配置允许访问的用户 ID/手机号/邮箱列表，支持使用 `*` 通配符允许所有用户访问（不推荐生产环境使用）。

### Step 4：可选功能配置
根据需要配置自动回复规则、背景任务通知级别、会话超时时间、命令审批开关等功能。

## 部署启动方式

### 1. 前台运行
```bash
hermes gateway
# 或者
python -m gateway.run
```
直接在终端运行网关，日志输出到 stdout，适合调试和测试场景。

### 2. 系统服务安装
```bash
hermes gateway install
```
自动生成 systemd（Linux）或 launchd（macOS）服务配置，将网关安装为系统后台服务，开机自动启动。

### 3. 服务管理
```bash
hermes gateway start    # 启动网关服务
hermes gateway stop     # 停止网关服务
hermes gateway status   # 查看服务状态
hermes gateway restart  # 重启网关服务
```
管理网关服务的生命周期，适合生产环境部署。

## 权限控制机制

网关提供多层权限控制，确保使用安全：

### 1. 访问白名单
每个平台支持独立配置允许访问的用户 ID 列表，未授权用户的消息会被直接拒绝，不会进入 Agent 处理流程。

示例（WhatsApp 配置）：
```env
WHATSAPP_ALLOWED_USERS=15551234567,15557654321
```

### 2. 高危命令审批
默认开启高危命令审批机制（通过 `HERMES_EXEC_ASK=1` 控制），执行终端命令、文件修改、系统操作等危险操作前，会向用户发送确认请求，用户确认后才会执行。

### 3. 会话隔离
不同用户、不同平台的会话完全隔离，上下文、记忆、工具执行环境互不干扰，确保数据安全。

### 4. 凭据隔离
平台的登录凭据、API 密钥仅在当前 Profile 内生效，不会跨 Profile 泄露，支持多环境独立部署。

### 5. 配对机制
部分平台（如 WhatsApp、Signal）需要扫码配对才能启用，防止未授权访问。

## 平台配置示例

### Telegram 配置示例
```env
# .env 文件
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
TELEGRAM_ALLOWED_USERS=123456789,987654321
TELEGRAM_ENABLED=true
```

### 企业微信配置示例
```env
# .env 文件
WECOM_CORP_ID=your_corp_id
WECOM_AGENT_ID=your_agent_id
WECOM_SECRET=your_agent_secret
WECOM_TOKEN=your_callback_token
WECOM_AES_KEY=your_callback_aes_key
WECOM_ALLOWED_USERS=user1@company.com,user2@company.com
WECOM_ENABLED=true
```

### 飞书配置示例
```env
# .env 文件
FEISHU_APP_ID=your_feishu_app_id
FEISHU_APP_SECRET=your_feishu_app_secret
FEISHU_VERIFICATION_TOKEN=your_verification_token
FEISHU_ENCRYPT_KEY=your_encrypt_key
FEISHU_ALLOWED_USERS=user1@company.com,user2@company.com
FEISHU_ENABLED=true
```

## 背景任务通知配置

网关支持后台任务完成后的通知功能，通过配置 `display.background_process_notifications` 控制通知级别：
- `all`：发送运行过程输出 + 最终完成消息（默认）
- `result`：仅发送最终完成消息
- `error`：仅在任务执行失败时发送通知
- `off`：关闭所有后台任务通知

示例配置：
```yaml
# config.yaml
display:
  background_process_notifications: result
```

## 常见问题

**Q：网关启动失败，提示缺少平台凭据？**
A：检查 `.env` 文件中对应平台的配置是否正确，确保所有必填的凭据都已配置。

**Q：用户发送消息没有响应？**
A：检查用户 ID 是否在 `ALLOWED_USERS` 白名单中，查看网关日志是否有错误信息。

**Q：如何同时启用多个平台？**
A：在 `.env` 中配置所有需要启用的平台的凭据和 `ENABLED=true`，网关会自动启动所有已启用的平台适配器。

**Q：如何更新网关配置？**
A：修改 `.env` 或 `config.yaml` 文件后，重启网关服务即可生效。
