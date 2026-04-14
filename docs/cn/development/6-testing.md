# 6. 测试指南

本文档介绍 Hermes Agent 的测试体系、测试运行方法和测试编写规范，所有贡献的代码都需要包含配套测试。

## 概述

Hermes Agent 采用多层测试体系，确保代码质量和稳定性：
- **单元测试**：测试单个函数、类的功能，覆盖所有分支和异常场景
- **集成测试**：测试多个模块之间的交互，以及与外部服务的集成
- **端到端测试**：测试完整的用户场景，从输入到输出的全流程
- **跨平台测试**：在 Linux、macOS、Windows 多个平台上验证兼容性
- **性能测试**：验证高并发场景下的性能表现和资源占用

## 测试框架

### 核心测试工具
| 工具 | 用途 |
|------|------|
| `pytest` | 核心测试框架，所有测试都基于 pytest 编写 |
| `pytest-xdist` | 并行运行测试，提升测试速度 |
| `pytest-cov` | 测试覆盖率统计 |
| `pytest-mock` | Mock 支持，模拟外部依赖 |
| `hypothesis` | 基于属性的测试，自动生成测试用例 |

### 测试配置
测试配置定义在 `pyproject.toml` 的 `[tool.pytest.ini_options]` 部分：
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
addopts = "-v --strict-markers"
markers = [
    "integration: 标记集成测试，需要外部服务依赖",
    "e2e: 标记端到端测试，需要完整运行环境",
    "slow: 标记运行时间较长的测试",
    "windows: 仅在Windows平台运行的测试",
    "macos: 仅在macOS平台运行的测试",
    "linux: 仅在Linux平台运行的测试",
]
```

## 测试目录结构

测试代码统一存放在 `tests/` 目录下，按模块分类：
```
tests/
├── conftest.py               # pytest 全局配置和fixture
├── acp/                      # ACP 适配器测试
├── agent/                    # Agent 核心模块测试
│   ├── test_context_compressor.py
│   ├── test_prompt_builder.py
│   └── test_prompt_caching.py
├── cli/                      # CLI 命令测试
├── cron/                     # 定时任务模块测试
├── e2e/                      # 端到端测试
│   ├── test_cli_chat.py
│   └── test_gateway_flow.py
├── environments/             # 执行环境后端测试
├── fakes/                    # 测试用的Fake实现（模拟外部依赖）
│   ├── fake_llm_client.py
│   └── fake_platform_adapter.py
├── gateway/                  # 网关模块测试
│   ├── platforms/            # 各平台适配器测试
│   └── test_run.py
├── hermes_cli/               # CLI 子命令测试
│   ├── test_config.py
│   ├── test_gateway.py
│   └── test_skills.py
├── integration/              # 集成测试（需要外部服务）
│   ├── test_browser_tools.py
│   ├── test_web_tools.py
│   └── test_llm_integration.py
├── run_agent/                # AIAgent 类核心测试
├── skills/                   # 技能功能测试
└── tools/                    # 工具实现测试
    ├── test_file_tools.py
    ├── test_terminal_tools.py
    └── test_web_tools.py
```

## 运行测试的方法

### 1. 运行全量单元测试
默认运行所有不需要外部依赖的单元测试：
```bash
pytest tests/ -v
```

### 2. 并行运行测试
使用多进程并行运行测试，大幅提升速度：
```bash
pytest tests/ -n auto
```

### 3. 运行指定模块测试
只运行某个模块的测试：
```bash
# 运行工具模块测试
pytest tests/tools/ -v

# 运行单个测试文件
pytest tests/tools/test_terminal_tools.py -v

# 运行单个测试用例
pytest tests/tools/test_terminal_tools.py::test_terminal_exec_basic -v
```

### 4. 运行集成测试
集成测试需要依赖外部服务（比如 API 密钥、浏览器服务等），需要加上 `--run-integration` 参数：
```bash
# 运行所有集成测试
pytest tests/integration/ -v --run-integration

# 运行特定集成测试
pytest tests/integration/test_web_tools.py -v --run-integration
```

### 5. 运行端到端测试
端到端测试需要完整的运行环境：
```bash
pytest tests/e2e/ -v
```

### 6. 查看测试覆盖率
```bash
# 生成覆盖率报告
pytest tests/ --cov=./ --cov-report=term --cov-report=html

# 查看HTML报告
open htmlcov/index.html
```

要求核心模块的测试覆盖率不低于 80%，新增代码的测试覆盖率不低于 90%。

### 7. 运行指定标记的测试
```bash
# 仅运行慢测试
pytest tests/ -v -m slow

# 排除慢测试
pytest tests/ -v -m "not slow"

# 仅运行Linux平台测试
pytest tests/ -v -m linux
```

## 编写测试规范

### 1. 单元测试规范
- **隔离性**：测试用例之间相互独立，不共享状态，每个测试用例运行前后环境一致
- **覆盖性**：覆盖所有代码分支、正常路径和异常路径
- **原子性**：每个测试用例只测试一个功能点，避免大而全的测试
- **清晰性**：测试用例命名清晰，能够直接看出测试的功能和场景
- **可重复性**：每次运行结果一致，不依赖外部环境或随机数

示例：
```python
def test_terminal_exec_basic():
    """测试终端执行基本命令"""
    result = terminal_exec("echo hello")
    data = json.loads(result)
    assert data["success"] == True
    assert data["output"].strip() == "hello"

def test_terminal_exec_nonexistent_command():
    """测试执行不存在的命令的异常场景"""
    result = terminal_exec("nonexistent_command_1234")
    data = json.loads(result)
    assert data["success"] == False
    assert "command not found" in data["error"]
```

### 2. 集成测试规范
- 集成测试需要标记 `@pytest.mark.integration`
- 明确说明依赖的外部服务和配置要求
- 尽量使用模拟服务替代真实外部服务，提高测试稳定性
- 测试完成后清理测试产生的临时数据
- 避免依赖生产环境的资源

示例：
```python
import pytest
from tools.web import web_search

@pytest.mark.integration
@pytest.mark.requires_env("SEARCH_API_KEY")
def test_web_search_real():
    """测试真实的网络搜索功能"""
    result = web_search("Hermes Agent 开源项目")
    data = json.loads(result)
    assert data["success"] == True
    assert len(data["results"]) > 0
    assert any("Hermes" in result["title"] for result in data["results"])
```

### 3. Mock 使用规范
- 优先使用 `pytest-mock` 提供的 `mocker` fixture
- Mock 的目标是外部依赖，而不是内部函数
- Mock 的返回值要尽量模拟真实场景
- 验证 Mock 被正确调用，参数符合预期

示例：
```python
def test_web_search_mocked(mocker):
    """使用Mock测试网络搜索，不需要真实API"""
    # Mock requests.get 方法
    mock_get = mocker.patch("requests.get")
    mock_response = mocker.Mock()
    mock_response.json.return_value = {
        "results": [
            {"title": "Hermes Agent 开源项目", "url": "https://github.com/NousResearch/hermes-agent", "snippet": "AI Agent 框架"}
        ]
    }
    mock_response.raise_for_status.return_value = None
    mock_get.return_value = mock_response

    result = web_search("Hermes Agent")
    data = json.loads(result)
    assert data["success"] == True
    assert data["results"][0]["title"] == "Hermes Agent 开源项目"
    # 验证Mock被正确调用
    mock_get.assert_called_once()
```

### 4. Fixture 使用规范
- 通用的测试依赖定义在 `conftest.py` 中
- Fixture 作用域尽量小，减少不必要的初始化开销
- 临时目录使用 `tmp_path` fixture，自动清理
- 测试使用的 `HERMES_HOME` 自动重定向到临时目录，不会影响用户真实配置

示例：
```python
# conftest.py 中定义全局fixture
import pytest
import tempfile
import os
from pathlib import Path

@pytest.fixture(autouse=True)
def isolate_hermes_home(monkeypatch, tmp_path):
    """自动隔离测试用的HERMES_HOME目录，不影响用户真实配置"""
    hermes_home = tmp_path / ".hermes"
    hermes_home.mkdir()
    monkeypatch.setenv("HERMES_HOME", str(hermes_home))
    return hermes_home
```

### 5. 跨平台测试规范
- 跨平台代码需要在所有支持的平台上测试
- 使用平台标记 `@pytest.mark.linux`、`@pytest.mark.macos`、`@pytest.mark.windows` 标记平台专属测试
- 路径相关测试使用 `pathlib` 处理，避免硬编码路径分隔符
- 命令行相关测试注意不同平台的命令差异

示例：
```python
import pytest
import platform

@pytest.mark.linux
def test_linux_specific_feature():
    """仅在Linux平台运行的测试"""
    assert platform.system() == "Linux"
    # 测试Linux专属功能

@pytest.mark.windows
def test_windows_path_handling():
    """测试Windows路径处理"""
    from pathlib import Path
    path = Path("C:\\Users\\test\\.hermes")
    assert path.name == ".hermes"
```

## 测试最佳实践

### 1. 测试左移
- 开发过程中随时运行相关测试，提前发现问题
- 提交代码前必须运行相关模块的测试，确保不会破坏现有功能
- CI 会自动运行全量测试，不通过的 PR 无法合并

### 2. 测试性能
- 单个单元测试运行时间不超过 1 秒
- 全量测试运行时间不超过 5 分钟
- 慢测试需要标记 `@pytest.mark.slow`，默认 CI 可以跳过，定期运行
- 避免不必要的 IO 操作和等待

### 3. 测试可维护性
- 测试代码和生产代码同等重要，需要遵循相同的代码规范
- 避免重复代码，公共逻辑抽象为 fixture 或工具函数
- 测试用例命名清晰，见名知意
- 测试代码注释说明测试的场景和预期结果

### 4. 避免不稳定测试
- 测试不依赖网络、外部服务、时间等不稳定因素
- 避免使用固定等待时间，使用条件等待
- 随机数相关测试使用固定种子，保证可重复性
- 并发相关测试考虑线程安全，避免竞争条件

### 5. 故障注入测试
- 模拟各种异常场景，测试错误处理逻辑是否正确
- 模拟网络超时、API 错误、权限不足、磁盘满等场景
- 测试边界条件：空输入、超长输入、非法参数等
- 测试异常恢复能力，确保系统不会因为单次错误而崩溃
