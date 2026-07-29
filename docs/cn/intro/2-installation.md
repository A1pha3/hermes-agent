# 2. 快速安装

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 完成 Hermes Agent 的安装和初始化配置
> - 验证安装是否成功
> - 了解安装过程中的常见问题和解决方法



## 环境要求

- Python 3.10+
- macOS/Linux 系统（Windows 支持 WSL2）
- 至少 2GB 内存
- 至少 10GB 可用磁盘空间

**为什么需要这些环境？**  
Python 3.10+ 提供了更好的类型提示和性能支持，WSL2 让 Windows 用户可以获得和 Linux 一致的体验，足够的内存和磁盘空间保障大模型上下文和工具缓存的正常运行。

## 安装方式

### 方式一：源码安装（推荐）
**适用场景**：需要修改源码、参与开发或者使用最新功能的用户。

1. 克隆仓库
```bash
git clone https://github.com/a1pha3/hermes-agent.git
cd hermes-agent
```
> 为什么推荐源码安装？可以随时拉取最新的功能更新，方便调试和修改。

2. 创建并激活虚拟环境
```bash
python -m venv venv
source venv/bin/activate  # macOS/Linux / WSL2
# venv\Scripts\activate  # 原生Windows（不推荐，建议使用WSL2）
```
> 为什么需要虚拟环境？避免和系统中的其他 Python 包产生版本冲突，保持环境隔离。

3. 安装依赖
```bash
pip install -e .
```
> `-e` 参数表示开发模式安装，修改源码后不需要重新安装即可生效。

4. 验证安装
```bash
hermes --version
```
> 输出版本号说明安装成功。

---

### 方式二：pip 安装
**适用场景**：只需要使用稳定版本，不需要修改源码的用户。

```bash
# 安装完整功能（包含浏览器自动化、MCP等所有可选依赖）
pip install hermes-agent[all]

# 安装最小版本，仅包含核心功能
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
> 如果你是通过源码安装的，卸载完成后可以直接删除克隆的仓库目录。

---

## ✍️ 练习
1. 根据你的使用场景选择合适的安装方式，完成 Hermes Agent 的安装
2. 运行 `hermes --version` 验证安装是否成功

---

## ❓ 常见问题
### Q：安装过程中出现权限错误怎么办？
A：可以在 pip 命令后面加上 `--user` 参数，或者使用 sudo 权限运行。

### Q：Python 版本不够怎么办？
A：建议使用 pyenv 安装 Python 3.10+ 版本，不会影响系统自带的 Python。

### Q：WSL2 下安装遇到路径问题怎么办？
A：建议将代码放在 WSL2 的文件系统中（/home/xxx/ 目录下），不要放在 /mnt/ 挂载的 Windows 分区中，避免性能和权限问题。

### Q：安装完成后运行 hermes 提示命令不存在怎么办？
A：检查虚拟环境是否已经激活，或者确认 Python 的 bin 目录是否在 PATH 环境变量中。

### Q：pip安装速度很慢怎么办？
A：可以使用国内镜像源加速，例如：
```bash
pip install hermes-agent -i https://pypi.tuna.tsinghua.edu.cn/simple
```

---

## 📖 导航
- 上一篇：[产品介绍](./1-product-intro.md)
- 下一篇：[首次运行](./3-first-run.md)
- [返回目录](../SUMMARY.md)
