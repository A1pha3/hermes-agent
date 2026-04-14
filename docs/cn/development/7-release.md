# 7. 发布流程

本文档介绍 Hermes Agent 的版本发布流程，仅核心维护者可以执行发布操作，贡献者不需要关注。

## 概述

Hermes Agent 采用固定的发布周期：
- 次版本：每 2 个月发布一个新的次版本，包含新功能、性能优化、配置变更
- 修订版本：按需发布，用于修复 bug、安全漏洞，不包含新功能和不兼容变更
- 主版本：根据 roadmap 规划，包含重大架构变更和不兼容 API 调整

发布流程经过严格的测试和验证，确保版本稳定可用。

## 版本号规则

使用语义化版本号规范：`主版本.次版本.修订号`，例如 `0.8.0`、`1.0.0`、`1.1.1`。

| 版本段 | 变更规则 |
|--------|----------|
| 主版本号 | 不兼容的 API 变更、重大架构调整、核心概念变更，用户需要修改代码或配置才能升级 |
| 次版本号 | 向下兼容的功能新增、性能优化、新工具/新技能发布、配置项新增，用户可以平滑升级 |
| 修订号 | 向下兼容的 bug 修复、安全漏洞修复、文档更新，用户可以无感知升级 |

### 预发布版本
正式发布前可以发布预发布版本用于测试，格式：`主版本.次版本.修订号-<阶段><序号>`，阶段包括：
- `alpha`：内部测试版本，功能不完整，可能有严重 bug
- `beta`：公开测试版本，功能完整，可能存在未知 bug
- `rc`：发布候选版本，功能冻结，仅修复 blocker 级别的 bug

示例：`0.8.0-alpha1`、`0.8.0-beta2`、`0.8.0-rc1`

## 发布前检查清单

发布前必须完成所有检查项，确保版本质量：

### 1. 功能检查
- [ ] 所有计划包含在该版本的功能都已经合并到 main 分支
- [ ] 所有已知的 blocker 级别的 bug 都已经修复
- [ ] 新功能都已经添加对应的文档
- [ ] 废弃的功能都已经添加废弃警告和迁移指南
- [ ] 配置变更都已经添加自动迁移逻辑

### 2. 测试检查
- [ ] 本地运行全量单元测试通过：`pytest tests/ -v`
- [ ] 集成测试全部通过：`pytest tests/integration/ -v --run-integration`
- [ ] 跨平台测试通过：在 Linux、macOS、Windows（WSL2）三个平台都完成测试
- [ ] 端到端测试通过：验证 CLI、网关、ACP 等主要场景正常工作
- [ ] 性能测试：性能没有明显下降，资源占用符合预期
- [ ] 兼容性测试：升级过程正常，现有配置和数据不会丢失

### 3. 代码检查
- [ ] 代码无编译错误和 lint 警告
- [ ] 所有安全漏洞都已经修复：依赖扫描无高危漏洞
- [ ] 没有提交敏感信息：API 密钥、密码等都没有硬编码
- [ ] 所有 PR 都已经审核通过，没有未合入的紧急变更

### 4. 文档检查
- [ ] 版本发布说明已经编写完成，包含新功能介绍、变更说明、升级指南
- [ ] 官方文档已经更新，对应新功能的使用说明齐全
- [ ] API 文档已经更新，变更的接口都有说明
- [ ] 配置项文档已经更新，新增/废弃的配置项都有说明

## 发布步骤

### 1. 准备发布分支
从 main 分支创建发布分支：
```bash
git checkout main
git pull origin main
git checkout -b release/v0.8.0
```

### 2. 更新版本号
修改 `pyproject.toml` 中的版本号：
```toml
[project]
name = "hermes-agent"
version = "0.8.0" # 修改为对应版本号
```

### 3. 编写发布说明
创建 `RELEASE_NOTES/v0.8.0.md` 文件，包含以下内容：
```markdown
# v0.8.0 发布说明

## 🎉 新功能
- 新增飞书平台适配器，支持飞书机器人
- 新增 MCP 协议支持，可以对接所有 MCP 服务
- 上下文压缩算法优化，Token 消耗降低 30%
- 新增 10+ 内置技能，覆盖开发、效率、数据等场景

## 🚀 性能优化
- LLM 调用速度提升 25%
- 启动速度提升 40%
- 内存占用降低 20%

## 🛠️ 工具更新
- 终端工具支持后台执行和完成通知
- 浏览器工具支持 Chrome 插件模式
- 文件工具支持批量操作和目录递归扫描

## 🔒 安全更新
- 修复路径遍历漏洞 CVE-2024-xxxx
- 新增敏感信息自动脱敏功能
- 增强命令执行权限控制

## ⚙️ 配置变更
- 新增 `FEISHU_*` 相关配置项，用于飞书平台
- 新增 `context.compression_threshold` 配置项，控制压缩触发阈值
- 废弃 `old_config` 配置项，将在 v0.9.0 中移除，请使用 `new_config` 替代

## 📝 升级指南
1. 升级命令：`pip install --upgrade hermes-agent`
2. 运行 `hermes config migrate` 自动迁移配置
3. 重启服务即可完成升级

## 🧪 校验和
- SHA256: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 4. 提交版本变更
```bash
git add pyproject.toml RELEASE_NOTES/v0.8.0.md
git commit -m "chore: bump version to v0.8.0"
git push origin release/v0.8.0
```

### 5. 构建发布包
```bash
# 清理旧的构建产物
rm -rf dist/ build/ *.egg-info

# 构建源码包和wheel包
python -m build

# 验证构建产物
twine check dist/*
```

### 6. 预发布测试
安装构建好的包进行测试：
```bash
pip install dist/hermes_agent-0.8.0-py3-none-any.whl

# 验证版本
hermes --version # 应该显示 v0.8.0

# 验证基本功能
hermes chat -q "Hello"
hermes doctor
```

### 7. 上传到 PyPI
```bash
# 测试环境上传（可选）
twine upload --repository testpypi dist/*

# 正式环境上传
twine upload dist/*
```

### 8. 合并发布分支
发布验证通过后，合并发布分支到 main 分支：
```bash
git checkout main
git merge --no-ff release/v0.8.0
git push origin main
```

### 9. 打版本标签
```bash
git tag -a v0.8.0 -m "Release v0.8.0"
git push origin v0.8.0
```

### 10. 发布 GitHub Release
在 GitHub 上创建新版本：
- 版本标签：`v0.8.0`
- 标题：`v0.8.0`
- 内容：复制发布说明的内容
- 上传构建产物：`dist/` 目录下的所有文件

### 11. 官方通知
- 更新官方文档站点，发布新版本文档
- 官方社区、Discord、微信群发布版本更新通知
- 相关 Issue 关闭，通知用户版本已发布

## 发布后验证

### 1. 安装验证
测试 PyPI 包可以正常安装：
```bash
pip install --upgrade hermes-agent
hermes --version
```

### 2. 功能验证
- 验证新功能正常工作
- 验证现有功能没有 regression
- 验证升级过程正常，旧版本用户可以平滑升级

### 3. 问题收集
发布后 72 小时内重点收集用户反馈，及时响应问题：
- 严重问题立即发布修订版本修复
- 普通问题记录到下个版本修复

## 补丁版本发布流程

补丁版本用于紧急修复 bug 和安全漏洞，流程相对简化：

1. 从对应版本的标签创建补丁分支：`git checkout -b hotfix/v0.8.1 v0.8.0`
2. 修复 bug，添加测试用例
3. 升级版本号到 `0.8.1`
4. 编写补丁版本发布说明，重点说明修复的问题
5. 运行全量测试，确保没有 regression
6. 构建发布包，上传到 PyPI
7. 打标签 `v0.8.1`，创建 GitHub Release
8. 合并补丁分支到 main 分支

补丁版本不包含新功能，只修复问题，尽量减少变更范围，降低风险。

## 回滚流程
如果发布后发现严重问题，需要紧急回滚：
1. 从 PyPI 删除有问题的版本
2. 通知用户不要升级，或降级到上一个稳定版本
3. 定位问题，修复后发布补丁版本
4. 发布公告说明问题和影响范围
```

## 开发篇完成
现在更新todo状态，把开发篇标记为完成：
<|FunctionCallBegin|>[{"name":"todowrite","parameters":{"todos":[{"content":"完成入门篇所有文档","status":"completed","priority":"high"},{"content":"完成使用篇所有文档","status":"completed","priority":"high"},{"content":"完成架构篇所有文档","status":"completed","priority":"medium"},{"content":"完成开发篇所有文档","status":"completed","priority":"medium"},{"content":"完成进阶篇所有文档","status":"pending","priority":"low"},{"content":"完成参考手册篇所有文档","status":"pending","priority":"low"},{"content":"整体文档校验和优化","status":"pending","priority":"high"}]}}]<|FunctionCallEnd|>
