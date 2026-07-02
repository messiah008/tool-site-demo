# 部署方案

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 设计完成
> **目的**: 标准化部署流程,可重复执行

---

## 一、部署架构

```
┌─────────────────────────────────────────────┐
│  Cloudflare (CDN + WAF + DNS + TLS)         │
└──────────────────┬──────────────────────────┘
                   │
        ┌──────────▼──────────┐
        │   Linux VPS (2C4G)   │
        │   ├─ Nginx           │
        │   ├─ FastAPI 容器    │
        │   ├─ PostgreSQL 容器 │
        │   ├─ Redis 容器      │
        │   ├─ MinIO 容器      │
        │   ├─ Prometheus      │
        │   └─ Grafana         │
        └─────────────────────┘
                   │
                   │ (内网/VPN)
                   ▼
        ┌─────────────────────┐
        │  Windows Worker 机器  │
        │  ├─ Python 3.11      │
        │  ├─ pywinauto        │
        │  ├─ WPS 商业版       │
        │  └─ NSSM 服务        │
        └─────────────────────┘
```

---

## 二、环境准备

### 2.1 Linux VPS

| 项 | 要求 |
|----|------|
| 系统 | Ubuntu 22.04+ |
| 配置 | 2C4G,40G SSD |
| 端口 | 80/443 开放 |
| Docker | 24.0+ |
| Docker Compose | v2.20+ |

### 2.2 Windows Worker 机器

| 项 | 要求 |
|----|------|
| 系统 | Windows Server 2022 或 Win 11 |
| 配置 | 4C8G,100G SSD |
| 桌面 | 配置 autologon,自动登录 |
| Python | 3.11+ |
| WPS | 商业版 |

---

## 三、Linux 部署

### 3.1 目录结构

```
/opt/zhiniao/
├─ docker-compose.yml
├─ .env
├─ nginx/
│  ├─ nginx.conf
│  └─ certs/
├─ frontend/
│  └─ dist/                # 前端构建产物
├─ backend/
│  ├─ Dockerfile
│  ├─ app/
│  └─ requirements.txt
├─ schemas/                # 单一事实来源(从项目复制)
│  ├─ design-tokens.json
│  ├─ tools.registry.json
│  ├─ tool.schema.json
│  └─ api/
│     └─ openapi.yaml
├─ prometheus.yml
└─ data/                    # 持久化数据
   ├─ postgres/
   ├─ redis/
   └─ minio/
```

### 3.2 环境变量 (.env)

```env
# 数据库
POSTGRES_DB=zhiniao
POSTGRES_USER=zhiniao
POSTGRES_PASSWORD=<强密码>

# Redis
REDIS_PASSWORD=<强密码>

# MinIO
MINIO_ACCESS_KEY=<访问键>
MINIO_SECRET_KEY=<强密码>

# JWT
JWT_SECRET=<随机字符串>
JWT_EXPIRE_HOURS=168

# API
API_BASE_URL=https://api.zhiniao.tools
CORS_ORIGINS=https://zhiniao.tools,https://pdf.zhiniao.tools

# 支付(可选)
WECHAT_APP_ID=
WECHAT_APP_SECRET=
ALIPAY_APP_ID=
ALIPAY_PRIVATE_KEY=

# 限流
RATE_LIMIT_PER_MINUTE=60
RATE_LIMIT_REDEEM_PER_MINUTE=5

# 文件
MAX_FILE_SIZE_MB=50
FILE_RETENTION_HOURS=1
```

### 3.3 docker-compose.yml

```yaml
version: '3.8'
services:
  api:
    build: ./backend
    restart: always
    ports: ["8000:8000"]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }
      minio: { condition: service_started }
    env_file: .env
    volumes:
      - ./schemas:/app/schemas:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  nginx:
    image: nginx:alpine
    restart: always
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
      - ./frontend/dist:/usr/share/nginx/html:ro
    depends_on: [api]

  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes: ["./data/postgres:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: always
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes: ["./data/redis:/data"]

  minio:
    image: minio/minio
    restart: always
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: ${MINIO_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${MINIO_SECRET_KEY}
    volumes: ["./data/minio:/data"]
    command: server /data --console-address ":9001"

  prometheus:
    image: prom/prometheus
    restart: always
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./data/prometheus:/prometheus

  grafana:
    image: grafana/grafana
    restart: always
    ports: ["3001:3000"]
    volumes: ["./data/grafana:/var/lib/grafana"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: <强密码>
```

### 3.4 Nginx 配置

```nginx
# nginx/nginx.conf
events { worker_connections 1024; }
http {
  # 限流
  limit_req_zone $binary_remote_addr zone=api:10m rate=60r/m;
  limit_req_zone $binary_remote_addr zone=upload:10m rate=10r/m;

  # 前端
  server {
    listen 80;
    server_name zhiniao.tools www.zhiniao.tools;
    return 301 https://$host$request_uri;
  }
  server {
    listen 443 ssl http2;
    server_name zhiniao.tools www.zhiniao.tools;
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    root /usr/share/nginx/html;
    index index.html;

    # 静态资源
    location / {
      try_files $uri $uri/ /index.html;
    }

    # API 反代
    location /api/ {
      limit_req zone=api burst=20 nodelay;
      proxy_pass http://api:8000;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      client_max_body_size 60M;
    }

    # 上传限流更严
    location /api/convert {
      limit_req zone=upload burst=5 nodelay;
      proxy_pass http://api:8000;
      client_max_body_size 60M;
    }
  }

  # 子域名工具页(pdf.zhiniao.tools)
  server {
    listen 443 ssl http2;
    server_name *.zhiniao.tools;
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    root /usr/share/nginx/html;
    location / { try_files $uri $uri/ /index.html; }
    location /api/ { proxy_pass http://api:8000; }
  }
}
```

### 3.5 部署步骤

```bash
# 1. 克隆项目
git clone <repo> /opt/zhiniao
cd /opt/zhiniao

# 2. 配置环境
cp .env.example .env
vim .env  # 填密码

# 3. 构建前端
cd frontend
npm install
npm run build:tokens
npm run gen:api
npm run build
cd ..

# 4. 启动服务
docker compose up -d

# 5. 初始化数据库
docker compose exec api alembic upgrade head

# 6. 创建 MinIO buckets
docker compose exec minio mc mb minio/pdf2word-input
docker compose exec minio mc mb minio/pdf2word-output
docker compose exec minio mc mb minio/pdf2word-screenshots

# 7. 验证
curl https://zhiniao.tools/api/health
```

---

## 四、Windows Worker 部署

### 4.1 环境准备脚本

```powershell
# install-worker.ps1

# 1. 安装 Python 3.11
winget install Python.Python.3.11

# 2. 安装依赖
pip install pywinauto redis minio pyyaml pillow

# 3. 安装 WPS 商业版(手动,需 License)

# 4. 配置 autologon
# 注册表设置
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" -Name "AutoAdminLogon" -Value "1"
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" -Name "DefaultUserName" -Value "worker"
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" -Name "DefaultPassword" -Value "<密码>"

# 5. 配置 hosts 屏蔽 WPS 广告
Add-Content C:\Windows\System32\drivers\etc\hosts @"
127.0.0.1 ad.wps.cn
127.0.0.1 info.wps.cn
127.0.0.1 push.wps.cn
"@

# 6. 下载 Worker 代码
git clone <repo> C:\worker

# 7. 配置
Copy-Item C:\worker\worker_config.example.yaml C:\worker\worker_config.yaml
# 编辑 worker_config.yaml 填 Redis/MinIO 地址
```

### 4.2 NSSM 注册服务

```powershell
# 下载 NSSM
# https://nssm.cc/

# 注册 Worker 为服务
nssm install zhiniao-worker "C:\Python311\python.exe" "C:\worker\main.py"
nssm set zhiniao-worker AppDirectory "C:\worker"
nssm set zhiniao-worker AppStdout "C:\worker\logs\stdout.log"
nssm set zhiniao-worker AppStderr "C:\worker\logs\stderr.log"
nssm set zhiniao-worker Start SERVICE_AUTO_START
nssm set zhiniao-worker Type SERVICE_INTERACTIVE_PROCESS  # 关键:需要桌面会话

# 启动
nssm start zhiniao-worker
```

### 4.3 worker_config.yaml

```yaml
worker:
  worker_id: "win-worker-01"
  work_dir: "C:/worker/work"
  log_dir: "C:/worker/logs"
  screenshot_dir: "C:/worker/screenshots"

wps:
  exe_path: "C:/Program Files/Kingsoft/WPS Office/office6/wps.exe"
  restart_interval: 50
  ad_block_hosts: true

redis:
  host: "<VPS_IP>"
  port: 6379
  password: "${REDIS_PASSWORD}"
  db: 0

minio:
  endpoint: "<VPS_IP>:9000"
  access_key: "${MINIO_ACCESS_KEY}"
  secret_key: "${MINIO_SECRET_KEY}"
  secure: true

task:
  timeout: 300
  max_retry: 2
  heartbeat_interval: 10
```

---

## 五、监控与运维

### 5.1 Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'api'
    static_configs:
      - targets: ['api:8000']
  - job_name: 'worker'
    static_configs:
      - targets: ['<worker_ip>:9100']
```

### 5.2 关键告警

| 告警 | 条件 | 通知方式 |
|------|------|---------|
| API 5xx 错误率 | > 1% | 邮件/微信 |
| 转换成功率 | < 90% | 立即 |
| 队列堆积 | > 50 | 立即 |
| Worker 离线 | 心跳过期 | 立即 |
| 数据库连接数 | > 80% | 邮件 |
| 磁盘使用 | > 80% | 邮件 |

### 5.3 日常运维

```bash
# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f api
docker compose logs -f nginx

# 重启服务
docker compose restart api

# 更新镜像
docker compose pull
docker compose up -d

# 数据库备份
docker compose exec postgres pg_dump -U zhiniao zhiniao > backup.sql

# MinIO 备份
docker compose exec minio mc mirror minio/ /backup/minio
```

---

## 六、上线检查清单

### 6.1 上线前

- [ ] 域名已解析,SSL 证书已配置
- [ ] .env 中所有密码已设置(强密码)
- [ ] WPS 商业版 License 已激活
- [ ] Windows Worker 已配置 autologon
- [ ] 数据库迁移已执行
- [ ] MinIO buckets 已创建
- [ ] 限流配置已生效
- [ ] 监控告警已配置
- [ ] 备份策略已就位
- [ ] 备案信息已填写(国内服务器)

### 6.2 上线后

- [ ] 健康检查通过 `/api/health`
- [ ] 首页可访问
- [ ] 测试卡密可兑换
- [ ] 测试 PDF 上传转换
- [ ] Worker 心跳正常
- [ ] 监控面板数据正常
- [ ] 告警通道测试通过

---

## 七、回滚方案

```bash
# 回滚到上一版本
docker compose down
git checkout HEAD~1 -- .
docker compose up -d

# 数据库回滚
docker compose exec api alembic downgrade -1
```

---

## 八、扩容方案

### 8.1 横向扩 Worker

1. 准备新 Windows 机器
2. 执行 4.1 安装脚本
3. 修改 `worker_id` 为 `win-worker-02`
4. 启动服务,自动注册到 Redis
5. API 自动发现新 Worker

### 8.2 垂直扩 VPS

1. 升级 VPS 配置(4C8G)
2. 重启
3. Docker 自动恢复

### 8.3 数据库扩容

1. 启用 PostgreSQL 读写分离
2. 添加只读副本
3. 报表查询走只读库

---

## 九、相关文档

- [03-backend-architecture.md](./03-backend-architecture.md) — 后端架构
- [07-consistency-link.md](./07-consistency-link.md) — 一致性链接
- 原型: [prototypes/](./prototypes/)
