# 2. 代码规范和贡献指南

本文档是 Hermes Agent 开发必须遵循的规范，所有贡献的代码都需要符合这些要求才能被合并。

## 代码风格

### 基本规范
1. 遵循 **PEP 8** 规范，但不强制严格的行长度限制，建议行长度不超过 120 字符
2. 使用 4 空格缩进，禁止使用 Tab
3. 文件编码统一使用 UTF-8
4. 所有 Python 文件顶部添加必要的 `__future__` 导入（如果需要兼容旧版本）

### 注释规范
- 注释仅用于解释非明显的意图、权衡或 API 特性
- 禁止编写描述代码行为的冗余注释，代码本身应该是自解释的
- 公共 API 必须添加 docstring，使用 Google 风格 docstring 格式
- 复杂的算法实现需要添加注释说明设计思路和参考资料

```python
# 好的注释：解释为什么这么做
# 使用WAL模式提升并发写入性能，参考https://sqlite.org/wal.html
db.execute("PRAGMA journal_mode = WAL;")

# 坏的注释：描述代码做什么，冗余
# 执行PRAGMA命令设置journal_mode为WAL
db.execute("PRAGMA journal_mode = WAL;")
```

### 错误处理规范
- 捕获明确的异常类型，禁止使用裸 `except:` 捕获所有异常
- 使用 `logger.warning()`/`logger.error()` 记录错误日志
- 意外错误需要添加 `exc_info=True` 参数输出完整栈轨迹
- 错误信息应该包含足够的上下文信息，方便定位问题

```python
# 好的错误处理
try:
    result = execute_command(command)
except PermissionError as e:
    logger.error(f"执行命令{command}权限不足: {e}", exc_info=True)
    return json.dumps({"error": f"权限不足: {e}"})

# 坏的错误处理
try:
    result = execute_command(command)
except:
    logger.error("执行失败")
    return "错误"
```

### 路径处理规范
- 始终使用 `pathlib.Path` 而非字符串拼接路径
- 路径访问控制前必须使用 `os.path.realpath()` 解析符号链接，防止路径遍历攻击
- 跨平台路径处理使用 `pathlib` 的自动适配能力，不要硬编码 `/` 或 `\` 路径分隔符

```python
# 好的路径处理
from pathlib import Path
file_path = Path.home() / ".hermes" / "config.yaml"

# 坏的路径处理
file_path = "~/.hermes/config.yaml" # 不跨平台，不会自动展开~
```

## 提交规范

所有 Git 提交必须遵循 **Conventional Commits** 规范：
```
<type>(<scope>): <description>
```

### 类型说明
| 类型 | 用途 |
|------|------|
| `fix` | 修复 bug |
| `feat` | 新增功能 |
| `docs` | 文档更新 |
| `test` | 测试新增或更新 |
| `refactor` | 代码重构（无行为变更） |
| `chore` | 构建、CI、依赖更新等事务性变更 |

### 范围说明
范围表示变更影响的模块，常见值：
- `cli`：CLI 模块
- `gateway`：网关模块
- `tools`：工具系统
- `skills`：技能系统
- `agent`：核心 Agent 模块
- `security`：安全相关变更
- `docs`：文档相关变更

### 示例
```
fix(cli): 修复 save_config_value 在 model 为字符串时的崩溃问题
feat(gateway): 新增 WhatsApp 多用户会话隔离
docs(zh-cn): 添加开发环境搭建中文文档
test(tools): 补充 terminal_exec 工具的异常场景测试
```

### 提交信息要求
- 描述简洁明了，不超过 72 字符
- 使用祈使句，开头动词用原形（比如"修复"而不是"修复了"）
- 不需要添加句号结尾
- 重大变更需要在提交信息 body 中说明变更内容和影响

## PR 流程

### 1. 分支创建
分支命名规范：
- `fix/description`：bug 修复分支
- `feat/description`：新功能分支
- `docs/description`：文档分支
- `test/description`：测试分支
- `refactor/description`：重构分支

示例：
```bash
git checkout -b fix/cli-config-save-error
```

### 2. 提交前检查
提交代码前必须完成以下检查：
1. 运行全量测试，确保所有测试通过：
   ```bash
   pytest tests/ -v
   ```
2. 手动测试修改的代码路径，验证功能正常
3. 如果修改涉及文件IO、进程管理、终端处理，需要验证跨平台兼容性（至少测试 Linux 和 macOS）
4. 保持 PR 单一职责：一个 PR 仅包含一个逻辑变更，不混合 bug 修复、重构、新功能

### 3. PR 描述要求
PR 描述必须包含以下内容：
1. **变更内容**：清楚说明修改了什么，为什么修改
2. **测试方法**：bug 修复需要提供复现步骤，新功能需要提供使用示例
3. **测试环境**：说明测试过的操作系统和 Python 版本
4. **关联 Issue**：如果关联了 Issue，使用 `Fixes #123` 格式关联，合并 PR 时会自动关闭 Issue

示例 PR 描述：
```
## 变更内容
修复 CLI 中 save_config_value 函数在 model 参数为字符串时的崩溃问题，原因是代码中假设 model 是字典类型，直接取了 model["name"] 字段。

## 测试方法
1. 执行 `hermes config set model anthropic/claude-3-5-sonnet`
2. 确认配置成功保存，没有崩溃
3. 查看 ~/.hermes/config.yaml 确认配置正确更新

## 测试环境
- macOS 14.4, Python 3.11.8
- Ubuntu 22.04, Python 3.11.0

Fixes #456
```

## 代码审核要求

所有 PR 必须经过至少一位核心开发者审核通过才能合并。

### 安全审核
安全相关代码必须明确标注，需要额外重点审核：
- 所有用户输入拼接 Shell 命令必须使用 `shlex.quote()` 防止注入攻击
- 路径访问控制前必须解析符号链接，防止路径遍历
- 禁止在日志中输出 API 密钥、token、密码等敏感信息
- 新增的网络请求必须校验域名合法性，防止 SSRF 攻击

### 兼容性审核
- 跨平台代码必须提供全平台兼容方案，不允许出现仅支持单一平台的代码
- API 变更必须保持向后兼容，必须修改 API 时需要提供过渡期和迁移指南
- 配置变更必须支持自动迁移，不允许破坏现有用户的配置

### 质量审核
- 新增功能必须包含配套的单元测试和集成测试
- 测试覆盖率不能低于变更前的水平
- 代码逻辑清晰，没有不必要的复杂度
- 没有重复代码，通用逻辑需要抽象到公共模块

## 跨平台兼容规范

Hermes Agent 支持 Linux、macOS、Windows（WSL2）三个平台，所有代码必须考虑跨平台兼容性。

### 平台判断
使用 `platform.system()` 判断操作系统：
```python
import platform
system = platform.system()
if system == "Darwin":
    # macOS 特定逻辑
elif system == "Linux":
    # Linux 特定逻辑
elif system == "Windows":
    # Windows 特定逻辑
```

### 常见兼容问题处理
1. **终端 API**：Unix 专属 API（如 `termios`/`fcntl`）必须捕获 `ImportError` 并提供 Windows 回退方案
2. **文件编码**：加载文件时处理 `UnicodeDecodeError`，提供 `latin-1` 编码回退
3. **进程管理**：Windows 下不支持 `os.setsid()` 等 API，需要判断平台后分别处理
4. **路径展开**：使用 `Path.expanduser()` 展开 `~` 路径，不要手动拼接
5. **换行符**：处理文本时使用 `\n`，输出到文件时让 Python 自动处理换行符转换

## 贡献者 License 协议
所有贡献的代码默认遵循项目的 MIT 许可证，提交代码即表示你同意将你的代码贡献到该项目下，遵循 MIT 协议开源。
