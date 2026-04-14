# 2. 快速安装

## 环境要求

- Python 3.10+
- macOS/Linux 系统（Windows 支持 WSL2）
- 至少 2GB 内存
- 至少 10GB 可用磁盘空间

## 安装方式

### 方式一：源码安装（推荐）

1. 克隆仓库
```bash
git clone https://github.com/a1pha3/hermes-agent.git
cd hermes-agent
```

2. 创建并激活虚拟环境
```bash
python -m venv venv
source venv/bin/activate  # macOS/Linux
# venv\Scripts\activate  # Windows WSL2
```

3. 安装依赖
```bash
pip install -e .
```

4. 验证安装
```bash
hermes --version
```

### 方式二：pip 安装

```bash
pip install hermes-agent
```

## 初始化配置

安装完成后，首次运行会自动启动配置向导：
```bash
hermes
```

配置向导会引导你完成以下设置：
1. 选择默认模型
2. 配置 API 密钥
3. 选择默认皮肤
4. 配置启用的工具集
5. 配置启用的技能

如果需要重新运行配置向导，可以执行：
```bash
hermes setup
```

## 验证安装成功

运行以下命令，确认可以正常启动 CLI：
```bash
hermes
```

如果看到 Hermes Agent 的欢迎界面，说明安装成功。

## 卸载

如需卸载 Hermes Agent：
```bash
pip uninstall hermes-agent
```

默认配置和数据存储在 `~/.hermes/` 目录，如需完全清理：
```bash
rm -rf ~/.hermes/
```
