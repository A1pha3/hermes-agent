# 2. API 参考

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 了解 Hermes Agent 提供的所有 API 接口
> - 使用 API 集成 Hermes Agent 到你的系统
> - 实现自定义客户端和扩展

---

## 📖 导航
- 上一篇：[配置项完整参考](./1-config-reference.md)
- 下一篇：[命令参考](./3-command-reference.md)
- [返回目录](../SUMMARY.md)

---

## 概述

Hermes Agent 提供 RESTful API 接口，允许你通过 HTTP 请求和智能体交互、管理会话、配置系统等。所有 API 都使用 JSON 格式，返回标准的 HTTP 状态码。

**API 基础地址**：`http://localhost:8080/api/v1`

**认证方式**：在请求头中携带 `Authorization: Bearer <your-api-key>`，API 密钥可以在配置文件中设置 `api_key` 项。

---

## 通用响应格式

所有 API 响应都遵循以下格式：

```json
{
  "success": true,
  "code": 200,
  "message": "操作成功",
  "data": {},
  "request_id": "xxxx-xxxx-xxxx-xxxx"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | boolean | 请求是否成功 |
| `code` | integer | HTTP 状态码 |
| `message` | string | 响应消息 |
| `data` | any | 响应数据，失败时为错误详情 |
| `request_id` | string | 请求唯一 ID，用于排查问题 |

---

## 会话管理 API

### 创建新会话
**接口**：`POST /sessions`

**请求体**：
```json
{
  "title": "新会话",
  "model": "anthropic/claude-3-sonnet",
  "system_prompt": "你是一个 helpful 的 AI 助手",
  "metadata": {
    "user_id": "user123",
    "platform": "web"
  }
}
```

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": {
    "session_id": "sess_xxxxxx",
    "title": "新会话",
    "model": "anthropic/claude-3-sonnet",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

---

### 获取会话列表
**接口**：`GET /sessions?page=1&page_size=20&keyword=xxx`

**查询参数**：
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `page` | integer | 否 | 页码，默认 1 |
| `page_size` | integer | 否 | 每页数量，默认 20 |
| `keyword` | string | 否 | 搜索关键词，用于搜索会话标题和内容 |

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": {
    "total": 100,
    "page": 1,
    "page_size": 20,
    "items": [
      {
        "session_id": "sess_xxxxxx",
        "title": "新会话",
        "model": "anthropic/claude-3-sonnet",
        "message_count": 5,
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### 获取会话详情
**接口**：`GET /sessions/{session_id}`

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": {
    "session_id": "sess_xxxxxx",
    "title": "新会话",
    "model": "anthropic/claude-3-sonnet",
    "system_prompt": "你是一个 helpful 的 AI 助手",
    "messages": [
      {
        "role": "user",
        "content": "你好",
        "created_at": "2024-01-01T00:00:00Z"
      },
      {
        "role": "assistant",
        "content": "你好！有什么可以帮你的？",
        "created_at": "2024-01-01T00:00:01Z"
      }
    ],
    "metadata": {},
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

---

### 删除会话
**接口**：`DELETE /sessions/{session_id}`

**响应**：
```json
{
  "success": true,
  "code": 200,
  "message": "删除成功"
}
```

---

## 消息交互 API

### 发送消息（同步）
**接口**：`POST /sessions/{session_id}/messages`

**请求体**：
```json
{
  "content": "帮我写一个 Python 快速排序",
  "stream": false,
  "model": "anthropic/claude-3-sonnet",  # 可选，覆盖会话默认模型
  "temperature": 0.7  # 可选，覆盖默认温度参数
}
```

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": {
    "message_id": "msg_xxxxxx",
    "role": "assistant",
    "content": "这是一个 Python 快速排序的实现...",
    "tool_calls": [],
    "usage": {
      "prompt_tokens": 100,
      "completion_tokens": 200,
      "total_tokens": 300
    },
    "created_at": "2024-01-01T00:00:00Z"
  }
}
```

---

### 发送消息（流式）
**接口**：`POST /sessions/{session_id}/messages`

**请求体**：
```json
{
  "content": "帮我写一个 Python 快速排序",
  "stream": true
}
```

**响应**：返回 `text/event-stream` 流式响应，每个事件格式如下：
```
data: {"type": "content", "content": "这是", "done": false}
data: {"type": "content", "content": "一个", "done": false}
data: {"type": "content", "content": "Python", "done": false}
data: {"type": "done", "content": "这是一个 Python 快速排序的实现...", "usage": {"prompt_tokens": 100, "completion_tokens": 200, "total_tokens": 300}, "done": true}
```

---

### 中断当前请求
**接口**：`POST /sessions/{session_id}/interrupt`

**响应**：
```json
{
  "success": true,
  "code": 200,
  "message": "已中断当前请求"
}
```

---

## 工具管理 API

### 获取可用工具列表
**接口**：`GET /tools`

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": [
    {
      "name": "list_dir",
      "description": "列出目录下的文件和文件夹",
      "parameters": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string",
            "description": "目录路径，默认当前目录"
          }
        }
      },
      "toolset": "file",
      "enabled": true
    }
  ]
}
```

### 直接调用工具
**接口**：`POST /tools/{tool_name}/call`

**请求体**：
```json
{
  "path": "/home/user"
}
```

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": {
    "files": ["file1.py", "file2.md", "dir1/"]
  }
}
```

---

## 技能管理 API

### 获取已安装技能列表
**接口**：`GET /skills`

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": [
    {
      "name": "alpha-loop",
      "version": "1.0.0",
      "description": "AI 自主迭代执行引擎",
      "author": "Hermes Team",
      "enabled": true,
      "trigger_phrases": ["用alpha-loop完成", "自动化迭代"]
    }
  ]
}
```

### 安装技能
**接口**：`POST /skills/install`

**请求体**：
```json
{
  "source": "https://github.com/hermes-agent/skill-alpha-loop.git"
}
```

### 卸载技能
**接口**：`POST /skills/{skill_name}/uninstall`

---

## 系统管理 API

### 获取系统状态
**接口**：`GET /system/status`

**响应**：
```json
{
  "success": true,
  "code": 200,
  "data": {
    "version": "1.0.0",
    "model": "anthropic/claude-3-sonnet",
    "enabled_tools": 15,
    "enabled_skills": 5,
    "active_sessions": 3,
    "uptime": 3600,
    "cpu_usage": 20.5,
    "memory_usage": 512.5
  }
}
```

### 获取配置
**接口**：`GET /system/config`

### 更新配置
**接口**：`PUT /system/config`

**请求体**：接收完整或部分配置项，和 `config.yaml` 格式一致。

---

## 错误状态码说明

| 状态码 | 说明 |
|--------|------|
| 200 | 请求成功 |
| 400 | 请求参数错误 |
| 401 | 未授权，API 密钥错误或缺失 |
| 403 | 权限不足，禁止访问 |
| 404 | 资源不存在 |
| 429 | 请求频率超限 |
| 500 | 服务器内部错误 |
| 503 | 服务不可用，模型 API 故障 |

---

## SDK 示例

### Python SDK 使用示例
```python
import requests

BASE_URL = "http://localhost:8080/api/v1"
API_KEY = "your-api-key"

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

# 创建会话
response = requests.post(f"{BASE_URL}/sessions", json={"title": "测试会话"}, headers=headers)
session_id = response.json()["data"]["session_id"]

# 发送消息
response = requests.post(
    f"{BASE_URL}/sessions/{session_id}/messages",
    json={"content": "你好", "stream": False},
    headers=headers
)
print(response.json()["data"]["content"])
```

---

## ✍️ 练习
1. 使用 API 创建一个新会话并发送消息
2. 开发一个简单的 Web 客户端，通过 API 和 Hermes Agent 交互

---

## ❓ 常见问题
### Q：API 支持跨域请求吗？
A：是的，默认支持 CORS 跨域请求，也可以在配置中自定义允许的域名。

### Q：可以自定义 API 路径吗？
A：可以，通过配置 `gateway.api_prefix` 项修改 API 前缀。

### Q：API 有请求频率限制吗？
A：默认没有限制，可以通过配置 `gateway.rate_limit` 项启用频率限制。

---

## 📖 导航
- 上一篇：[配置项完整参考](./1-config-reference.md)
- 下一篇：[命令参考](./3-command-reference.md)
- [返回目录](../SUMMARY.md)