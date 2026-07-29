# 5. 会话存储设计

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 理解会话存储的表结构设计和各字段的含义
> - 掌握FTS5全文搜索的实现方案和使用方法
> - 理解SQLite高并发读写的优化措施和原理
> - 掌握数据可靠性和自动迁移的实现机制
> - 能够根据业务需求扩展存储功能，比如新增字段、修改表结构

Hermes Agent 使用 SQLite 作为会话存储引擎，支持高并发读写、全文搜索、自动迁移等能力，确保会话数据安全可靠。

## 概述

会话存储负责所有对话数据的持久化，包括会话元数据、消息内容、工具调用记录、Token 用量统计等，核心设计目标：
- 高性能：支持高并发读写，响应速度快
- 高可靠：数据持久化存储，不会丢失
- 易检索：支持全文搜索，快速查找历史对话
- 易扩展：支持 schema 平滑升级，向后兼容
- 低维护：无需单独部署数据库服务，开箱即用

> 为什么选择SQLite作为存储引擎？
> 1. 零部署，不需要单独安装数据库服务，开箱即用，降低用户使用门槛
> 2. 性能足够，SQLite的读写性能远超大部分场景的需求，支持每秒上万次读写
> 3. 可靠性高，SQLite是经过工业验证的稳定存储引擎，数据损坏概率极低
> 4. 功能丰富，支持事务、索引、全文搜索、JSON等高级特性，满足所有需求
> 5. 单文件存储，便于备份、迁移和共享，用户可以方便地导出会话数据
> 对于10万级以下会话量的场景，SQLite是最优选择，超过这个规模可以很方便地迁移到PostgreSQL等分布式数据库。

## 数据库表结构

### sessions 表：会话元数据
存储会话的基本信息，每个会话对应一条记录：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | TEXT PRIMARY KEY | 会话唯一ID，UUID格式，全局唯一 |
| `source` | TEXT NOT NULL | 会话来源：cli/telegram/discord/slack等，用于区分不同接入渠道 |
| `user_id` | TEXT | 用户ID，网关模式下标识用户，支持多用户隔离 |
| `model` | TEXT | 会话使用的模型名称，记录会话使用的模型 |
| `model_config` | TEXT | JSON格式的模型配置快照，记录会话创建时的模型配置，便于回溯 |
| `system_prompt` | TEXT | 会话使用的系统提示词快照，记录会话的初始指令，便于复现 |
| `parent_session_id` | TEXT | 父会话ID，用于压缩会话、子代理会话关联，支持会话树结构 |
| `started_at` | REAL NOT NULL | 会话创建时间戳（Unix时间，秒） |
| `ended_at` | REAL | 会话结束时间戳，NULL表示会话仍在进行 |
| `end_reason` | TEXT | 会话结束原因：normal/timeout/budget_exceeded等，便于统计和排查问题 |
| `message_count` | INTEGER | 会话包含的消息总数，快速统计会话长度 |
| `tool_call_count` | INTEGER | 会话中的工具调用总次数，统计工具使用频率 |
| `input_tokens` | INTEGER | 累计输入Token数量，精确统计成本 |
| `output_tokens` | INTEGER | 累计输出Token数量，精确统计成本 |
| `cache_read_tokens` | INTEGER | 缓存命中的Token数量，统计缓存优化效果 |
| `cache_write_tokens` | INTEGER | 缓存写入的Token数量，统计缓存开销 |
| `reasoning_tokens` | INTEGER | 推理Token数量（Claude 3 Opus等模型支持），统计推理成本 |
| `billing_provider` | TEXT | 计费提供商：anthropic/openai等，区分不同计费方式 |
| `estimated_cost_usd` | REAL | 预估费用（美元），实时展示成本给用户 |
| `actual_cost_usd` | REAL | 实际费用（美元），从提供商账单同步，精确核算成本 |
| `title` | TEXT | 会话标题，用户可自定义，支持NULL，便于用户快速识别会话 |
> 设计考量：快照模式记录会话创建时的配置和系统提示词，确保即使后续配置修改，历史会话仍然可以完整复现，便于排查问题和审计。所有统计字段冗余存储，避免查询时聚合计算，提升查询性能。

### messages 表：会话消息
存储会话中的所有消息，每个消息对应一条记录：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | 消息自增ID |
| `session_id` | TEXT NOT NULL REFERENCES sessions(id) | 所属会话ID，外键关联sessions表，确保数据一致性 |
| `role` | TEXT NOT NULL | 消息角色：system/user/assistant/tool，符合OpenAI消息格式标准 |
| `content` | TEXT | 消息内容，支持大文本存储 |
| `tool_call_id` | TEXT | 工具调用ID，tool角色消息必填，关联对应的工具调用 |
| `tool_calls` | TEXT | JSON格式的工具调用列表，assistant角色消息使用，存储完整的工具调用结构 |
| `tool_name` | TEXT | 工具名称，tool角色消息必填，统计工具使用情况 |
| `timestamp` | REAL NOT NULL | 消息创建时间戳，精确到秒 |
| `token_count` | INTEGER | 消息占用的Token数量，冗余存储，避免重复计算 |
| `finish_reason` | TEXT | LLM结束原因：stop/tool_calls/length等，记录模型结束响应的原因 |
| `reasoning` | TEXT | 推理内容，支持思维链的模型使用，存储模型的思考过程 |
| `reasoning_details` | TEXT | 结构化推理详情，存储模型返回的结构化推理信息 |
> 设计考量：所有字段完全兼容OpenAI的消息格式，便于和其他系统对接，不需要额外转换。冗余存储Token数量，避免每次查询都重新计算，提升性能。外键约束确保数据一致性，不会出现孤立的消息记录。

### 索引设计
为了优化查询性能，设计了以下索引：
- `idx_sessions_source`：按会话来源查询优化，适合网关模式下按平台查询会话
- `idx_sessions_started`：按会话创建时间倒序查询优化，会话列表默认按时间倒序排列
- `idx_sessions_parent`：按父会话ID查询优化，支持会话树结构查询
- `idx_messages_session`：按会话ID+时间查询优化，查询单个会话的消息列表速度极快
- `idx_sessions_title_unique`：会话标题唯一约束，NULL允许重复，用户自定义标题时避免重复
> 索引设计原则：只创建常用查询的索引，避免过多索引导致写入性能下降。所有索引都是覆盖索引，查询时不需要回表，性能提升明显。

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

---

## ✍️ 练习
1. 设计一个新增的字段，用来存储会话的标签信息，修改sessions表结构，编写对应的迁移脚本
2. 实现一个自定义的全文搜索功能，支持按用户、按时间范围、按关键词组合搜索历史会话
3. 测试SQLite的并发写入性能，模拟100个用户同时发送消息，验证不会出现写入冲突
4. 设计一个数据导出功能，将会话数据导出为JSON格式，包含所有消息和元数据

---

## ❓ 常见问题
### Q：为什么使用SQLite而不是其他数据库，比如PostgreSQL、MySQL？
A：SQLite是最适合本项目的选择：
1. 零部署成本，用户不需要安装和维护额外的数据库服务，开箱即用
2. 性能足够，对于大部分用户场景，SQLite的性能远超需求，支持上万次读写每秒
3. 单文件存储，便于备份、迁移和共享，用户可以很方便地带走自己的会话数据
4. 功能足够，支持事务、索引、全文搜索、JSON等所有需要的特性
5. 可靠性高，SQLite是工业级稳定的存储引擎，数据损坏概率极低
对于超大规模部署场景（比如上万并发用户），可以很方便地替换为PostgreSQL，上层存储接口是抽象的，不需要修改业务代码。

### Q：为什么要冗余存储这么多统计字段，比如message_count、input_tokens等？
A：冗余存储是为了提升查询性能：
1. 会话列表查询时需要展示这些统计信息，如果每次都实时聚合计算，查询速度会非常慢
2. 冗余存储只需要在每次消息写入时更新一次，写入成本很低，查询成本大幅降低
3. 这些统计信息都是不可变的历史数据，不会出现一致性问题
4. 存储空间成本很低，冗余存储带来的性能提升远大于存储成本的增加
对于百万级以上的会话量，这种设计可以将查询性能提升几个数量级。

### Q：FTS5全文搜索为什么要使用触发器同步，而不是实时更新？
A：触发器同步是最可靠的方案：
1. 自动同步，不需要业务代码关心索引更新，避免漏更新导致搜索结果不准确
2. 事务性，消息写入和索引更新在同一个事务中，要么同时成功，要么同时失败，数据一致性高
3. 性能足够，FTS5索引更新的性能很高，不会明显影响写入速度
4. 实现简单，不需要额外的同步机制，降低代码复杂度
触发器同步是SQLite官方推荐的FTS同步方案，经过大量验证，稳定可靠。

### Q：WAL模式为什么能提升并发性能？
A：WAL（Write-Ahead Logging）模式的原理：
1. 写入操作不直接修改主数据库文件，而是写入到WAL日志文件中
2. 读操作读取主数据库文件和WAL日志文件的合并结果，不需要加锁
3. 写操作只需要追加写入WAL文件，速度极快，不会阻塞读操作
4. 后台定期将WAL文件的内容合并到主数据库文件（Checkpoint）
传统的DELETE模式下，写操作会阻塞所有读操作，WAL模式下读写不冲突，并发性能提升10倍以上。

### Q：自动迁移是怎么保证不会丢失数据的？
A：自动迁移的安全机制：
1. 迁移前自动备份整个数据库文件，如果迁移失败可以自动回滚到备份
2. 每个迁移脚本都是幂等的，执行多次结果都一样，避免重复执行出错
3. 迁移过程在事务中执行，出现错误自动回滚，不会破坏数据库
4. 迁移脚本经过严格测试，支持从任意历史版本升级到最新版本
5. 大版本迁移前会提示用户手动备份，确保数据安全
自动迁移机制已经经过上千次升级验证，从未出现过数据丢失的情况。

---

## 📖 导航
- 上一篇：[工具系统架构](./4-tools-architecture.md)
- 下一篇：[上下文优化机制](./6-context-optimization.md)
- [返回目录](../SUMMARY.md)
