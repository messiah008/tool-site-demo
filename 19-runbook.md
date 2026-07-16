# 运维 Runbook

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 常见故障处理 SOP
> **依赖**: [13-operations.md](./13-operations.md), [18-go-live-checklist.md](./18-go-live-checklist.md)

---

## 一、故障分级

| 级别 | 定义 | 响应时间 | 处理 |
|------|------|---------|------|
| P0 严重 | 全站不可用 | 5 分钟 | 立即处理,通报 |
| P1 高 | 核心功能不可用(转换/支付) | 15 分钟 | 立即处理 |
| P2 中 | 部分功能异常 | 1 小时 | 工作时间内处理 |
| P3 低 | 体验问题 | 1 天 | 排期处理 |

---

## 二、P0:全站不可用

### 2.1 现象

- 首页打不开
- API 健康检查失败
- Prometheus 全红

### 2.2 排查步骤

```bash
# 1. 确认服务器存活
ssh user@server
docker compose ps

# 2. 检查 Nginx
docker compose logs nginx --tail=100
docker compose exec nginx nginx -t

# 3. 检查 API
docker compose logs api --tail=100
curl http://api:8000/api/health

# 4. 检查依赖
docker compose exec postgres pg_isready -U zhiniao
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" ping
```

### 2.3 常见原因和处理

| 原因 | 处理 |
|------|------|
| 服务器宕机 | 重启服务器,`docker compose up -d` |
| 磁盘满 | `docker system prune -a`,清理日志 |
| 内存不足 | `docker compose restart`,扩容 VPS |
| Nginx 配置错误 | `nginx -t` 测试,回滚配置 |
| 数据库挂了 | 见 P1.1 |

---

## 三、P1:核心功能不可用

### 3.1 数据库故障

**现象**: API 报数据库连接错误,`/api/health` 显示 `postgres: error`

```bash
# 1. 检查 PostgreSQL 状态
docker compose ps postgres
docker compose logs postgres --tail=50

# 2. 检查磁盘空间
df -h

# 3. 尝试重启
docker compose restart postgres

# 4. 如果无法启动,从备份恢复
bash scripts/restore-db.sh backups/zhiniao-latest.sql.gz
```

### 3.2 Redis 故障

**现象**: 任务队列不消费,Worker 心跳消失

```bash
# 1. 检查 Redis
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" ping

# 2. 查看内存
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" info memory

# 3. 重启
docker compose restart redis

# 4. 如果内存满,清空(会丢失队列任务,谨慎)
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" FLUSHALL
```

### 3.3 MinIO 故障

**现象**: 文件上传/下载失败

```bash
# 1. 检查 MinIO
docker compose ps minio
curl http://minio:9000/minio/health/live

# 2. 检查磁盘
df -h

# 3. 重启
docker compose restart minio

# 4. 检查 buckets
docker compose exec minio mc ls local/
```

### 3.4 WPS Worker 离线

**现象**: 高质量转换不处理,`workers_online == 0`

```bash
# 1. 检查 Worker 心跳
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" KEYS "worker:*"
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" GET worker:wps-worker-01

# 2. 如果没有心跳,登录 Windows 机器检查
# (通过远程桌面或 SSH)
# - 检查 NSSM 服务状态: nssm status zhiniao-wps-worker
# - 查看日志: C:\wps-worker\logs\stderr.log
# - 检查 WPS 是否登录 VIP

# 3. 重启 Worker
nssm restart zhiniao-wps-worker

# 4. 临时降级到标准转换(Stirling)
# 用户改用 standard quality
```

### 3.5 支付回调失败

**现象**: 用户支付但订单未变 paid

```bash
# 1. 查看支付回调日志
docker compose logs api --tail=200 | grep payment

# 2. 检查订单状态
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT order_no, status, paid_at FROM orders WHERE order_no='ZNxxx';"

# 3. 手动补单(确认虎皮椒收到钱后)
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "UPDATE orders SET status='paid', paid_at=NOW() WHERE order_no='ZNxxx';"
# 然后调用 pay_order_success 生成卡密
```

---

## 四、P2:部分功能异常

### 4.1 转换成功率低

**现象**: 转换成功率 < 90% 告警

```bash
# 1. 查看失败任务
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT tool_id, error_code, count(*) FROM convert_tasks
   WHERE status='failed' AND created_at > NOW() - INTERVAL '1 hour'
   GROUP BY tool_id, error_code;"

# 2. 查看截图诊断
docker compose exec minio mc ls local/zhiniao-screenshots/

# 3. 根据错误码处理(见 04-api-contract.md)
# E001 WPS 启动失败 → 重启 WPS Worker
# E005 转换超时 → 检查文件大小,可能需扩容
# E006 输出校验失败 → 检查 WPS 版本
```

### 4.2 队列积压

**现象**: `queue_size > 50` 告警

```bash
# 1. 查看各队列长度
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" LLEN queue:wps-worker
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" LLEN queue:stirling

# 2. 检查 Worker 处理速度
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" INFO stats

# 3. 扩容 Worker
docker compose up -d --scale stirling=3
# 或启动更多 WPS Worker 机器
```

### 4.3 卡密兑换失败率高

**现象**: `card_redeem_fail_rate > 0.2` 告警

```bash
# 1. 查看失败原因
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT key_string, status FROM card_keys WHERE status != 'unused' LIMIT 20;"

# 2. 检查是否有爆破(同 IP 大量失败)
docker compose logs api | grep "E101\|E102" | head

# 3. 如果是爆破,临时限流
# Nginx redeem_limit 已配 5 次/分钟,可调更严
```

---

## 五、P3:体验问题

### 5.1 响应慢

```bash
# 1. 检查 API P95
curl http://localhost:8000/metrics | grep http_request_duration

# 2. 检查数据库慢查询
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"

# 3. 检查连接池
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT count(*) FROM pg_stat_activity;"

# 4. 扩容或加索引
```

### 5.2 磁盘满

```bash
# 1. 查看磁盘使用
df -h
docker system df

# 2. 清理
docker system prune -a --volumes  # 谨慎!会删未使用的卷

# 3. 清理旧日志
docker compose exec nginx find /var/log/nginx -name "*.log" -mtime +7 -delete

# 4. 清理 MinIO 旧文件(已配置生命周期,可手动)
docker compose exec minio mc rm --recursive --force local/zhiniao-input/old/
```

---

## 六、安全事件

### 6.1 账号被盗

```bash
# 1. 禁用账号
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "UPDATE users SET status='banned' WHERE id='xxx';"

# 2. 撤销所有 token(改 JWT_secret 重启,会踢所有人)
# 谨慎:影响所有用户

# 3. 查日志,确认影响范围
docker compose logs api | grep "user_id"
```

### 6.2 卡密泄露

```bash
# 1. 批量禁用某批次卡密
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "UPDATE card_keys SET status='expired' WHERE batch_id='xxx' AND status='unused';"

# 2. 生成新批次
python scripts/generate_cards.py --package monthly_100 --count 50 --batch new-2026-07
```

### 6.3 DDoS 攻击

```bash
# 1. Cloudflare 开启 Under Attack 模式
# 2. Nginx 限流调严
# 3. 临时封 IP
docker compose exec nginx nginx -s reload
```

---

## 七、数据恢复

### 7.1 数据库恢复

```bash
# 1. 停止 API(避免写入)
docker compose stop api

# 2. 恢复
bash scripts/restore-db.sh backups/zhiniao-20260702-120000.sql.gz

# 3. 验证
docker compose exec postgres psql -U zhiniao -d zhiniao -c "SELECT count(*) FROM users;"

# 4. 重启 API
docker compose start api
```

### 7.2 MinIO 恢复

```bash
# 从备份恢复
docker compose exec minio mc mirror /backup/minio/zhiniao-input local/zhiniao-input
docker compose exec minio mc mirror /backup/minio/zhiniao-output local/zhiniao-output
```

---

## 八、常用诊断命令

### 8.1 查看系统状态

```bash
bash scripts/status.sh
```

### 8.2 查看日志

```bash
# API 日志
bash scripts/logs.sh api 100

# 实时跟踪
bash scripts/logs.sh api -f

# Loki 查询(在 Grafana 中)
# {container="api"} |= "error"
```

### 8.3 查看队列

```bash
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" KEYS "queue:*"
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" LLEN queue:wps-worker
```

### 8.4 查看任务状态

```bash
# 单个任务
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" HGETALL task:xxx

# 所有进行中任务
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" KEYS "task:*" | head
```

---

## 九、运维检查表(每日)

- [ ] 检查健康检查 `/api/health` 全绿
- [ ] 检查 Grafana 无告警
- [ ] 检查磁盘空间 > 20%
- [ ] 检查备份成功执行
- [ ] 检查 Worker 心跳正常
- [ ] 检查支付订单无异常

---

## 十、相关文档

- [13-operations.md](./13-operations.md) — 运维手册
- [18-go-live-checklist.md](./18-go-live-checklist.md) — 上线手册
- [04-api-contract.md](./04-api-contract.md) — 错误码表
