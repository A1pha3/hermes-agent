# 6. 常见问题解答

## 安装类问题

### Q：安装过程中提示依赖安装失败
**A：**
1. 确保 Python 版本 >= 3.10：`python --version`
2. 升级 pip 到最新版本：`pip install --upgrade pip`
3. 尝试使用国内镜像源安装：`pip install -e . -i https://pypi.tuna.tsinghua.edu.cn/simple`
4. 如果是特定依赖安装失败，手动安装该依赖的兼容版本：`pip install [依赖名]==[版本号]`

### Q：Windows 系统可以安装吗？
**A：** 原生 Windows 系统暂不支持，推荐使用 WSL2（Windows Subsystem for Linux）环境安装使用，和原生 Linux 体验一致。

### Q：安装完成后执行 `hermes` 提示命令不存在
**A：**
1. 检查 Python 的 bin 目录是否在 PATH 环境变量中：`echo $PATH`
2. 找到 hermes 命令的实际路径：`find ~ -name hermes`
3. 将路径添加到 PATH 中：`export PATH=$PATH:/path/to/hermes/bin`
4. 或者直接使用完整路径运行：`/path/to/hermes`

---

## 运行类问题

### Q：启动时提示 API 密钥未配置
**A：**
1. 检查 `~/.hermes/.env` 文件中是否配置了对应模型的 API 密钥
2. 如果使用 Anthropic 模型，确保配置了 `ANTHROPIC_API_KEY`
3. 如果使用 OpenAI 模型，确保配置了 `OPENAI_API_KEY`
4. 可以重新运行配置向导自动生成配置文件：`hermes setup`

### Q：启动时报错 "No module named 'xxx'"
**A：**
1. 确保已经激活了虚拟环境：`source venv/bin/activate`
2. 重新安装依赖：`pip install -e .`
3. 如果仍报错，手动安装缺失的模块：`pip install xxx`

### Q：模型响应很慢，经常超时
**A：**
1. 检查网络连接是否正常，是否可以访问模型服务商的 API 端点
2. 尝试更换网络环境，或者配置代理：在 `.env` 中添加 `HTTP_PROXY=http://your-proxy:port`
3. 调整模型超时时间：在 `config.yaml` 中设置 `model.timeout = 120`（单位秒）
4. 尝试切换到响应更快的模型，比如 `claude-3-5-sonnet` 比 `claude-opus` 响应更快

### Q：会话历史丢失了怎么办？
**A：**
1. 会话历史存储在 `~/.hermes/sessions/` 目录下的 SQLite 数据库中
2. 检查是否切换了 Profile，不同 Profile 的会话历史是隔离的
3. 可以使用 `/history` 命令查看当前会话的历史
4. 如果数据库文件损坏，可以从备份中恢复，或者使用 `session_search` 工具搜索历史内容

---

## 工具调用类问题

### Q：执行终端命令时提示需要审批
**A：**
这是默认的安全机制，防止恶意命令执行：
1. 输入 `yes` 确认执行即可
2. 如果不需要审批，可以在 `.env` 中设置 `HERMES_EXEC_ASK=0`（不推荐生产环境使用）

### Q：文件操作工具提示 "Path not allowed"
**A：**
这是路径安全限制，默认只能访问当前工作目录及子目录：
1. 在 `.env` 中配置 `HERMES_FILE_ALLOW_PATHS=/path/to/allow,/another/path` 添加允许访问的路径
2. 或者在启动时切换到要操作的目录下运行 hermes 命令

### Q：浏览器自动化工具调用失败
**A：**
1. 检查是否配置了 `BROWSERBASE_API_KEY` 凭据
2. 确保网络可以访问 Browserbase 服务
3. 如果使用本地浏览器，确保本地已经安装了 Chrome 或 Edge 浏览器，并且配置了 `BROWSER_LOCAL=true`

### Q：MCP 工具无法连接到服务器
**A：**
1. 检查 MCP 服务器是否正常运行
2. 确认 MCP 服务器的配置信息正确，地址、端口、密钥配置无误
3. 检查网络连接是否正常，是否可以访问 MCP 服务器的地址

---

## 技能使用类问题

### Q：安装技能后不生效
**A：**
1. 重启 Hermes Agent，技能会在启动时自动加载
2. 检查技能是否已经启用：`/skills list` 查看技能状态
3. 如果是自定义技能，检查 `SKILL.md` 文件格式是否正确，YAML 前置信息是否符合规范

### Q：调用技能时提示 "Skill not found"
**A：**
1. 检查技能名称是否输入正确，技能名称是大小写敏感的
2. 确认该技能已经安装：`/skills list` 查看已安装的技能列表
3. 如果是第三方技能，确保已经放到正确的技能目录下，并且已经重启 Agent

### Q：技能输出不符合预期
**A：**
1. 查看技能的使用说明：`/[技能名] --help` 了解正确的使用方式和参数
2. 检查是否提供了足够的上下文信息给技能
3. 尝试显式指定技能的参数，比如 `--style`、`--output` 等
4. 如果是自定义技能，可以修改 `SKILL.md` 中的提示词逻辑优化输出效果

### Q：技能执行速度很慢
**A：**
部分复杂技能需要多次调用工具和模型，执行时间会比较长：
1. 耐心等待， spinner 动画显示当前正在执行的步骤
2. 可以在配置中开启工具执行进度显示，实时查看执行状态
3. 如果长时间无响应，可以按 `Ctrl+C` 中断执行，重试一次

---

## 网关部署类问题

### Q：网关启动失败，提示 "Platform xxx not configured"
**A：**
1. 检查 `.env` 文件中对应平台的配置是否完整，所有必填参数都已配置
2. 确认对应平台的 `ENABLED` 变量设置为 `true`
3. 检查平台凭据是否正确，比如 Bot Token、API 密钥等是否有效

### Q：用户发送消息没有响应
**A：**
1. 检查用户 ID 是否在对应平台的 `ALLOWED_USERS` 白名单中
2. 查看网关日志，是否有错误信息：`journalctl -u hermes-gateway`（Linux 系统服务）
3. 确认模型 API 可以正常访问，模型调用没有报错
4. 检查是否是网络问题导致消息无法到达网关

### Q：网关后台运行时日志在哪里查看？
**A：**
- Linux systemd 服务：`journalctl -u hermes-gateway -f`
- macOS launchd 服务：`tail -f ~/.hermes/logs/gateway.log`
- 前台运行时日志直接输出到 stdout

### Q：如何配置网关的开机自启动？
**A：**
运行 `hermes gateway install` 命令，会自动生成系统服务配置并设置开机自启动，不需要手动配置。

---

## 性能优化类问题

### Q：Token 消耗太快，费用太高
**A：**
1. 开启自动上下文压缩：`config.yaml` 中设置 `session.auto_compress_context = true`
2. 开启提示词缓存：默认已开启，会自动缓存重复的系统提示词，降低 token 消耗
3. 切换到更便宜的模型，比如 `claude-3-haiku` 价格比 `claude-opus` 低很多
4. 调整会话历史长度：`config.yaml` 中设置 `session.max_history_length = 20`，减少历史消息的 token 占用

### Q：内存占用太高
**A：**
1. 减少最大历史消息长度，降低内存占用
2. 定期清理不需要的会话历史和缓存文件
3. 如果部署了多个网关 Profile，考虑使用容器化部署，限制每个容器的内存配额

### Q：并发响应慢
**A：**
1. 配置模型的并发请求限制：`config.yaml` 中设置 `model.max_concurrent_requests = 10`
2. 升级服务器配置，使用更高性能的 CPU 和更大的内存
3. 如果是高并发场景，考虑部署多实例负载均衡，将请求分散到多个 Agent 实例处理

---

## 其他问题

### Q：如何升级 Hermes Agent 到最新版本？
**A：**
```bash
cd /path/to/hermes-agent
git pull
source venv/bin/activate
pip install -e .
```
升级后重启所有运行中的服务即可。

### Q：如何备份数据？
**A：**
直接备份整个 `~/.hermes/` 目录即可，包含所有配置、凭据、会话历史、技能等数据。恢复时将备份的目录放回对应路径即可。

### Q：遇到未列出的问题怎么办？
**A：**
1. 查看官方文档和 GitHub Issues，是否有其他人遇到过相同的问题
2. 运行 `hermes status` 命令查看当前状态，获取诊断信息
3. 查看日志文件：`~/.hermes/logs/` 目录下的日志文件，定位错误原因
4. 在 GitHub 上提交 Issue，提供错误信息和复现步骤，寻求社区帮助
