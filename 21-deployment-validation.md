# 部署验证记录

> **版本**: v1.0
> **最后更新**: 2026-07-03
> **状态**: 本地部署验证完成,记录发现的问题和修复
> **环境**: WSL2 Ubuntu 24.04 + Docker 29.5.3

---

## 一、验证结论

**本地部署全部通过**,7 个服务运行正常,E2E 业务流程跑通。

### 1.1 服务状态

| 服务 | 状态 | 端口 | 备注 |
|------|------|------|------|
| nginx | ✅ Up | 80/443 | 静态前端 + API 反代 |
| api | ✅ Healthy | 8000(内部) | FastAPI,12 路由 |
| postgres | ✅ Healthy | 5432 | 6 表已迁移 |
| redis | ✅ Up | 6379 | 队列 + 心跳 |
| minio | ✅ Healthy | 9000/9001 | 3 buckets 已创建 |
| prometheus | ✅ Up | 9090 | 指标采集正常 |
| grafana | ✅ Up | 3001 | 默认密码见 .env |

### 1.2 E2E 流程验证

```
注册 e2e_user_001 → ✅ 201,返回 JWT
登录 → ✅ 200,返回 token
查询余额 → ✅ 0 次
兑换卡密 ZN-YOTB-ZEO7-BYIO → ✅ +100 次
查询余额 → ✅ 100 次
创建高质量转换任务 → ✅ queued,余额扣减 100→99
用户中心 dashboard → ✅ 余额/配额/计数 全正确
```

### 1.3 监控验证

- ✅ `/metrics` 暴露 Prometheus 指标
- ✅ Prometheus 抓取成功(http_requests_total 等指标可见)
- ✅ API 请求计数准确(health/register/login/redeem/balance/dashboard)

---

## 二、部署中发现并修复的问题

### 2.1 SCHEMAS_DIR 路径错误

**现象**: API 启动报 `FileNotFoundError: '/schemas/tools.registry.json'`

**原因**: `config.py` 用 `Path(__file__).parent.parent.parent / "schemas"` 计算路径:
- 本地: `backend/app/config.py` → parent×3 = 项目根 → `schemas/` ✓
- Docker: `/app/app/config.py` → parent×3 = `/` → `/schemas/` ✗

但 docker-compose 挂载是 `./schemas:/app/schemas`,实际在 `/app/schemas`。

**修复** (`backend/app/config.py`):
```python
_DOCKER_SCHEMAS = Path("/app/schemas")
_LOCAL_SCHEMAS = Path(__file__).parent.parent.parent / "schemas"
SCHEMAS_DIR = _DOCKER_SCHEMAS if _DOCKER_SCHEMAS.exists() else _LOCAL_SCHEMAS
```

### 2.2 UUID 主键无数据库默认值

**现象**: 导入卡密 SQL 报 `null value in column "id" violates not-null constraint`

**原因**: SQLAlchemy 模型有 `default=uuid.uuid4`(应用层默认),但迁移 SQL 未加 `server_default`,直接 SQL INSERT 不带 id 时数据库无法自动生成。

**修复** (`backend/alembic/versions/0001_initial.py`):
```python
# 所有 UUID 主键加 server_default
sa.Column("id", postgresql.UUID(as_uuid=True),
    primary_key=True,
    server_default=sa.text("gen_random_uuid()"))
```

**注意**: 修改迁移文件后需重建 API 镜像(`docker compose up -d --build api`),因为迁移文件在构建时 baked 进镜像。

### 2.3 bcrypt 版本不兼容

**现象**: 注册接口 500 错误,日志报 `ValueError: password cannot be longer than 72 bytes`

**原因**: passlib 1.7.4 与 bcrypt 4.1+ API 不兼容(误报密码过长)。

**修复** (`backend/requirements.txt`):
```
passlib[bcrypt]==1.7.4
bcrypt==4.0.1   # ← 固定到兼容版本
```

### 2.4 Nginx public 目录挂载冲突

**现象**: Nginx 启动报 `unable to start container: read-only file system`

**原因**: docker-compose 同时挂载:
```yaml
- ./frontend/dist:/usr/share/nginx/html:ro      # 只读
- ./public:/usr/share/nginx/html/public:ro       # 在只读目录下创建子挂载 → 失败
```

**修复**:
1. 移除 `./public` 挂载行
2. 构建前端时把 `robots.txt` 和 `llms.txt` 复制到 `frontend/dist/`:
```bash
cp public/robots.txt public/llms.txt frontend/dist/
```

**已在 `frontend/scripts/build-tokens.mjs` 后补充**(或手动执行)。

### 2.5 Nginx 需 SSL 证书

**现象**: Nginx 配置强制 HTTPS,但本地无证书。

**修复**(本地开发用自签证书):
```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout nginx/certs/privkey.pem \
  -out nginx/certs/fullchain.pem \
  -days 365 -subj '/CN=localhost'
```

生产环境用 Let's Encrypt 或 Cloudflare 证书。

---

## 三、完整部署步骤(验证版)

```bash
# 1. 配置环境变量
cp .env.example .env
cp backend/.env.example backend/.env
# 编辑两个 .env,填入强密码(backend/.env 的 POSTGRES_HOST=postgres, REDIS_HOST=redis)

# 2. 构建前端
cd frontend
npm install
npm run gen:tools
npm run build:tokens
npm run build
cp ../public/robots.txt ../public/llms.txt dist/
cd ..

# 3. 生成自签证书(本地,生产用真实证书)
mkdir -p nginx/certs
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout nginx/certs/privkey.pem \
  -out nginx/certs/fullchain.pem \
  -days 365 -subj '/CN=localhost'

# 4. 启动服务
docker compose up -d postgres redis minio
# 等待 minio healthy(约 30s)
docker compose up -d api minio-init
# 等待 api healthy
docker compose up -d nginx prometheus grafana

# 5. 初始化数据库
docker compose exec api alembic upgrade head

# 6. 生成并导入测试卡密
python scripts/generate_cards.py --package monthly_100 --count 5
cat cards/monthly_100-*.sql | docker compose exec -T postgres psql -U zhiniao -d zhiniao

# 7. 验证
curl -sk https://localhost/api/health | python -m json.tool
# 应返回 status: ok, 所有组件 ok

# 8. E2E 测试(可选)
python scripts/e2e_test.py
# 注意: e2e_test.py 的 TEST_CARD_KEY 需改为实际生成的卡密
```

---

## 四、访问地址

| 服务 | 地址 | 凭证 |
|------|------|------|
| 前端 | https://localhost | 浏览器需接受自签证书 |
| API 文档 | https://localhost/docs | - |
| 健康检查 | https://localhost/api/health | - |
| MinIO 控制台 | http://localhost:9001 | .env 中的 MINIO_ACCESS_KEY/SECRET |
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3001 | .env 中的 GRAFANA_PASSWORD |

---

## 五、未验证部分(需 Windows 机器)

以下功能需要 Windows + WPS 商业版,本地 Linux 环境无法验证:

- [ ] WPS Worker 实际转换(kwpsconvert.exe)
- [ ] 高质量 PDF→Word 端到端
- [ ] Stirling Worker(需拉取 1GB 镜像,本次未启动)
- [ ] Loki/Promtail 日志聚合(本次未启动)

WPS Worker 对接见 [14-wps-worker-integration.md](./14-wps-worker-integration.md)。

---

## 六、相关文档

- [09-deployment.md](./09-deployment.md) — 部署方案
- [13-operations.md](./13-operations.md) — 运维手册
- [18-go-live-checklist.md](./18-go-live-checklist.md) — 上线手册
- [19-runbook.md](./19-runbook.md) — 故障处理
