# 3. 命令参考

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 掌握所有 Hermes Agent 命令的用法
> - 熟练使用命令行工具和交互式命令
> - 通过命令完成各种操作

---

## 📖 导航
- 上一篇：[API 参考](./2-api-reference.md)
- 下一篇：[工具参数参考](./4-tool-params-reference.md)
- [返回目录](../SUMMARY.md)

---

## 概述

Hermes Agent 提供两类命令：
1. **终端子命令：在终端直接执行的 `hermes xxx` 命令，用于安装、配置、启动服务等操作
2. **交互式斜杠命令：在 CLI 交互模式下使用的 `/xxx` 命令，用于会话管理、功能切换等操作

---

## 终端子命令

### `hermes` - 启动交互式 CLI
**用法**：
```bash
hermes [选项]
```

**选项**：
| 选项 | 说明 |
|------|------|
| `--config <路径` | 指定配置文件路径，默认 `~/.hermes/config.yaml |
| `--profile <名称>` | 指定使用的配置集（Profile） |
| `--debug` | 启用调试模式，输出详细日志 |
| `--version` / `-v` | 显示版本号 |
| `--help` / `-h` | 显示帮助信息 |

**示例**：
```bash
hermes --profile work --debug
```

---

### `hermes run` - 运行单次请求
**用法**：
```bash
hermes run <查询内容> [选项]
```

**选项**：
| 选项 | 说明 |
|------|------|
| `--model <模型名>` | 指定使用的模型 |
| `--stream` / `--no-stream` | 是否启用流式输出，默认启用 |
| `--output <格式>` | 输出格式：text/json，默认 text |

**示例**：
```bash
hermes run "帮我写一个 Python 快速排序" --model anthropic/claude-3-opus
hermes run "列出当前目录文件" --output json
```

---

### `hermes setup` - 运行配置向导
**用法**：
```bash
hermes setup
```

功能：交互式引导你完成基础配置，包括模型选择、API 密钥配置、皮肤选择、工具和技能启用等。

---

### `hermes gateway` - 启动网关服务
**用法**：
```bash
hermes gateway [选项]
```

**选项**：
| 选项 | 说明 |
|------|------|
| `--host <地址>` | 监听地址，默认 0.0.0.0 |
| `--port <端口>` | 监听端口，默认 8080 |
| `--daemon` | 后台守护进程运行 |

**示例**：
```bash
hermes gateway --port 8000 --daemon
```

---

### `hermes skills` - 技能管理
**子命令**：
| 命令 | 说明 |
|------|------|
| `hermes skills list` | 列出已安装的技能 |
| `hermes skills install <源地址>` | 安装技能，支持本地路径或 Git 仓库地址 |
| `hermes skills uninstall <技能名>` | 卸载指定技能 |
| `hermes skills update [技能名]` | 更新指定技能，不指定名称则更新所有技能 |
| `hermes skills search <关键词>` | 在技能市场搜索技能 |
| `hermes skills enable <技能名>` | 启用指定技能 |
| `hermes skills disable <技能名>` | 禁用指定技能 |

**示例**：
```bash
hermes skills install https://github.com/hermes-agent/skill-alpha-loop.git
hermes skills update alpha-loop
hermes skills search "文档"
```

---

### `hermes tools` - 工具管理
**子命令**：
| 命令 | 说明 |
|------|------|
| `hermes tools list` | 列出所有可用工具 |
| `hermes tools enable <工具名>` | 启用指定工具 |
| `hermes tools disable <工具名>` | 禁用指定工具 |
| `hermes tools call <工具名> [参数]` | 直接调用工具 |

**示例**：
```bash
hermes tools call list_dir path=/home/user
hermes tools disable terminal
```

---

### `hermes session` - 会话管理
**子命令**：
| 命令 | 说明 |
|------|------|
| `hermes session list` | 列出所有会话 |
| `hermes session show <会话ID>` | 查看指定会话的详细内容 |
| `hermes session delete <会话ID>` | 删除指定会话 |
| `hermes session clear` | 清空所有会话 |
| `hermes session export <会话ID> <输出路径>` | 导出会话到文件 |

**示例**：
```bash
hermes session export sess_xxxxxx ./session.md
hermes session clear
```

---

### `hermes config` - 配置管理
**子命令**：
| 命令 | 说明 |
|------|------|
| `hermes config show` | 显示当前配置 |
| `hermes config get <配置路径>` | 获取指定配置项的值 |
| `hermes config set <配置路径> <值>` | 设置指定配置项的值 |
| `hermes config reset <配置路径>` | 重置指定配置项为默认值 |

**示例**：
```bash
hermes config get model.default
hermes config set display.skin ares
```

---

### `hermes profile` - 配置集管理
**子命令**：
| 命令 | 说明 |
|------|------|
| `hermes profile list` | 列出所有配置集 |
| `hermes profile create <名称>` | 创建新的配置集 |
| `hermes profile use <名称>` | 切换到指定配置集 |
| `hermes profile delete <名称>` | 删除指定配置集 |
| `hermes profile copy <源名称> <目标名称>` | 复制配置集 |

**示例**：
```bash
hermes profile create work
hermes profile use work
```

---

### `hermes batch` - 批量处理
**用法**：
```bash
hermes batch <输入文件> [选项]
```

功能：批量处理文件内容，支持批量提问、批量处理文档等。

**选项**：
| 选项 | 说明 |
|------|------|
| `--prompt <提示词>` | 处理提示词 |
| `--output <目录>` | 输出目录 |
| `--concurrency <数量>` | 并发处理并发数，默认 5 |

**示例**：
```bash
hermes batch ./docs/*.md --prompt "帮我优化这篇文档的中文表达" --output ./output
```

---

### `hermes test` - 运行测试
**用法**：
```bash
hermes test [选项]
```

功能：运行系统自检，检查配置是否正确、API 密钥是否有效、工具是否可用等。

---

### `hermes update` - 升级版本
**用法**：
```bash
hermes update
```

功能：检查并升级到最新版本。

---

## 交互式斜杠命令

在 CLI 交互模式下输入以下命令，不需要退出界面就可以执行操作。

### 基础命令
| 命令 | 说明 | 示例 |
|------|------|------|
| `/help` | 显示所有命令帮助 | `/help` |
| `/exit` / `/quit` | 退出 CLI | `/exit` |
| `/clear` | 清空当前会话上下文 | `/clear` |
| `/history` | 查看当前会话历史 | `/history` |

---

### 模型与配置
| 命令 | 说明 | 示例 |
|------|------|------|
| `/model <模型名>` | 切换当前会话使用的模型 | `/model anthropic/claude-3-5-sonnet` |
| `/temperature <值>` | 设置当前会话的温度参数 | `/temperature 0.9` |
| `/skin <皮肤名>` | 切换界面皮肤 | `/skin ares` |
| `/config get <路径>` | 查看指定配置项 | `/config get model.default` |
| `/config set <路径> <值>` | 修改当前会话配置（临时生效） | `/config set show_tool_calls false` |

---

### 会话管理
| 命令 | 说明 | 示例 |
|------|------|------|
| `/session new [标题]` | 创建新会话 | `/session new 代码开发` |
| `/session list` | 列出所有会话 | `/session list` |
| `/session switch <会话ID>` | 切换到指定会话 | `/session switch sess_xxxxxx` |
| `/session rename <新标题>` | 重命名当前会话 | `/session rename 项目文档编写` |
| `/session delete [会话ID]` | 删除指定会话，不指定则删除当前会话 | `/session delete` |
| `/session export <输出路径>` | 导出当前会话到文件 | `/session export ./session.md` |

---

### 工具与技能
| 命令 | 说明 | 示例 |
|------|------|------|
| `/tools list` | 列出当前可用工具 | `/tools list` |
| `/tools call <工具名> [参数]` | 直接调用工具 | `/tools call web_search query="Python 教程" |
| `/skills list` | 列出已安装技能 | `/skills list` |
| `/skills use <技能名> [参数]` | 直接调用技能 | `/skills use alpha-loop prompt="开发一个 Todo 应用" |

---

### 其他命令
| 命令 | 说明 | 示例 |
|------|------|------|
| `/save <路径>` | 保存最后一次回复内容到文件 | `/save ./code.py` |
| `/copy` | 复制最后一次回复内容到剪贴板 | `/copy` |
| `/debug` | 切换调试模式 | `/debug` |
| `/retry` | 重新生成最后一次回复 | `/retry` |
| `/undo` | 撤销上一次对话，回到之前的状态 | `/undo` |

---

## 命令返回码说明

| 返回码 | 说明 |
|--------|------|
| 0 | 执行成功 |
| 1 | 通用错误 |
| 2 | 配置错误 |
| 3 | API 密钥错误 |
| 4 | 模型 API 调用失败 |
| 5 | 工具调用失败 |
| 6 | 权限不足 |
| 130 | 用户中断（Ctrl+C） |

---

## ✍️ 练习
1. 练习使用 `hermes run` 命令执行单次请求
2. 练习使用斜杠命令创建新会话、切换模型、导出会话
3. 使用 `hermes skills` 命令安装一个技能并使用

---

## ❓ 常见问题
### Q：斜杠命令支持自动补全吗？
A：支持，输入 `/` 后按 Tab 键可以自动补全命令和参数。

### Q：可以自定义命令别名吗？
A：可以，在配置文件的 `aliases` 项中配置自定义别名。

### Q：命令执行的历史记录保存在哪里？
A：保存在 `~/.hermes/history` 文件中，可以通过上下箭头调用历史命令。

---

## 📖 导航
- 上一篇：[API 参考](./2-api-reference.md)
- 下一篇：[工具参数参考](./4-tool-params-reference.md)
- [返回目录](../SUMMARY.md)