# 后端架构方案

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **依赖文档**: [04-api-contract.md](./04-api-contract.md), [06-tool-registry.md](./06-tool-registry.md)
> **状态**: 设计完成,待实现

---

## 一、架构概览

```
公网用户
   │ HTTPS
   ▼
┌──────────────────────────────────┐
│ Cloudflare (CDN + WAF + 限流)    │
└──────────────┬───────────────────┘
               │
       ┌───────▼────────┐
       │  Nginx 反代     │
       │  (静态+API路由) │
       └───┬────────┬────┘
           │        │
   ┌───────▼──┐  ┌──▼──────────────┐
   │ 静态前端 │  │  API 服务       │ ← FastAPI
   │ (Vue)    │  │  (Python)       │
   └──────────┘  └───┬────────┬────┘
                     │        │
             ┌───────▼──┐  ┌──▼──────┐
             │ Redis    │  │PostgreSQL│
             │ (队列+   │  │ (用户/   │
             │  缓存)   │  │  订单)   │
             └───┬──────┘  └─────────┘
                 │ BRPOP        ▲
                 ▼              │ 上传结果
       ┌──────────────────────────────────┐
       │ Windows Worker (1-N 台)            │
       │ ├─ Python + pywinauto             │
       │ ├─ WPS 客户端(商业版)              │
       │ └─ 状态机驱动转换流程              │
       └──────────────────────────────────┘
                   ▲
                   │ MinIO 文件存储
               ┌───┴──────┐
               │ MinIO    │
               └──────────┘
```

---

## 二、技术选型

| 组件 | 技术 | 理由 |
|------|------|------|
| API 服务 | FastAPI (Python 3.11) | 自动生成 OpenAPI,异步,性能好 |
| 数据库 | PostgreSQL 16 | 关系型,支持事务,JSON 字段 |
| 缓存/队列 | Redis 7 | 队列 + 缓存 + 会话 |
| 对象存储 | MinIO | 自托管,兼容 S3 |
| Worker | Python + pywinauto | WPS 自动化 |
| 进程管理 | Docker Compose | 一键部署 |
| 反代 | Nginx | 静态托管 + API 反代 |
| 监控 | Prometheus + Grafana | 指标 + 告警 |
| 日志 | Loki | 日志聚合 |

---

## 三、模块划分

### 3.1 模块依赖

```
┌─────────────────────────────────────────┐
│ API 服务 (FastAPI)                       │
├─────────────────────────────────────────┤
│ ├─ auth 模块      (用户/会话/卡密)       │
│ ├─ tools 模块     (工具注册/调用)        │
│ ├─ convert 模块   (转换任务调度)        │
│ ├─ payment 模块   (支付/订单)           │
│ ├─ admin 模块     (后台管理)           │
│ └─ worker-dispatch (Worker 调度)        │
└─────────────────────────────────────────┘
        │
        ▼ (通过 Redis 队列)
┌─────────────────────────────────────────┐
│ Windows Worker (独立部署)                │
├─────────────────────────────────────────┤
│ ├─ worker-main    (主循环)              │
│ ├─ wps-engine     (WPS 自动化)          │
│ ├─ task-handler   (任务处理)            │
│ └─ heartbeat      (心跳上报)            │
└─────────────────────────────────────────┘
```

### 3.2 各模块职责

| 模块 | 职责 | 主要表 |
|------|------|--------|
| auth | 用户注册/登录,卡密生成/校验,余额管理 | users, card_keys, balances |
| tools | 工具注册表管理,工具元数据查询 | (从 registry 加载,无表) |
| convert | 转换任务创建/查询/取消,文件上传下载 | convert_tasks |
| payment | 订单创建,支付回调,对账 | orders, payments |
| admin | 后台管理,统计,卡密批量生成 | (复用上述表) |
| worker-dispatch | Worker 注册/心跳/任务分发 | (Redis) |

---

## 四、数据库设计

### 4.1 表结构

```sql
-- 用户表
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(64) UNIQUE,
  email VARCHAR(128) UNIQUE,
  password_hash VARCHAR(128),
  wechat_openid VARCHAR(64) UNIQUE,
  status VARCHAR(16) DEFAULT 'active',  -- active/banned
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 卡密表
CREATE TABLE card_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key_string VARCHAR(32) UNIQUE NOT NULL,   -- ZN-XXXX-XXXX-XXXX
  package_type VARCHAR(32) NOT NULL,         -- single_50 / monthly_100 / yearly_unlimited
  credits INTEGER NOT NULL,                  -- 兑换后获得的次数
  valid_days INTEGER DEFAULT 90,
  status VARCHAR(16) DEFAULT 'unused',       -- unused/used/expired
  batch_id VARCHAR(32),                      -- 批次(批量生成用)
  created_by VARCHAR(64),                    -- 创建者(admin)
  redeemed_by UUID REFERENCES users(id),
  redeemed_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 余额表
CREATE TABLE balances (
  user_id UUID PRIMARY KEY REFERENCES users(id),
  credits INTEGER DEFAULT 0,                 -- 高质量转换剩余次数
  total_redeemed INTEGER DEFAULT 0,          -- 累计兑换
  total_consumed INTEGER DEFAULT 0,           -- 累计消耗
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 转换任务表
CREATE TABLE convert_tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  tool_id VARCHAR(64) NOT NULL,              -- pdf-to-word 等
  quality VARCHAR(16) NOT NULL,              -- standard/premium
  input_key VARCHAR(256) NOT NULL,           -- MinIO key
  output_key VARCHAR(256),
  status VARCHAR(16) DEFAULT 'queued',       -- queued/processing/success/failed/timeout/cancelled
  worker_id VARCHAR(64),
  options JSONB,                              -- {ocr: true, format: 'docx'}
  error_code VARCHAR(8),
  error_msg TEXT,
  retry_count INTEGER DEFAULT 0,
  file_size BIGINT,
  duration_ms INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ
);

-- 订单表
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  order_no VARCHAR(32) UNIQUE NOT NULL,
  channel VARCHAR(16) NOT NULL,              -- xianyu/pdd/wechat/alipay
  package_type VARCHAR(32) NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  status VARCHAR(16) DEFAULT 'pending',       -- pending/paid/delivered/refunded
  card_key_id UUID REFERENCES card_keys(id), -- 关联卡密
  external_order_no VARCHAR(64),              -- 闲鱼订单号
  paid_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_card_keys_key ON card_keys(key_string);
CREATE INDEX idx_card_keys_status ON card_keys(status);
CREATE INDEX idx_convert_tasks_user ON convert_tasks(user_id, created_at DESC);
CREATE INDEX idx_convert_tasks_status ON convert_tasks(status);
CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
```

### 4.2 Redis 数据结构

```
# 任务队列(人员 B 消费)
queue:convert                    # List, 普通任务
queue:convert:priority           # List, 高优先级任务

# 任务状态(快速查询)
task:{task_id}                   # Hash, TTL 24h
  - status, worker_id, started_at, completed_at, retry_count, error_code

# Worker 心跳
worker:{worker_id}               # String(JSON), TTL 30s

# 卡密防重兑(原子操作)
card:lock:{key_string}           # String, TTL 60s, 兑换时加锁

# 速率限制
ratelimit:{ip}                   # String, TTL 1min, 每分钟上传次数
```

---

## 五、API 模块详细设计

### 5.1 Auth 模块

```python
# app/api/auth.py
router = APIRouter(prefix="/api/auth", tags=["auth"])

@router.post("/register")
async def register(data: RegisterRequest): ...

@router.post("/login")
async def login(data: LoginRequest): ...

@router.post("/redeem")  # 卡密兑换
async def redeem_card(data: RedeemRequest, user: User = Depends(get_current_user)):
    # 1. 加锁防重兑
    # 2. 查询卡密
    # 3. 校验状态/有效期
    # 4. 事务:卡密标记 used + 余额增加
    # 5. 释放锁
    # 6. 返回新余额

@router.get("/balance")
async def get_balance(user: User = Depends(get_current_user)): ...
```

### 5.2 Convert 模块

```python
# app/api/convert.py
router = APIRouter(prefix="/api/convert", tags=["convert"])

@router.post("/")
async def create_convert(
    file: UploadFile,
    tool_id: str = Form(...),
    quality: str = Form(...),
    options: str = Form(...),
    user: User = Depends(get_current_user)
):
    # 1. 校验文件(类型/大小/病毒)
    # 2. 校验配额(免费3次/天,付费检查余额)
    # 3. 高质量转换:扣减余额
    # 4. 上传 PDF 到 MinIO
    # 5. 创建 convert_tasks 记录
    # 6. LPUSH 到 Redis 队列
    # 7. 返回 task_id

@router.get("/{task_id}")
async def get_task(task_id: str, user: User = Depends(get_current_user)):
    # 优先查 Redis,回退 PostgreSQL

@router.get("/{task_id}/download")
async def download(task_id: str, user: User = Depends(get_current_user)):
    # 生成 MinIO 预签名下载 URL

@router.delete("/{task_id}")
async def cancel_task(task_id: str, user: User = Depends(get_current_user)):
    # 标记 cancelled,Worker 检测后跳过
```

### 5.3 Payment 模块

```python
# app/api/payment.py
router = APIRouter(prefix="/api/payment", tags=["payment"])

@router.post("/create-order")
async def create_order(data: CreateOrderRequest): ...

@router.post("/callback/wechat")
async def wechat_callback(request: Request): ...

@router.post("/callback/alipay")
async def alipay_callback(request: Request): ...
```

### 5.4 Worker-Dispatch 模块

```python
# app/api/workers.py
router = APIRouter(prefix="/api/workers", tags=["workers"])

@router.get("/")
async def list_workers(admin: User = Depends(get_admin_user)):
    # 扫描 Redis worker:* key,聚合状态

@router.get("/metrics")
async def worker_metrics():
    # Prometheus 指标
```

---

## 六、Worker 调度设计

### 6.1 任务消息格式(Redis Queue)

```json
{
  "task_id": "uuid",
  "tool_id": "pdf-to-word",
  "input_bucket": "pdf2word-input",
  "input_key": "input/2026/07/02/uuid.pdf",
  "output_bucket": "pdf2word-output",
  "output_key": "output/2026/07/02/uuid.docx",
  "options": {"ocr": true, "format": "docx"},
  "priority": 0,
  "created_at": "2026-07-02T10:00:00Z",
  "expires_at": "2026-07-02T11:00:00Z",
  "user_id": "uuid",
  "quality": "premium"
}
```

### 6.2 Worker 端实现

详见 [prototypes/](./prototypes/) 同级的设计文档:
- `wps-pdf2word-task-B-worker.md`(在 `C:\Users\Administrator\` 下)

Worker 主循环:
1. BRPOP 拉取任务(优先级队列优先)
2. 检查任务是否过期
3. 下载输入文件
4. 调用 WPS 自动化引擎
5. 上传输出文件
6. 更新任务状态
7. 失败重试(最多 2 次)

---

## 七、安全设计

| 风险 | 措施 |
|------|------|
| 恶意 PDF | ClamAV 扫描 + MIME 校验 + 大小限制(50MB) |
| 路径穿越 | 文件名 UUID 重命名 |
| 卡密爆破 | 单 IP 每分钟最多 5 次兑换尝试 |
| 余额篡改 | 后端事务保证扣减原子性 |
| 越权访问 | 所有查询校验 user_id |
| CSRF | SameSite Cookie + CSRF Token |
| SQL 注入 | ORM 参数化查询 |
| XSS | 输出转义,CSP 头 |

---

## 八、监控指标

| 指标 | 类型 | 来源 |
|------|------|------|
| 转换成功率 | Counter | convert_tasks 表 |
| 单任务耗时 | Histogram | convert_tasks.duration_ms |
| 队列堆积 | Gauge | Redis LLEN |
| Worker 在线数 | Gauge | Redis worker:* 扫描 |
| 卡密兑换成功率 | Counter | card_keys 表 |
| API 响应时间 | Histogram | FastAPI middleware |
| 错误码分布 | Counter | convert_tasks.error_code |

**告警阈值**:
- 转换成功率 < 90% → 告警
- 队列堆积 > 50 → 扩容 Worker
- Worker 离线 → 立即告警
- API 5xx 错误率 > 1% → 告警

---

## 九、目录结构

```
backend/
├─ app/
│  ├─ main.py                    # FastAPI 入口
│  ├─ config.py                  # 配置(环境变量)
│  ├─ database.py                # PostgreSQL 连接
│  ├─ redis_client.py            # Redis 连接
│  ├─ minio_client.py            # MinIO 连接
│  ├─ api/                       # API 路由
│  │  ├─ auth.py
│  │  ├─ convert.py
│  │  ├─ payment.py
│  │  ├─ workers.py
│  │  └─ admin.py
│  ├─ models/                    # SQLAlchemy 模型
│  │  ├─ user.py
│  │  ├─ card_key.py
│  │  ├─ balance.py
│  │  ├─ convert_task.py
│  │  └─ order.py
│  ├─ services/                  # 业务逻辑
│  │  ├─ auth_service.py
│  │  ├─ convert_service.py
│  │  ├─ payment_service.py
│  │  └─ worker_dispatch.py
│  ├─ schemas/                   # Pydantic 模型(从 OpenAPI 生成)
│  └─ utils/
│     ├─ security.py             # JWT/密码哈希
│     ├─ ratelimit.py
│     └─ file_validator.py
├─ alembic/                      # 数据库迁移
├─ tests/
├─ openapi.yaml                  # 单一来源(参见 schemas/api/)
└─ requirements.txt
```

---

## 十、部署

### 10.1 docker-compose.yml

```yaml
services:
  api:
    build: ./backend
    ports: ["8000:8000"]
    depends_on: [postgres, redis, minio]
    env_file: .env

  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./frontend/dist:/usr/share/nginx/html
      - ./certs:/etc/nginx/certs

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: zhiniao
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes: ["pg-data:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes: ["redis-data:/data"]

  minio:
    image: minio/minio
    ports: ["9000:9000", "9001:9001"]
    environment:
      MINIO_ROOT_USER: ${MINIO_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${MINIO_SECRET_KEY}
    volumes: ["minio-data:/data"]
    command: server /data --console-address ":9001"

  prometheus:
    image: prom/prometheus
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml"]

  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]

volumes:
  pg-data:
  redis-data:
  minio-data:
```

### 10.2 Worker 部署(Windows 机器)

```yaml
# Windows 上单独部署,不进 docker-compose
# 用 NSSM 注册为服务
nssm install zhiniao-worker "C:\Python311\python.exe" "C:\worker\main.py"
nssm set zhiniao-worker AppDirectory "C:\worker"
nssm start zhiniao-worker
```

详见 [09-deployment.md](./09-deployment.md)。

---

## 十一、与前端对接

后端所有接口在 `schemas/api/openapi.yaml` 定义,前端从该文件自动生成 API Client。

**关键**: 后端不手写接口文档,FastAPI 自动从代码生成 OpenAPI,与 `openapi.yaml` 比对保证一致。

详见 [04-api-contract.md](./04-api-contract.md) 和 [07-consistency-link.md](./07-consistency-link.md)。
