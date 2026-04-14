# 2. 生产环境部署指南

本文档介绍如何将 Hermes Agent 部署到生产环境，保障服务的高可用、高性能、安全性和可维护性。

## 部署架构选择

### 1. 单机部署（适合小型团队/10人以下）
- 所有组件部署在同一台服务器，结构简单，维护成本低
- 配置要求：4核CPU/8GB内存/100GB SSD
- 组件：核心服务 + SQLite数据库 + 网关服务（可选）

### 2. 主从部署（适合中型团队/10~100人）
- 主节点负责写入，多个从节点负责读，分担压力
- 会话数据共享存储（NAS/PostgreSQL），支持负载均衡
- 配置要求：主节点8核16GB，从节点4核8GB*N

### 3. 分布式部署（适合大型企业/100人以上）
- 微服务架构，各组件独立部署，支持弹性扩缩容
- 组件：API网关 + Agent服务集群 + 工具执行集群 + Redis缓存 + 消息队列 + PostgreSQL集群 + 监控告警

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
```bash
mkdir -p /var/log/hermes
useradd -r -s /sbin/nologin hermes
chown -R hermes:hermes /opt/hermes-agent /etc/hermes /var/log/hermes

cat > /etc/supervisor/conf.d/hermes.conf << EOF
[program:hermes-gateway]
command=/opt/hermes-agent/venv/bin/hermes gateway
directory=/opt/hermes-agent
user=hermes
environment=PATH="/opt/hermes-agent/venv/bin",HOME="/opt/hermes",HERMES_HOME="/etc/hermes"
stdout_logfile=/var/log/hermes/gateway.log
stderr_logfile=/var/log/hermes/gateway.err.log
autostart=true
autorestart=true
startretries=3
stopasgroup=true
killasgroup=true
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

### docker-compose.yml
```yaml
version: '3.8'
services:
  hermes:
    build: .
    restart: always
    ports: ["8080:8080"]
    volumes:
      - ./config:/etc/hermes
      - ./data:/var/lib/hermes
      - ./logs:/var/log/hermes
    environment:
      - TZ=Asia/Shanghai
      - HERMES_HOME=/etc/hermes
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - HERMES_ENV=production
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  redis-data:
```

启动命令：
```bash
echo "ANTHROPIC_API_KEY=your_key\nOPENAI_API_KEY=your_key" > .env
docker compose up -d
```

## 高可用配置

### 1. 数据库高可用（PostgreSQL）
```yaml
persistence:
  driver: postgresql
  dsn: postgresql://user:password@db-host:5432/hermes?sslmode=require
  pool_size: 20
  max_overflow: 30
```

### 2. 会话缓存（Redis）
```yaml
cache:
  driver: redis
  dsn: redis://redis-host:6379/0
  ttl: 86400
```

### 3. 负载均衡配置（Nginx）
```nginx
upstream hermes {
    server hermes-node1:8080 weight=1;
    server hermes-node2:8080 weight=1;
    keepalive 32;
}
server {
    location / {
        proxy_pass http://hermes;
        ip_hash; # 会话保持
    }
}
```

## 安全配置
1. 使用非root用户运行服务，禁止写入系统目录
2. 所有对外访问使用HTTPS加密，API接口配置IP白名单
3. 敏感信息加密存储，操作日志保留至少6个月
4. 开启高危命令审批，代码执行在隔离沙箱中运行
5. 定期安全扫描，修复系统漏洞

## 监控告警
核心监控指标：
- 服务指标：在线节点数、请求成功率、响应时间（告警：成功率<99%，响应时间>5s）
- 资源指标：CPU/内存/磁盘使用率（告警：CPU>80%，内存>85%，磁盘>90%）
- 模型指标：API调用成功率、缓存命中率（告警：成功率<95%，命中率<50%）
- 工具指标：工具调用成功率、异常调用次数（告警：成功率<90%）

## 备份与恢复
### 备份脚本
```bash
#!/bin/bash
BACKUP_DIR=/data/backup/hermes
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR
sqlite3 /etc/hermes/sessions.db ".backup $BACKUP_DIR/sessions_$DATE.db"
cp -r /etc/hermes $BACKUP_DIR/config_$DATE
find $BACKUP_DIR -type f -mtime +7 -delete
```

添加定时任务：每天凌晨2点执行备份
```bash
0 2 * * * /opt/scripts/backup.sh
```

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
