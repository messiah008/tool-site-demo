# 运维手册

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **目的**: 固化所有运维操作,避免重复思考
> **适用**: 开发/测试/生产环境

---

## 一、脚本清单

所有脚本在 `scripts/` 目录:

| 脚本 | 用途 | 用法 |
|------|------|------|
| `dev-up.sh` | 启动开发环境 | `bash scripts/dev-up.sh` |
| `dev-down.sh` | 停止服务 | `bash scripts/dev-down.sh` |
| `init-db.sh` | 初始化数据库 | `bash scripts/init-db.sh` |
| `status.sh` | 查看状态 | `bash scripts/status.sh` |
| `logs.sh` | 查看日志 | `bash scripts/logs.sh api 100` |
| `backup-db.sh` | 备份数据库 | `bash scripts/backup-db.sh` |
| `restore-db.sh` | 恢复数据库 | `bash scripts/restore-db.sh backup.sql.gz` |
| `check_consistency.py` | 一致性校验 | `python scripts/check_consistency.py` |
| `generate_cards.py` | 生成卡密 | `python scripts/generate_cards.py --package monthly_100 --count 50` |
| `test_backend_import.py` | 后端导入测试 | `python scripts/test_backend_import.py` |

---

## 二、首次部署流程

### 2.1 环境准备

```bash
# 1. 克隆仓库
git clone https://github.com/messiah008/tool-site-demo.git
cd tool-site-demo

# 2. 配置环境变量
cp .env.example .env
vim .env  # 修改所有密码为强密码

# 3. 配置后端环境变量
cp backend/.env.example backend/.env
vim backend/.env  # 与顶层 .env 保持一致
```

### 2.2 启动服务

```bash
# 一键启动(包含等待就绪)
bash scripts/dev-up.sh

# 初始化数据库
bash scripts/init-db.sh

# 验证
bash scripts/status.sh
```

### 2.3 验证部署

```bash
# API 健康检查
curl http://localhost:8000/api/health

# 访问 API 文档
# 浏览器打开 http://localhost:8000/docs

# 查看工具列表
curl http://localhost:8000/api/tools
```

---

## 三、日常运维

### 3.1 启停服务

```bash
# 启动
bash scripts/dev-up.sh

# 停止(保留数据)
bash scripts/dev-down.sh

# 停止并清除数据(慎用!)
docker compose down -v
```

### 3.2 查看状态

```bash
bash scripts/status.sh
```

输出包含:
- Docker 服务状态
- API 健康检查
- Redis 队列任务数
- Worker 心跳
- 磁盘使用

### 3.3 查看日志

```bash
# 查看 API 最近 100 行日志
bash scripts/logs.sh api 100

# 实时跟踪 API 日志
bash scripts/logs.sh api -f

# 查看其他服务
bash scripts/logs.sh postgres 100
bash scripts/logs.sh redis 100
bash scripts/logs.sh minio 100
```

### 3.4 重启服务

```bash
# 重启单个服务
docker compose restart api

# 重启所有服务
docker compose restart
```

---

## 四、数据库管理

### 4.1 迁移

```bash
# 执行迁移(升级到最新)
docker compose exec api alembic upgrade head

# 回滚一个版本
docker compose exec api alembic downgrade -1

# 查看当前版本
docker compose exec api alembic current

# 查看迁移历史
docker compose exec api alembic history
```

### 4.2 备份

```bash
# 手动备份
bash scripts/backup-db.sh

# 备份到指定文件
bash scripts/backup-db.sh backups/my-backup.sql.gz
```

备份文件存放在 `backups/` 目录,自动保留最近 7 个。

### 4.3 恢复

```bash
bash scripts/restore-db.sh backups/zhiniao-20260702-120000.sql.gz
```

### 4.4 定时备份(crontab)

```bash
# 每天凌晨 3 点备份
0 3 * * * cd /opt/zhiniao && bash scripts/backup-db.sh
```

### 4.5 直接连接数据库

```bash
# psql 交互
docker compose exec postgres psql -U zhiniao -d zhiniao

# 执行 SQL
docker compose exec postgres psql -U zhiniao -d zhiniao -c "SELECT * FROM users LIMIT 5;"
```

---

## 五、卡密管理

### 5.1 生成卡密

```bash
# 生成 50 张月卡(100 次/张)
python scripts/generate_cards.py --package monthly_100 --count 50

# 生成 100 张单次卡(50 次/张)
python scripts/generate_cards.py --package single_50 --count 100

# 指定批次 ID
python scripts/generate_cards.py --package monthly_100 --count 50 --batch 2026-07-xianyu

# 只生成 CSV(便于打印)
python scripts/generate_cards.py --package monthly_100 --count 50 --format csv
```

输出在 `cards/` 目录:
- `monthly_100-20260702-120000.csv` — Excel 可打开
- `monthly_100-20260702-120000.sql` — 直接导入数据库

### 5.2 导入数据库

```bash
# 方法 1: 用生成的 SQL
cat cards/monthly_100-*.sql | docker compose exec -T postgres psql -U zhiniao -d zhiniao

# 方法 2: psql \copy
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "\copy card_keys(key_string,package_type,credits,valid_days,status,batch_id,expires_at) FROM '/path/to/cards.csv' WITH CSV HEADER"
```

### 5.3 查询卡密

```bash
# 查看未使用卡密
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT key_string, package_type, credits FROM card_keys WHERE status='unused' LIMIT 10;"

# 按批次统计
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT batch_id, count(*), sum(CASE WHEN status='used' THEN 1 ELSE 0 END) as used FROM card_keys GROUP BY batch_id;"

# 查看某张卡密状态
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT * FROM card_keys WHERE key_string='ZN-XXXX-XXXX-XXXX';"
```

### 5.4 套餐配置

套餐在 `scripts/generate_cards.py` 的 `PACKAGES` 字典:

| 套餐 | 次数 | 有效期 | 建议售价 |
|------|------|--------|---------|
| `single_50` | 50 次 | 90 天 | ¥12 |
| `monthly_100` | 100 次 | 30 天 | ¥19 |
| `yearly_unlimited` | 不限次 | 365 天 | ¥99 |

修改套餐:编辑 `PACKAGES` 字典后重新生成。

---

## 六、MinIO 管理

### 6.1 控制台

访问 http://localhost:9001
- 用户名: `.env` 中的 `MINIO_ACCESS_KEY`
- 密码: `.env` 中的 `MINIO_SECRET_KEY`

### 6.2 命令行(mc)

```bash
# 配置 mc alias
docker compose exec minio mc alias set local http://localhost:9000 \
  minioadmin minioadmin

# 列出 buckets
docker compose exec minio mc ls local/

# 查看文件
docker compose exec minio mc ls local/zhiniao-input

# 删除文件
docker compose exec minio mc rm local/zhiniao-input/old-file.pdf

# 设置生命周期(1 小时自动删除)
docker compose exec minio mc ilm add --expire-days 0 --expire-hours 1 local/zhiniao-input
```

### 6.3 Buckets

| Bucket | 用途 | 保留期 |
|--------|------|--------|
| `zhiniao-input` | 上传的原始文件 | 1 小时 |
| `zhiniao-output` | 转换结果 | 24 小时 |
| `zhiniao-screenshots` | 失败诊断截图 | 7 天 |

---

## 七、Redis 管理

### 7.1 命令行

```bash
# 进入 redis-cli
docker compose exec redis redis-cli -a "$REDIS_PASSWORD"

# 查看队列
LLEN queue:wps-worker
LLEN queue:stirling

# 查看 Worker 心跳
KEYS worker:*
GET worker:wps-worker-01

# 清空队列(慎用)
DEL queue:wps-worker
```

### 7.2 队列名约定

| 队列 | 消费者 |
|------|--------|
| `queue:wps-worker` | WPS Worker |
| `queue:stirling` | Stirling Worker |
| `queue:ocr-worker` | OCR Worker |
| `queue:ai-worker` | AI Worker |
| `queue:*:priority` | 高优先级(付费用户) |

---

## 八、一致性校验

```bash
# 校验所有 schema
python scripts/check_consistency.py
```

校验内容:
1. `tools.registry.json` 符合 `tool.schema.json`
2. `design-tokens.json` 格式正确
3. `openapi.yaml` 格式正确
4. 后端实际 OpenAPI 与契约一致(如后端在运行)

**CI 集成**: 每次提交自动执行,失败则阻止合并。

---

## 九、监控

### 9.1 Grafana

访问 http://localhost:3001 (admin/`.env` 中的 `GRAFANA_PASSWORD`)

### 9.2 Prometheus

访问 http://localhost:9090

### 9.3 关键指标

| 指标 | 查询 |
|------|------|
| API 请求量 | `rate(http_requests_total[5m])` |
| 队列积压 | `queue_size` |
| Worker 在线 | `worker_online` |
| 转换成功率 | `worker_tasks_total{status="success"}` |

---

## 十、故障排查

### 10.1 API 启动失败

```bash
# 查看日志
bash scripts/logs.sh api 200

# 常见原因
# 1. 数据库未就绪 → 等待或检查 postgres
# 2. 环境变量错误 → 检查 .env
# 3. 端口占用 → lsof -i :8000
```

### 10.2 健康检查报错

```bash
# 查看哪个组件报错
curl http://localhost:8000/api/health | python -m json.tool

# 示例输出:
# {
#   "status": "degraded",
#   "postgres": "ok",
#   "redis": "error: ConnectionError",  ← Redis 出问题
#   "minio": "ok"
# }
```

### 10.3 Worker 离线

```bash
# 查看 Worker 心跳
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" KEYS "worker:*"

# 如果没有心跳,Worker 进程可能挂了
# Windows Worker: 检查 NSSM 服务
# Linux Worker: docker compose logs stirling-worker
```

### 10.4 卡密兑换失败

```bash
# 查看卡密状态
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT key_string, status, redeemed_by FROM card_keys WHERE key_string='ZN-XXXX-XXXX-XXXX';"

# 常见错误码
# E101 卡密格式错误
# E102 卡密无效或已使用
# E103 卡密已过期
```

---

## 十一、升级流程

### 11.1 拉取最新代码

```bash
git pull origin main
```

### 11.2 重新构建

```bash
docker compose build
docker compose up -d
```

### 11.3 执行迁移

```bash
docker compose exec api alembic upgrade head
```

### 11.4 验证

```bash
bash scripts/status.sh
python scripts/check_consistency.py
```

---

## 十二、生产部署 Checklist

上线前确认:

- [ ] `.env` 所有密码已改为强密码
- [ ] `JWT_SECRET` 已改为随机字符串
- [ ] 域名已解析,SSL 证书已配置
- [ ] Cloudflare 已配置 CDN + WAF
- [ ] 备案完成(国内服务器)
- [ ] 定时备份已配置(crontab)
- [ ] 监控告警已配置
- [ ] 防火墙仅开放 80/443
- [ ] `DEBUG=false`
- [ ] `CORS_ORIGINS` 设为具体域名
- [ ] WPS 商业版 License 已激活
- [ ] Worker 心跳正常

---

## 十三、应急联系

| 场景 | 联系方式 |
|------|---------|
| 数据库故障 | 先执行 `backup-db.sh`,再排查 |
| 服务不可用 | `status.sh` → `logs.sh` → 定位 |
| 卡密问题 | 查 `card_keys` 表 |
| 安全事件 | 立即改密码 + 查日志 |

---

## 十四、相关文档

- [09-deployment.md](./09-deployment.md) — 部署方案
- [12-implementation-playbook.md](./12-implementation-playbook.md) — 实施流程
- [03-backend-architecture.md](./03-backend-architecture.md) — 后端架构
