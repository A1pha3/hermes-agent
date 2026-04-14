# 4. 新增自定义技能

技能是 Hermes Agent 扩展领域能力的核心方式，无需编写代码，通过 Markdown 格式的指令即可让 Agent 获得特定领域的专业能力。本文档介绍如何开发自定义技能。

## 概述

技能是一种纯声明式的扩展，每个技能是一个包含 `SKILL.md` 文件的目录，`SKILL.md` 包含技能的元数据和指令，Agent 加载后会自动遵循这些指令执行任务。技能无需编译、无需修改核心代码，放置到技能目录即可自动加载。

技能适合实现：
- 特定领域的工作流程（比如代码评审、项目部署、文档写作）
- 专业知识和最佳实践（比如安全审计、性能优化、架构设计）
- 重复任务的标准化流程（比如会议纪要整理、周报生成、BUG 分析）
- 第三方服务的集成使用（比如云服务操作、API 调用规范）

## 技能文件结构

### 官方技能目录结构
官方内置技能按分类存放在 `optional-skills/` 目录：
```
optional-skills/
├── development/        # 开发类技能
│   ├── code-review/    # 代码评审技能
│   ├── debug-assistant/ # 调试助手技能
│   └── deploy-helper/  # 部署助手技能
├── productivity/       # 效率类技能
│   ├── meeting-minutes/ # 会议纪要技能
│   └── email-writer/   # 邮件写作技能
└── data/               # 数据类技能
    └── report-generator/ # 报告生成技能
```

### 单个技能目录结构
每个技能是独立的目录，包含以下内容：
```
<技能名>/
├── SKILL.md              # 必须：技能主配置和指令文件
├── scripts/              # 可选：技能用到的辅助脚本
│   └── check_security.sh # 示例脚本
├── references/           # 可选：参考文档、模板、配置文件
│   └── pr_template.md    # 示例模板
└── assets/               # 可选：图片、图标等资源
    └── icon.png          # 技能图标
```

## SKILL.md 格式详解

`SKILL.md` 文件分为两部分：YAML 前置元数据和 Markdown 指令内容。

### YAML 元数据部分
位于文件开头，用 `---` 包裹，包含技能的基本信息：
```yaml
---
name: code-review          # 技能唯一标识符，小写字母+横杠
description: 自动化代码评审，检查代码质量、安全漏洞、性能问题 # 简短描述，技能搜索时展示
version: 1.0.0             # 语义化版本号
author: Hermes Team        # 作者名
license: MIT               # 开源协议
tags: [development, code-quality, security] # 标签，用于搜索分类
related_skills: [debug-assistant, deploy-helper] # 相关技能
platforms: [macos, linux]  # 可选：限制支持的操作系统，省略表示全平台支持
required_environment_variables: # 可选：依赖的环境变量，加载时提示用户配置
  - name: GITHUB_TOKEN
    prompt: GitHub 访问令牌
    help: 从 https://github.com/settings/tokens 获取，需要repo权限
    required_for: 完整功能
prerequisites:            # 可选：运行时依赖，仅作提示
  commands: [git, gh, node] # 依赖的命令行工具
metadata:
  hermes:
    requires_toolsets: [terminal, file] # 可选：仅当指定工具集可用时才展示
    fallback_for_toolsets: [web]        # 可选：仅当指定工具集不可用时才展示
---
```

### Markdown 指令部分
YAML 部分之后是技能的指令内容，通常包含以下章节：

| 章节 | 用途 | 必填 |
|------|------|------|
| # 技能名称 | 技能的展示名称和简介 | 是 |
| ## When to Use | 说明 Agent 应该何时自动使用这个技能，触发场景 | 是 |
| ## Capabilities | 技能的能力范围，能做什么不能做什么 | 是 |
| ## Procedure | 执行任务的详细步骤，Agent 会严格遵循这个流程执行 | 是 |
| ## Best Practices | 执行任务时需要遵循的最佳实践和规范 | 否 |
| ## Pitfalls | 常见陷阱和需要避免的错误 | 否 |
| ## Verification | 任务完成后如何验证结果是否正确 | 否 |
| ## Example Output | 示例输出，让 Agent 了解期望的输出格式 | 否 |

## 完整技能示例

以下是一个代码评审技能的完整 `SKILL.md` 示例：
```yaml
---
name: code-review
description: 自动化代码评审，检查代码质量、安全漏洞、性能问题、编码规范
version: 1.0.0
author: Hermes Team
license: MIT
tags: [development, code-quality, security, review]
related_skills: [debug-assistant, test-generator]
required_environment_variables:
  - name: GITHUB_TOKEN
    prompt: GitHub 访问令牌
    help: 从 https://github.com/settings/tokens 获取，需要repo权限
    required_for: 评审GitHub PR
prerequisites:
  commands: [git, gh]
metadata:
  hermes:
    requires_toolsets: [terminal, file]
---
# 代码评审助手
专业的代码评审专家，遵循行业最佳实践，提供全面、准确的代码评审意见。

## When to Use
当用户提出以下需求时自动使用本技能：
- 代码评审/代码审查/Code Review
- 检查代码质量/安全漏洞/性能问题
- 评审PR/MR/Pull Request/Merge Request
- 分析代码中的问题

## Capabilities
支持的能力：
- 代码规范检查（PEP 8/ESLint/Go Lint等）
- 安全漏洞扫描（SQL注入/XSS/认证漏洞等）
- 性能问题识别（慢查询/内存泄漏/低效算法等）
- 可维护性评估（代码复杂度/重复代码/注释完整性等）
- Bug 发现（逻辑错误/边界条件处理缺失等）
- 最佳实践建议（架构设计/API设计/错误处理等）

不支持的能力：
- 评审二进制文件或非文本格式的代码
- 评审超过10万行的超大型项目
- 修改代码或直接提交PR（仅提供评审意见）

## Procedure
执行代码评审严格遵循以下步骤：
1. **确认评审范围**：首先明确需要评审的代码范围（本地目录/文件/Git PR/提交记录）
2. **获取代码内容**：
   - 本地代码：使用 file_read 工具读取代码文件内容
   - GitHub PR：使用 gh 命令获取 PR 改动内容
   - 远程仓库：克隆仓库到临时目录后读取代码
3. **静态检查**：
   - 运行对应语言的 Lint 工具检查编码规范
   - 运行安全扫描工具检查漏洞
   - 分析代码复杂度和重复率
4. **人工（智能）评审**：
   - 逐文件分析代码逻辑
   - 检查错误处理、边界条件、并发安全
   - 评估架构设计合理性
   - 识别性能瓶颈和安全风险
5. **分类整理问题**：
   - 按严重程度分类：严重/重要/建议/优化
   - 每个问题包含：位置、问题描述、风险、修复建议
6. **生成评审报告**：按照统一格式输出评审报告
7. **验证修复（可选）**：如果用户提供了修复后的代码，重新评审验证问题是否解决

## Best Practices
评审时遵循以下最佳实践：
- 客观公正，基于事实给出意见，不带有主观偏见
- 提供具体的修复建议，而不仅仅是指出问题
- 区分必改问题和建议优化，让用户明确优先级
- 尊重现有代码风格，不要为了风格而提出无关建议
- 考虑业务场景，不要脱离实际需求提出不切实际的建议

## Pitfalls
需要避免的常见错误：
- 不要建议过度优化，优先考虑代码的可读性和可维护性
- 不要提出与项目现有规范冲突的建议
- 不要漏报安全漏洞和严重逻辑错误
- 不要误报正常的代码逻辑为问题

## Verification
评审完成后需要验证：
- 所有代码文件都已经覆盖
- 所有问题都有明确的修复建议
- 严重问题都已经标注出来
- 报告格式符合要求，清晰易读

## Example Output
### 代码评审报告
#### 项目：example-project
#### 评审范围：src/utils.py
#### 总问题数：5（严重：1，重要：2，建议：2）

| 严重程度 | 位置 | 问题描述 | 修复建议 |
|----------|------|----------|----------|
| 严重 | src/utils.py:45 | SQL注入漏洞：用户输入直接拼接到SQL语句中 | 使用参数化查询替代字符串拼接 |
| 重要 | src/utils.py:123 | 未处理空指针异常：调用user.name时没有检查user是否为None | 增加None判断，或使用可选链 |
| 建议 | src/utils.py:201 | 函数过长：超过50行，建议拆分 | 拆分为多个小函数，每个函数单一职责 |
```

## 测试方法

### 1. 安装技能
将技能目录复制到 `~/.hermes/skills/` 目录：
```bash
cp -r code-review ~/.hermes/skills/
```

### 2. 验证加载
启动 Hermes CLI，验证技能是否正常加载：
```bash
hermes --skills code-review -q "你能做什么？"
```
Agent 应该会说明它具备代码评审的能力。

### 3. 功能测试
测试技能的实际效果：
```bash
hermes --skills code-review -q "评审当前目录下的src/utils.py文件"
```
确认 Agent 严格遵循技能中定义的流程执行评审，输出符合预期格式的报告。

### 4. 边界场景测试
- 测试缺失依赖的场景：比如没有配置 `GITHUB_TOKEN` 时评审 GitHub PR，确认 Agent 会提示用户配置
- 测试不支持的场景：比如评审二进制文件，确认 Agent 会说明不支持
- 测试异常场景：比如代码文件不存在，确认 Agent 会正确处理错误

## 技能发布

### 发布到官方技能市场
如果你的技能质量高、通用性强，可以提交PR到官方 `optional-skills` 仓库，审核通过后会加入官方技能市场，所有用户都可以搜索安装。

提交PR要求：
- 技能功能完整，经过充分测试
- 文档完善，所有指令清晰明确
- 没有安全风险和恶意代码
- 符合官方技能规范
- 提供使用示例和测试用例

### 私有技能发布
对于公司内部使用的私有技能，可以搭建内部技能仓库，员工可以通过配置内部技能源来安装使用。

## 最佳实践

### 技能设计原则
- **职责单一**：一个技能专注于做好一件事，不要实现大而全的技能
- **流程明确**：执行步骤清晰可落地，Agent 能够严格遵循执行
- **边界清晰**：明确说明能做什么、不能做什么，避免幻觉
- **可验证**：有明确的完成标准，能够验证结果是否符合要求
- **通用性**：尽量减少对特定环境的依赖，让更多用户可以使用

### 指令编写规范
- 使用清晰、明确、无歧义的语言，避免模糊描述
- 使用列表、表格等结构化格式，方便 Agent 理解
- 提供示例输出，让 Agent 明确期望的结果格式
- 说明常见陷阱和错误，避免 Agent 踩坑
- 对于专业领域的技能，提供相关的标准和规范参考

### 安全规范
- 技能指令中不要包含执行危险命令的内容
- 需要执行命令时明确说明风险，需要用户确认
- 不要要求用户提供不必要的敏感信息
- 涉及第三方服务的技能需要说明数据流向，避免隐私泄露
