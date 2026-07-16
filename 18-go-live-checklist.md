# 上线手册

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 上线检查清单 + 灰度发布流程
> **依赖**: [09-deployment.md](./09-deployment.md), [13-operations.md](./13-operations.md)

---

## 一、上线前检查清单

### 1.1 基础设施

- [ ] 域名已注册并解析(`zhiniao.tools`)
- [ ] SSL 证书已申请(Let's Encrypt 或 Cloudflare)
- [ ] 证书文件放至 `nginx/certs/fullchain.pem` 和 `privkey.pem`
- [ ] Linux VPS 已购买(2C4G+)
- [ ] Docker 24+ 和 Docker Compose v2+ 已安装
- [ ] Windows Worker 机器已准备(WPS 商业版已激活)
- [ ] 防火墙仅开放 80/443/22

### 1.2 配置

- [ ] `.env` 中所有密码已改为强密码
- [ ] `JWT_SECRET` 改为随机字符串(`openssl rand -hex 32`)
- [ ] `DEBUG=false`
- [ ] `CORS_ORIGINS` 设为 `https://zhiniao.tools`
- [ ] 虎皮椒/PayJS 凭证已配置(`XUNHUPAY_APPID` 等)
- [ ] 支付回调地址已在虎皮椒后台配置

### 1.3 数据库

- [ ] Alembic 迁移已执行(`alembic upgrade head`)
- [ ] MinIO buckets 已创建(zhiniao-input/output/screenshots)
- [ ] 测试卡密已生成并导入
- [ ] 管理员账号已创建

### 1.4 服务

- [ ] `docker compose up -d` 启动成功
- [ ] `curl https://zhiniao.tools/api/health` 返回 200,全绿
- [ ] Nginx 配置测试通过(`nginx -t`)
- [ ] Prometheus 可抓取 `/metrics`
- [ ] Grafana 可访问,数据源已配置
- [ ] Loki 可接收日志

### 1.5 安全

- [ ] 限流配置生效(Nginx limit_req)
- [ ] 安全头已配置(HSTS/CSP/X-Frame-Options)
- [ ] ClamAV 病毒扫描已配置(可选)
- [ ] WAF 规则已配置(Cloudflare)
- [ ] 备案完成(国内服务器)
- [ ] 用户协议和隐私政策已上线

### 1.6 监控告警

- [ ] Prometheus 告警规则已加载(`alerts.yml`)
- [ ] 告警通道已配置(邮件/微信/钉钉)
- [ ] 模拟故障测试告警能收到
- [ ] Grafana 面板已配置(API/转换/Worker/队列)
- [ ] 日志查询正常(Loki)

### 1.7 备份

- [ ] 数据库定时备份已配置(crontab)
- [ ] MinIO 数据备份脚本已配置
- [ ] 备份恢复演练已完成
- [ ] 备份文件异地存储(可选)

### 1.8 业务

- [ ] 闲鱼/拼多多店铺已开
- [ ] 客服微信号已配置
- [ ] 公众号已注册(可选)
- [ ] WPS 商业版 License 已激活
- [ ] Worker 心跳正常

---

## 二、灰度发布流程

### 2.1 预发环境验证

```bash
# 1. 在预发环境部署
docker compose -f docker-compose.staging.yml up -d

# 2. 跑 E2E 测试
python scripts/e2e_test.py

# 3. 验证监控
curl http://staging:8000/api/health
curl http://staging:8000/metrics
```

### 2.2 灰度发布(10% 流量)

用 Nginx upstream 权重灰度:

```nginx
upstream api_backend {
    server api-blue:8000 weight=9;    # 旧版本 90%
    server api-green:8000 weight=1;   # 新版本 10%
}
```

观察 30 分钟:
- 错误率是否上升
- 响应时间是否变慢
- 转换成功率是否下降

### 2.3 全量发布

灰度无异常 → 切换 100% 流量:

```bash
# 1. 部署新版本到 green
docker compose up -d api-green

# 2. Nginx 切换 upstream
sed -i 's/weight=9/weight=0/' nginx.conf
sed -i 's/weight=1/weight=10/' nginx.conf
docker compose exec nginx nginx -s reload

# 3. 观察 1 小时,确认无问题后下线 blue
docker compose stop api-blue
```

### 2.4 回滚

发现问题立即回滚:

```bash
# 1. Nginx 切回旧版本
sed -i 's/weight=0/weight=9/' nginx.conf
sed -i 's/weight=10/weight=1/' nginx.conf
docker compose exec nginx nginx -s reload

# 2. 数据库回滚(如需)
docker compose exec api alembic downgrade -1

# 3. 通知相关人员
```

---

## 三、上线后验证

### 3.1 功能验证

```bash
# 首页
curl -I https://zhiniao.tools/

# 健康检查
curl https://zhiniao.tools/api/health

# 工具列表
curl https://zhiniao.tools/api/tools

# 注册测试用户
curl -X POST https://zhiniao.tools/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test_go_live","password":"test12345678"}'

# 兑换卡密
curl -X POST https://zhiniao.tools/api/auth/redeem \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"card_key":"ZN-TEST-CARD-HERE"}'

# 创建转换任务
curl -X POST https://zhiniao.tools/api/convert \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test.pdf" \
  -F "tool_id=pdf-to-word" \
  -F "quality=standard"
```

### 3.2 监控验证

- [ ] Grafana 首页显示 API 在线
- [ ] Prometheus Targets 全绿
- [ ] Loki 日志可查询
- [ ] 告警通道测试通过

### 3.3 性能验证

```bash
# 响应时间(目标 P95 < 1s)
ab -n 100 -c 10 https://zhiniao.tools/api/health

# 压力测试(可选)
ab -n 1000 -c 50 https://zhiniao.tools/api/tools
```

---

## 四、上线后 24 小时观察

### 4.1 关键指标

| 指标 | 目标 | 查询 |
|------|------|------|
| API 可用性 | ≥ 99.9% | `up{job="api"}` |
| 5xx 错误率 | < 0.1% | `rate(http_requests_total{status=~"5.."}[5m])` |
| P95 响应时间 | < 500ms | `histogram_quantile(0.95, ...)` |
| 转换成功率 | ≥ 90% | `rate(convert_tasks_total{status="success"}[10m])` |
| 队列积压 | < 10 | `queue_size` |
| Worker 在线 | ≥ 1 | `workers_online` |

### 4.2 告警处理

收到告警后:
1. 先看告警描述和标签
2. 在 Grafana 查看相关图表
3. 在 Loki 查询相关日志
4. 按 [19-runbook.md](./19-runbook.md) 处理
5. 处理后在告警系统确认恢复

---

## 五、应急联系人

| 角色 | 职责 | 联系方式 |
|------|------|---------|
| 运维 | 服务可用性 | (填写) |
| 开发 | 代码问题 | (填写) |
| DBA | 数据库 | (填写) |
| 安全 | 安全事件 | (填写) |

---

## 六、相关文档

- [09-deployment.md](./09-deployment.md) — 部署方案
- [13-operations.md](./13-operations.md) — 运维手册
- [19-runbook.md](./19-runbook.md) — 故障处理 SOP
