# 5. 会话存储设计

Hermes Agent 使用 SQLite 作为会话存储引擎，支持高并发读写、全文搜索、自动迁移等能力，确保会话数据安全可靠。

## 概述

会话存储负责所有对话数据的持久化，包括会话元数据、消息内容、工具调用记录、Token 用量统计等，核心设计目标：
- 高性能：支持高并发读写，响应速度快
- 高可靠：数据持久化存储，不会丢失
- 易检索：支持全文搜索，快速查找历史对话
- 易扩展：支持 schema 平滑升级，向后兼容
- 低维护：无需单独部署数据库服务，开箱即用

## 数据库表结构

### sessions 表：会话元数据
存储会话的基本信息，每个会话对应一条记录：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | TEXT PRIMARY KEY | 会话唯一ID，UUID格式 |
| `source` | TEXT NOT NULL | 会话来源：cli/telegram/discord/slack等 |
| `user_id` | TEXT | 用户ID，网关模式下标识用户 |
| `model` | TEXT | 会话使用的模型名称 |
| `model_config` | TEXT | JSON格式的模型配置快照 |
| `system_prompt` | TEXT | 会话使用的系统提示词快照 |
| `parent_session_id` | TEXT | 父会话ID，用于压缩会话、子代理会话关联 |
| `started_at` | REAL NOT NULL | 会话创建时间戳（Unix时间，秒） |
| `ended_at` | REAL | 会话结束时间戳，NULL表示会话仍在进行 |
| `end_reason` | TEXT | 会话结束原因：normal/timeout/budget_exceeded等 |
| `message_count` | INTEGER | 会话包含的消息总数 |
| `tool_call_count` | INTEGER | 会话中的工具调用总次数 |
| `input_tokens` | INTEGER | 累计输入Token数量 |
| `output_tokens` | INTEGER | 累计输出Token数量 |
| `cache_read_tokens` | INTEGER | 缓存命中的Token数量 |
| `cache_write_tokens` | INTEGER | 缓存写入的Token数量 |
| `reasoning_tokens` | INTEGER | 推理Token数量（Claude 3 Opus等模型支持） |
| `billing_provider` | TEXT | 计费提供商：anthropic/openai等 |
| `estimated_cost_usd` | REAL | 预估费用（美元） |
| `actual_cost_usd` | REAL | 实际费用（美元） |
| `title` | TEXT | 会话标题，用户可自定义，支持NULL |

### messages 表：会话消息
存储会话中的所有消息，每个消息对应一条记录：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | 消息自增ID |
| `session_id` | TEXT NOT NULL REFERENCES sessions(id) | 所属会话ID，外键关联sessions表 |
| `role` | TEXT NOT NULL | 消息角色：system/user/assistant/tool |
| `content` | TEXT | 消息内容 |
| `tool_call_id` | TEXT | 工具调用ID，tool角色消息必填 |
| `tool_calls` | TEXT | JSON格式的工具调用列表，assistant角色消息使用 |
| `tool_name` | TEXT | 工具名称，tool角色消息必填 |
| `timestamp` | REAL NOT NULL | 消息创建时间戳 |
| `token_count` | INTEGER | 消息占用的Token数量 |
| `finish_reason` | TEXT | LLM结束原因：stop/tool_calls/length等 |
| `reasoning` | TEXT | 推理内容，支持思维链的模型使用 |
| `reasoning_details` | TEXT | 结构化推理详情 |

### 索引设计
为了优化查询性能，设计了以下索引：
- `idx_sessions_source`：按会话来源查询优化
- `idx_sessions_started`：按会话创建时间倒序查询优化
- `idx_sessions_parent`：按父会话ID查询优化
- `idx_messages_session`：按会话ID+时间查询优化
- `idx_sessions_title_unique`：会话标题唯一约束，NULL允许重复

## FTS5 全文搜索实现

Hermes Agent 使用 SQLite 的 FTS5（Full-Text Search 5）扩展实现全文搜索能力，支持快速搜索历史对话内容。

### 实现方案
1. **虚拟表设计**：创建 `messages_fts` 虚拟表，存储消息内容的全文索引：
   ```sql
   CREATE VIRTUAL TABLE messages_fts USING fts5(
       content,
       content='messages',
       content_rowid='id',
       tokenize='unicode61'
   );
   ```
   使用 `unicode61` 分词器，支持中文、英文、数字等多语言分词。

2. **触发器同步**：创建触发器，自动同步 `messages` 表的增删改操作到 FTS 索引：
   - `messages_after_insert`：插入消息时同步到FTS表
   - `messages_after_update`：更新消息时同步到FTS表
   - `messages_after_delete`：删除消息时同步到FTS表

3. **搜索逻辑**：
   - 支持关键词搜索、精确短语搜索、布尔逻辑搜索、前缀匹配搜索
   - 自动过滤特殊字符，防止SQL注入
   - 搜索结果返回匹配片段、上下文、会话元数据
   - 结果按相关性排序，最相关的结果排在前面

### 搜索语法示例
```sql
-- 简单关键词搜索
SELECT * FROM messages_fts WHERE messages_fts MATCH 'Python 性能优化'

-- 精确短语搜索
SELECT * FROM messages_fts WHERE messages_fts MATCH '"Hermes Agent"'

-- 布尔逻辑搜索
SELECT * FROM messages_fts WHERE messages_fts MATCH 'Python AND 优化 NOT Java'

-- 前缀匹配搜索
SELECT * FROM messages_fts WHERE messages_fts MATCH 'arch*'
```

## 会话持久化机制

### 实时写入
每个对话回合结束后，异步将会话数据写入SQLite数据库，不阻塞主流程：
- 消息内容实时写入 `messages` 表
- 会话统计信息更新到 `sessions` 表
- FTS索引自动同步更新

### 并发控制
采用 SQLite 的 WAL（Write-Ahead Logging）模式实现高并发读写：
- 读操作不会阻塞写操作，写操作也不会阻塞读操作
- 支持多进程并发读，单进程写
- 使用 `BEGIN IMMEDIATE` + 随机重试机制解决写入冲突，避免 convoy 效应
- 支持高并发场景下的稳定写入，适合网关多用户场景

### 数据可靠性
- 所有写入操作都启用同步模式，确保数据写入磁盘后才返回成功
- 定期执行 checkpoint，将WAL文件的内容合并到主数据库文件
- 支持数据库损坏自动修复，尝试从备份恢复数据
- 支持手动备份和恢复功能

### 自动迁移
启动时自动检测数据库 schema 版本，执行增量迁移：
- 每个版本的 schema 变化都有对应的迁移脚本
- 支持从任意历史版本升级到最新版本
- 迁移过程自动备份数据，失败自动回滚
- 向后兼容所有历史版本的数据库，升级不会丢失数据

## 性能优化措施

### 1. WAL 模式配置
```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA wal_autocheckpoint = 1000;
```
平衡性能和可靠性，写入性能比传统 DELETE 模式提高10倍以上。

### 2. 缓存配置
```sql
PRAGMA cache_size = -20000; -- 20MB 页缓存
PRAGMA temp_store = MEMORY; -- 临时表存储在内存中
```
提高查询性能，减少磁盘IO。

### 3. 定期清理
- 自动清理超过保留期限的历史会话，默认保留90天
- 定期清理过期的临时文件和缓存数据
- 支持手动执行 `VACUUM` 命令整理数据库文件，减少磁盘占用

### 4. 批量写入
批量处理的场景下使用批量写入机制，减少事务提交次数：
- 批量导入会话数据时使用事务批量提交
- 历史数据迁移时使用批量插入优化速度
- 每1000条记录提交一次事务，平衡性能和可靠性
