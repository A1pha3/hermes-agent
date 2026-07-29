# 2. 生产环境部署指南

> **📚 学习目标**
> 阅读本文后，你将能够：
> - 根据业务规模选择合适的部署架构
> - 完成从单机到分布式的生产环境部署
> - 掌握高可用、安全、监控、备份等生产级配置
> - 排查生产环境常见问题，保障服务稳定运行

本文档介绍如何将 Hermes Agent 部署到生产环境，保障服务的高可用、高性能、安全性和可维护性。

## 部署架构选择
> 架构选型原则：优先选择满足当前业务需求的最简单架构，避免过度设计增加维护成本，同时保留未来扩展的能力。

### 1. 单机部署（适合小型团队/10人以下）
- 所有组件部署在同一台服务器，结构简单，维护成本低
- 配置要求：4核CPU/8GB内存/100GB SSD
- 组件：核心服务 + SQLite数据库 + 网关服务（可选）
- ✅ 优点：部署简单，维护成本低，资源利用率高
- ❌ 缺点：存在单点故障风险，扩展能力有限

### 2. 主从部署（适合中型团队/10~100人）
- 主节点负责写入，多个从节点负责读，分担压力
- 会话数据共享存储（NAS/PostgreSQL），支持负载均衡
- 配置要求：主节点8核16GB，从节点4核8GB*N
- ✅ 优点：支持中等规模并发，有一定的高可用能力，维护成本适中
- ❌ 缺点：主节点仍然是单点，扩展能力有限

### 3. 分布式部署（适合大型企业/100人以上）
- 微服务架构，各组件独立部署，支持弹性扩缩容
- 组件：API网关 + Agent服务集群 + 工具执行集群 + Redis缓存 + 消息队列 + PostgreSQL集群 + 监控告警
- ✅ 优点：高可用，弹性扩缩容，支持大规模并发
- ❌ 缺点：架构复杂，维护成本高，需要专业的运维团队

## 单机部署步骤

### 1. 下载安装
```bash
git clone https://github.com/a1pha3/hermes-agent.git /opt/hermes-agent
cd /opt/hermes-agent
python3.11 -m venv venv
source venv/bin/activate
pip install -e .[all]
hermes --version
```

### 2. 配置环境变量
```bash
mkdir -p /etc/hermes
cat > /etc/hermes/.env << EOF
ANTHROPIC_API_KEY=your_anthropic_api_key
OPENAI_API_KEY=your_openai_api_key
HERMES_EXEC_ASK=true
HERMES_FILE_RESTRICT=true
HERMES_ENV=production
HERMES_LOG_LEVEL=info
EOF
```

### 3. 配置文件
```yaml
# /etc/hermes/config.yaml
model:
  default: anthropic/claude-3-5-sonnet-20240620
  timeout: 120
  max_retries: 3
  cache_enabled: true
  cache_ttl: 3600

session:
  save_history: true
  max_history_length: 50
  auto_compress_context: true

tools:
  enabled_toolsets: [core, file, web, terminal, code]
  exec_ask: true

gateway:
  enabled_platforms: [feishu, wecom]
  host: 0.0.0.0
  port: 8080
  allow_all_users: false
```

### 4. 配置系统服务（Supervisor）
> 原理说明：使用supervisor管理进程可以保证服务异常退出时自动重启，提升服务可用性；使用非root用户运行服务可以最小化安全风险，即使服务被入侵也不会获得系统最高权限。

```bash
mkdir -p /var/log/hermes
useradd -r -s /sbin/nologin hermes # 创建无登录权限的系统用户
chown -R hermes:hermes /opt/hermes-agent /etc/hermes /var/log/hermes # 分配目录权限

cat > /etc/supervisor/conf.d/hermes.conf << EOF
[program:hermes-gateway]
command=/opt/hermes-agent/venv/bin/hermes gateway
directory=/opt/hermes-agent
user=hermes # 使用非root用户运行
environment=PATH="/opt/hermes-agent/venv/bin",HOME="/opt/hermes",HERMES_HOME="/etc/hermes"
stdout_logfile=/var/log/hermes/gateway.log
stderr_logfile=/var/log/hermes/gateway.err.log
autostart=true # 开机自动启动
autorestart=true # 异常退出自动重启
startretries=3 # 启动失败重试3次
stopasgroup=true # 停止时停止所有子进程
killasgroup=true # 杀死时杀死所有子进程
EOF

supervisorctl update
supervisorctl start hermes-gateway
```

### 5. 配置Nginx反向代理
```nginx
server {
    listen 80;
    server_name hermes.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name hermes.example.com;
    ssl_certificate /etc/ssl/certs/hermes.example.com.pem;
    ssl_certificate_key /etc/ssl/private/hermes.example.com.key;

    client_max_body_size 100M;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 300;
    }
}
```

## Docker 部署
> 原理说明：Docker部署可以保证运行环境的一致性，避免"本地能运行生产环境不能运行"的问题，同时方便快速扩缩容和版本升级，内置的安全配置可以最小化攻击面。

### docker-compose.yml
```yaml
version: '3.8'
services:
  hermes:
    build: .
    restart: always
    ports: ["8080:8080"]
    volumes:
      - ./config:/etc/hermes # 配置文件挂载到外部，方便修改
      - ./data:/var/lib/hermes # 数据目录挂载，容器重启不丢失数据
      - ./logs:/var/log/hermes # 日志目录挂载，方便排查问题
    environment:
      - TZ=Asia/Shanghai
      - HERMES_HOME=/etc/hermes
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY} # 从环境变量读取敏感信息，避免硬编码
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - HERMES_ENV=production
    security_opt:
      - no-new-privileges:true # 禁止提升权限
    cap_drop:
      - ALL # 丢弃所有不需要的系统权限，最小化攻击面

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes # 开启AOF持久化，保证数据不丢失

volumes:
  redis-data:
```

启动命令：
```bash
echo "ANTHROPIC_API_KEY=your_key\nOPENAI_API_KEY=your_key" > .env
docker compose up -d
```

## 高可用配置
> 原理说明：生产环境需要避免单点故障，通过数据库高可用、缓存集群、负载均衡等手段保证服务在单个节点故障时仍然可用，减少停机时间。

### 1. 数据库高可用（PostgreSQL）
使用PostgreSQL替换SQLite，支持多节点并发访问和高可用集群部署：
```yaml
persistence:
  driver: postgresql
  dsn: postgresql://user:password@db-host:5432/hermes?sslmode=require # 启用SSL加密传输
  pool_size: 20 # 连接池大小
  max_overflow: 30 # 最大溢出连接数
```
> 建议配置PostgreSQL主从集群或者云厂商的RDS服务，保证数据库高可用。

### 2. 会话缓存（Redis）
使用Redis存储会话缓存，提升访问速度，支持多节点共享缓存：
```yaml
cache:
  driver: redis
  dsn: redis://redis-host:6379/0
  ttl: 86400 # 缓存有效期1天
```
> 建议配置Redis集群或者哨兵模式，避免缓存单点故障。

### 3. 负载均衡配置（Nginx）
通过Nginx负载均衡将请求分发到多个服务节点，提升并发处理能力和可用性：
```nginx
upstream hermes {
    server hermes-node1:8080 weight=1;
    server hermes-node2:8080 weight=1;
    keepalive 32; # 长连接复用，提升性能
}
server {
    location / {
        proxy_pass http://hermes;
        ip_hash; # 会话保持，同一个用户的请求路由到同一个节点，保证上下文连续
    }
}
```

## 安全配置
> 原理说明：生产环境面临各种安全风险，多层安全防护可以最小化攻击面，降低安全事件的影响，数据泄露和系统入侵可能给企业带来严重损失。

1. 使用非root用户运行服务，禁止写入系统目录，最小化攻击面
2. 所有对外访问使用HTTPS加密，API接口配置IP白名单，避免未授权访问
3. 敏感信息加密存储，操作日志保留至少6个月，满足合规审计要求
4. 开启高危命令审批，代码执行在隔离沙箱中运行，避免恶意操作破坏系统
5. 定期安全扫描，修复系统漏洞，及时响应安全威胁

## 监控告警
> 原理说明：监控可以提前发现潜在的问题，在影响用户之前解决，避免服务不可用导致的损失，完善的监控是生产环境必不可少的部分。

核心监控指标及告警阈值：
- 服务指标：在线节点数、请求成功率、响应时间（告警：成功率<99%，响应时间>5s）
- 资源指标：CPU/内存/磁盘使用率（告警：CPU>80%，内存>85%，磁盘>90%）
- 模型指标：API调用成功率、缓存命中率（告警：成功率<95%，命中率<50%）
- 工具指标：工具调用成功率、异常调用次数（告警：成功率<90%）
> 建议搭配Prometheus+Grafana搭建监控可视化面板，配置邮件、企业微信、短信等告警渠道，及时接收告警通知。

## 备份与恢复
> 原理说明：定期备份可以在数据损坏、误操作、硬件故障时快速恢复数据，最小化数据损失和停机时间，备份数据需要定期验证可恢复性，避免备份不可用的情况。

### 备份脚本
```bash
#!/bin/bash
BACKUP_DIR=/data/backup/hermes
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR
# 备份会话数据库
sqlite3 /etc/hermes/sessions.db ".backup $BACKUP_DIR/sessions_$DATE.db"
# 备份配置文件
cp -r /etc/hermes $BACKUP_DIR/config_$DATE
# 删除7天前的旧备份，避免磁盘占用过高
find $BACKUP_DIR -type f -mtime +7 -delete
```

添加定时任务：每天凌晨2点执行备份（业务低峰期）
```bash
0 2 * * * /opt/scripts/backup.sh
```
> 建议定期将备份数据同步到异地存储，避免本地磁盘故障导致备份丢失，每季度至少做一次恢复演练，验证备份的有效性。

## 上线检查清单
- [ ] 服务启动正常，端口监听正常
- [ ] 模型API调用正常，能够返回响应
- [ ] 所有工具调用正常，权限控制生效
- [ ] 消息平台对接正常，能够收发消息
- [ ] 会话数据持久化正常，重启不丢失
- [ ] 日志输出正常，无报错
- [ ] 监控指标正常，无异常告警
- [ ] 安全配置生效，高危操作需要审批

## 常见问题处理
### 服务启动失败
- 查看日志 `/var/log/hermes/gateway.log` 排查错误
- 检查环境变量配置是否正确，API密钥是否有效
- 检查端口是否被占用，文件权限是否正确

### 响应缓慢
- 检查网络是否正常，模型API延迟是否过高
- 检查系统资源使用率，是否CPU/内存不足
- 开启提示缓存和上下文压缩，降低Token消耗

---

## ✍️ 练习
1. 按照文档步骤完成一次单机生产环境部署，通过上线检查清单所有项
2. 配置监控告警，模拟节点故障验证告警是否正常触发
3. 模拟数据损坏场景，练习从备份恢复数据，验证恢复流程
4. 设计一个适合100人团队的主从部署架构，列出需要的服务器资源和配置项

---

## ❓ 常见问题
### Q：生产环境需要开启哪些安全配置？
A：必须开启的安全配置包括：
1. 使用非root用户运行服务，最小化权限
2. 所有对外接口使用HTTPS加密
3. 开启高危命令审批，禁用不需要的工具
4. 配置IP白名单，限制API访问来源
5. 敏感信息加密存储，避免硬编码到配置文件
这些配置可以有效降低安全风险，避免被入侵。

### Q：单机部署最多支持多少并发用户？
A：根据服务器配置不同：
- 4核8GB服务器：支持最多50个并发用户
- 8核16GB服务器：支持最多100个并发用户
超过这个规模建议升级到主从或者分布式部署，避免性能瓶颈。

### Q：怎么升级版本不影响用户使用？
A：平滑升级步骤：
1. 灰度发布：先升级一个节点，验证正常后再升级其他节点
2. 负载均衡切流：升级过程中暂时将流量切到其他正常节点
3. 会话保持：保证用户会话不中断，升级后不需要重新登录
4. 回滚预案：升级出现问题时快速回滚到旧版本
停机发布建议选择业务低峰期，提前通知用户。

### Q：生产环境日志需要保留多久？
A：根据合规要求，一般建议：
1. 操作日志：保留至少6个月，满足审计要求
2. 错误日志：保留至少3个月，方便排查历史问题
3. 访问日志：保留至少1个月
日志定期归档到低成本存储，避免占用过多磁盘空间。

### Q：怎么防止误操作导致的数据丢失？
A：防止误操作的措施：
1. 开启定期备份，并且定期验证备份可恢复
2. 数据库操作前先备份，执行前二次确认
3. 配置权限控制，只有授权人员可以操作生产环境
4. 开启操作审计，所有修改操作都有日志可查
5. 重要操作先在测试环境验证，再到生产环境执行

---

## 📖 导航
- 上一篇：[性能优化指南](./1-performance.md)
- 下一篇：[自定义扩展指南](./3-custom-extension.md)
- [返回目录](../SUMMARY.md)
