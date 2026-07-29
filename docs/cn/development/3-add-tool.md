# 3. 新增内置工具

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 掌握内置工具的开发流程和规范，独立完成新工具的开发
> - 正确编写工具Schema、实现函数、依赖检查和注册逻辑
> - 为工具编写完整的单元测试和手动测试用例
> - 遵循工具设计的最佳实践，避免常见的安全和性能问题

本文档介绍如何为 Hermes Agent 新增内置工具，所有步骤均符合官方规范，可直接参考。

## 概述

内置工具是 Hermes Agent 能力扩展的核心，每个工具都是独立的模块，遵循统一的注册规范，无需修改核心代码即可动态加载。
**为什么要遵循这套规范？**
- 统一的注册机制保证工具可以自动被系统发现和加载
- 统一的Schema格式兼容所有主流大模型的工具调用协议
- 内置的权限控制、错误处理、依赖检查机制不需要重复实现
- 标准化的接口保证不同工具的用户体验一致，降低用户不需要学习不同的使用方式

新增工具需要完成：工具实现、注册、测试三个步骤。

## 开发流程

### 1. 确定工具分类
首先确定你的工具属于哪个现有工具集，或者需要新增工具集：
- `core`：核心能力工具（会话管理、记忆、待办等）
- `file`：文件操作工具
- `terminal`：终端执行工具
- `web`：网络搜索、网页抓取工具
- `browser`：浏览器自动化工具
- `code`：代码执行、评审、测试工具
- `mcp`：MCP 协议工具
- `delegate`：子代理、多代理协作工具
- `media`：音视频、图像处理工具
- `iot`：智能家居、物联网工具

如果不属于现有分类，可以新增工具集。
> **为什么要分类？** 工具集是按功能分组的，用户可以根据场景按需启用/禁用不同工具集，合理分类方便用户管理权限，也避免无关工具占用Token上下文。

### 2. 创建工具文件
在 `tools/` 目录下创建工具实现文件，命名为 `<工具名>.py`，例如 `weather.py`。
> **为什么每个工具单独文件？** 遵循单一职责原则，每个工具独立成文件，方便维护、测试和按需加载，避免单个文件过大难以维护。

### 3. 实现工具代码
按照统一的代码结构实现工具功能，包含：工具函数、Schema 定义、可用性检查函数、注册逻辑。
> **为什么要统一结构？** 统一的代码结构让其他开发者可以快速理解你的代码，降低维护成本，同时系统可以自动提取元信息做后续处理。

### 4. 注册工具
在 `model_tools.py` 中添加工具导入，让系统能够自动发现并注册工具。
> **自动注册原理？** 系统启动时会导入 `_modules` 列表中的所有模块，工具的注册逻辑在模块加载时自动执行，不需要修改其他核心代码。

### 5. 编写测试
为工具编写单元测试和集成测试，确保功能正常。
> **测试要求？** 所有新工具必须包含单元测试，覆盖率不低于80%，覆盖正常、异常、边界场景，避免后续变更导致功能损坏。

### 6. 提交PR
按照贡献指南提交PR，等待审核合并。

## 完整示例代码

以下是新增一个查询天气的工具的完整示例：

### 1. 创建工具文件 `tools/weather.py`
```python
"""weather — 查询指定城市的实时天气。"""
import json
import os
from typing import Optional
import requests
from tools.registry import registry

def get_weather(city: str, unit: str = "celsius", **kwargs) -> str:
    """
    查询指定城市的实时天气
    :param city: 城市名称，支持中英文
    :param unit: 温度单位，可选值 celsius（摄氏度）/ fahrenheit（华氏度），默认celsius
    :return: JSON格式的天气信息
    """
    api_key = os.getenv("WEATHER_API_KEY")
    if not api_key:
        return json.dumps({
            "success": False,
            "error": "WEATHER_API_KEY 环境变量未配置，请先配置天气API密钥"
        })
    
    try:
        # 调用天气API
        url = f"https://api.weatherapi.com/v1/current.json?key={api_key}&q={city}&aqi=no"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
        
        # 格式化返回结果
        result = {
            "success": True,
            "city": data["location"]["name"],
            "country": data["location"]["country"],
            "temperature": data["current"]["temp_c"] if unit == "celsius" else data["current"]["temp_f"],
            "unit": unit,
            "condition": data["current"]["condition"]["text"],
            "humidity": data["current"]["humidity"],
            "wind_kph": data["current"]["wind_kph"]
        }
        return json.dumps(result, ensure_ascii=False)
    
    except requests.exceptions.RequestException as e:
        return json.dumps({
            "success": False,
            "error": f"查询天气失败: {str(e)}"
        })

# 工具OpenAI格式Schema定义
WEATHER_TOOL_SCHEMA = {
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询指定城市的实时天气信息，包括温度、湿度、天气状况、风力等",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "要查询的城市名称，支持中英文，例如：北京、Shanghai、New York"
                },
                "unit": {
                    "type": "string",
                    "description": "温度单位，可选值：celsius（摄氏度）、fahrenheit（华氏度），默认celsius",
                    "enum": ["celsius", "fahrenheit"],
                    "default": "celsius"
                }
            },
            "required": ["city"]
        }
    }
}

def _check_requirements() -> bool:
    """
    检查工具依赖是否满足
    :return: True表示可用，False表示不可用（会自动隐藏工具）
    """
    # 检查requests库是否安装
    try:
        import requests
    except ImportError:
        return False
    # 不需要强制要求配置API密钥，运行时再提示用户配置
    return True

# 注册工具到注册中心
registry.register(
    name="get_weather",
    toolset="web", # 归属到web工具集
    schema=WEATHER_TOOL_SCHEMA,
    handler=lambda args, **kw: get_weather(**args, **kw),
    check_fn=_check_requirements,
    requires_env=["WEATHER_API_KEY"] # 依赖的环境变量，用户配置时会提示
)
```

### 2. 注册工具
编辑 `model_tools.py` 文件，在 `_modules` 列表中添加工具导入：
```python
_modules = [
    # ... 现有工具 ...
    "tools.weather", # 新增的天气工具
]
```

### 3. （可选）新增工具集
如果你的工具属于新的工具集，需要在 `toolsets.py` 中添加工具集定义：
```python
# 新增工具集
WEATHER_TOOLS = Toolset(
    name="weather",
    description="天气查询工具集",
    tools=[
        "get_weather",
    ]
)

# 添加到默认启用的工具集（可选）
DEFAULT_TOOLSETS = [
    # ... 现有工具集 ...
    WEATHER_TOOLS,
]
```

## 测试方法

### 1. 单元测试
在 `tests/tools/test_weather.py` 中编写单元测试：
```python
import json
from unittest.mock import patch, Mock
from tools.weather import get_weather

def test_get_weather_success():
    """测试天气查询成功场景"""
    mock_response = Mock()
    mock_response.json.return_value = {
        "location": {"name": "北京", "country": "中国"},
        "current": {"temp_c": 25, "temp_f": 77, "condition": {"text": "晴天"}, "humidity": 40, "wind_kph": 10}
    }
    mock_response.raise_for_status.return_value = None
    
    with patch("requests.get", return_value=mock_response):
        with patch.dict("os.environ", {"WEATHER_API_KEY": "test_key"}):
            result = get_weather("北京")
            data = json.loads(result)
            assert data["success"] == True
            assert data["city"] == "北京"
            assert data["temperature"] == 25
            assert data["unit"] == "celsius"

def test_get_weather_missing_api_key():
    """测试未配置API密钥场景"""
    with patch.dict("os.environ", {}, clear=True):
        result = get_weather("北京")
        data = json.loads(result)
        assert data["success"] == False
        assert "WEATHER_API_KEY 环境变量未配置" in data["error"]
```

运行测试：
```bash
pytest tests/tools/test_weather.py -v
```

### 2. 手动测试
配置 API 密钥到 `~/.hermes/.env`：
```bash
echo 'WEATHER_API_KEY=你的实际API密钥' >> ~/.hermes/.env
```

启动 Hermes CLI 测试工具调用：
```bash
hermes chat -q "查询北京今天的天气"
```
确认 Agent 正确调用 `get_weather` 工具并返回天气信息。

## 最佳实践

### 1. 工具设计原则
- **单一职责**：一个工具只做一件事，功能明确
- **幂等性**：重复调用工具不会产生副作用
- **容错性**：工具应该处理所有可能的异常，返回友好的错误信息
- **无状态**：工具不应该保存状态，所有状态都通过参数传递

### 2. Schema 编写规范
- 描述清晰准确，说明工具的功能、使用场景、参数含义
- 必填参数明确，可选参数提供默认值
- 参数类型正确，枚举值完整
- 避免模糊描述，比如"查询信息"，应该说明查询什么信息、返回什么内容

### 3. 安全规范
- 所有用户输入都需要校验和过滤，防止注入攻击
- 敏感操作需要用户确认，或者通过权限控制
- 不要在返回结果中泄露敏感信息（API密钥、密码等）
- 网络请求需要设置超时时间，避免长时间阻塞

### 4. 性能规范
- 工具执行时间尽量控制在10秒以内，超过的应该支持后台执行
- 大结果应该分页或者截断，避免返回过多内容占用Token
- 缓存重复请求的结果，避免重复调用外部API

### 5. 文档规范
- 工具文件开头添加模块注释，说明工具功能
- 工具函数添加 docstring，说明参数含义、返回值格式
- 复杂的逻辑添加注释说明设计思路
- 工具功能变更时同步更新 Schema 描述

---

## ✍️ 练习
1. 按照本文档的步骤，实现一个查询IP地址归属地的工具，功能是输入IP地址返回归属地、运营商等信息
2. 为该工具编写完整的单元测试，覆盖正常、异常（无效IP、网络错误）场景
3. 手动测试工具调用，确认Agent可以正确识别并调用该工具返回正确结果
4. 尝试新增一个自定义工具集，将你的工具归属到该工具集下，验证可以正常启用/禁用

---

## ❓ 常见问题
### Q：新增的工具没有出现在可用工具列表中怎么办？
A：检查以下几点：
1. 工具文件是否已经添加到 `model_tools.py` 的 `_modules` 列表中
2. 工具的 `_check_requirements()` 函数是否返回True，如果依赖不满足会自动隐藏
3. 工具所属的工具集是否已经在配置中启用
4. 重启Hermes Agent，工具会在启动时重新加载

### Q：工具调用时提示参数错误怎么办？
A：检查Schema定义是否正确：
1. 参数类型是否正确，枚举值是否完整
2. 必填参数是否在 `required` 列表中
3. 参数描述是否清晰，大模型能正确理解参数含义
4. 工具函数的参数是否和Schema定义的参数完全一致

### Q：工具返回的结果大模型无法正确解析怎么办？
A：确保工具返回的是标准JSON字符串：
1. 不要返回纯文本或者其他格式，必须是合法的JSON
2. 结果不要包含特殊字符或者转义错误
3. 错误场景也要返回包含`success: False`和`error`字段的JSON
4. 大结果可以适当压缩，避免超过Token限制

### Q：工具需要依赖第三方库怎么办？
A：处理方式：
1. 在 `_check_requirements()` 函数中检查依赖是否安装，缺失时返回False，工具会自动隐藏
2. 在 `pyproject.toml` 中添加可选依赖，说明安装方式
3. 在工具的错误提示中告诉用户需要安装什么依赖

---

## 📖 导航
- 上一篇：[贡献指南](./2-contribution.md)
- 下一篇：[新增技能指南](./4-add-skill.md)
- [返回目录](../SUMMARY.md)
