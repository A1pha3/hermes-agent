# 1. 开发环境搭建

本文档介绍如何搭建 Hermes Agent 的本地开发环境，所有步骤均经过官方验证，可直接执行。

## 依赖版本要求

在开始搭建环境之前，请确保你的系统满足以下依赖要求：

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| Git | 支持 `--recurse-submodules` 参数 | 克隆仓库必备，建议安装最新版本 |
| Python | 3.11+ | uv 可自动安装缺失的 Python 版本 |
| uv | 最新版本 | 官方推荐的 Python 包管理器，替代 pip 和 venv |
| Node.js | 18+ | 可选，浏览器工具和 WhatsApp 桥接功能需要 |
| 操作系统 | macOS 12+ / Linux (Kernel 4.15+) / Windows 10+ (WSL2) | 原生 Windows 仅支持 WSL2 环境 |

## 安装步骤

### 1. 克隆仓库
```bash
# 克隆主仓库和所有子模块
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
```

如果已经克隆过仓库但没有拉取子模块，可以执行：
```bash
git submodule update --init --recursive
```

### 2. 创建虚拟环境
使用 uv 创建独立的 Python 虚拟环境：
```bash
# 创建 Python 3.11 的虚拟环境
uv venv venv --python 3.11

# 激活虚拟环境（后续所有命令都需要先激活）
source venv/bin/activate
```

> Windows WSL2 环境下激活命令相同，原生 PowerShell 请使用 `venv\Scripts\Activate.ps1`

### 3. 安装所有依赖
```bash
# 安装核心依赖、开发依赖和所有可选功能依赖
uv pip install -e ".[all,dev]"
```

可选组件安装（根据需要选择）：
```bash
# 安装浏览器工具依赖（需要 Node.js 18+）
npm install

# 安装 RL 训练子模块（用于 Atropos 环境开发）
git submodule update --init tinker-atropos && uv pip install -e "./tinker-atropos"
```

### 4. 开发环境配置
```bash
# 创建配置目录结构
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}

# 复制示例配置文件
cp cli-config.yaml.example ~/.hermes/config.yaml

# 创建 .env 文件存储 API 密钥
touch ~/.hermes/.env
```

编辑 `~/.hermes/.env` 文件，添加至少一个 LLM 提供商的 API 密钥：
```bash
# OpenRouter（推荐，支持多模型）
OPENROUTER_API_KEY=sk-or-v1-你的密钥

# Anthropic
ANTHROPIC_API_KEY=sk-ant-你的密钥

# OpenAI
OPENAI_API_KEY=sk-你的密钥
```

## 验证安装

执行以下命令验证环境是否搭建成功：

### 1. 添加全局命令软链接
```bash
mkdir -p ~/.local/bin
ln -sf "$(pwd)/venv/bin/hermes" ~/.local/bin/hermes
```
确保 `~/.local/bin` 在你的 `PATH` 环境变量中。

### 2. 运行健康检查
```bash
hermes doctor
```
如果所有检查项都显示 OK，说明环境配置正确。

### 3. 测试聊天功能
```bash
hermes chat -q "Hello, Hermes!"
```
如果能正常返回回复，说明环境搭建成功。

## 常见问题解决

### 1. uv 安装失败
```bash
# 官方脚本安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 2. 依赖安装冲突
```bash
# 清理缓存后重新安装
uv cache clean
rm -rf venv
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"
```

### 3. Windows WSL2 环境问题
- 确保使用 WSL2 而非 WSL1
- 建议使用 Ubuntu 22.04 或更高版本的 WSL 发行版
- 避免在 Windows 文件系统（/mnt/c/）下开发，性能会很慢，建议在 WSL 内部文件系统操作

### 4. Node.js 安装问题
```bash
# 使用 nvm 安装 Node.js 18
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 18
nvm use 18
```

## 开发工具推荐

### 代码编辑器
- VS Code：安装 Python、Pylance、Code Spell Checker 插件
- PyCharm：配置 Python 解释器为 venv 中的 Python

### 调试工具
- `pdb`：Python 内置调试器
- `debugpy`：VS Code 远程调试支持
- `hermes --debug`：启用调试模式，输出详细日志

### 性能分析工具
- `py-spy`：Python 采样分析器
- `memory-profiler`：内存使用分析
- `snakeviz`：性能分析结果可视化

## 下一步
环境搭建完成后，你可以：
- 阅读 [代码规范和贡献指南](./2-contribution.md) 了解开发规范
- 尝试 [新增内置工具](./3-add-tool.md) 扩展功能
- 运行测试用例验证你的修改：`pytest tests/ -v`
