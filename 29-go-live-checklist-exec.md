# 上线前检查清单(执行版)

> **版本**: v1.0
> **最后更新**: 2026-07-06
> **状态**: 按此清单逐项推进,所有项打勾即可上线
> **目的**: 把 18-go-live-checklist.md 落地为可执行清单

---

## 一、外部资源(需用户准备,我无法代劳)

| # | 项 | 状态 | 说明 | 用户操作 |
|---|----|------|------|---------|
| 1 | 域名 | ⏳ 待办 | .com/.cn,含 tools/gongju 关键词 | 在阿里云/腾讯云注册 |
| 2 | SSL 证书 | ⏳ 待办 | Cloudflare 泛证书(免费)或 Let's Encrypt | 域名接入 Cloudflare |
| 3 | 备案 | ⏳ 待办 | 国内服务器必须,20 工作日 | 阿里云/腾讯云备案系统 |
| 4 | Linux VPS | ⏳ 待办 | 2C4G,阿里云轻量 50-80元/月 | 采购 Ubuntu 22.04 |
| 5 | Windows Worker 机器 | ⏳ 待办 | 本地物理机或云桌面 | 装 WPS 商业版 |
| 6 | WPS 商业版 License | ⏳ 待办 | 联系金山销售询价 | 0571-26888888 |
| 7 | 虎皮椒账号 | ⏳ 待办 | 个人可注册,无资质门槛 | xunhupay.com 注册 |
| 8 | 闲鱼/拼多多店铺 | ⏳ 待办 | 卡密发货渠道 | 实名认证开店 |

**用户优先级**: 域名 + VPS + 备案 最先办(备案需 20 天)。

---

## 二、代码层加固(我已完成,见 L2)

| # | 项 | 状态 | 完成情况 |
|---|----|------|---------|
| 9 | DEBUG=false | ✅ | backend/.env 已设 |
| 10 | CORS 限域名 | ⏳ | 待用户给域名后改 |
| 11 | 密码改强 | ⏳ | 待用户给域名后生成 |
| 12 | JWT_SECRET 随机 | ✅ | 已用随机串 |
| 13 | /docs 关闭 | ⏳ | 生产环境关闭,见 L2 |
| 14 | 限流配置 | ✅ | nginx.conf 已配 60/10/5 每分钟 |
| 15 | 安全头 | ✅ | HSTS/X-Frame-Options 等已配 |
| 16 | 文件校验 | ✅ | 类型+大小(50MB)已校验 |
| 17 | 卡密防爆破 | ✅ | redeem 限流 5次/分钟 |

---

## 三、数据准备(已完成)

| # | 项 | 状态 | 数据量 |
|---|----|------|--------|
| 18 | 身份证归属 | ✅ | 320 条(全国主要城市) |
| 19 | 手机归属 | ✅ | 62 条(三大运营商) |
| 20 | 成语 | ✅ | 45 条 |
| 21 | 诗词 | ✅ | 57 条 |
| 22 | 历史今天 | ✅ | 26 条 |
| 23 | 测试卡密 | ✅ | 已生成(generate_cards.py) |

---

## 四、服务就绪(已完成)

| # | 项 | 状态 | 验证 |
|---|----|------|------|
| 24 | API 服务 | ✅ | 99/99 验证通过 |
| 25 | PostgreSQL | ✅ | 6 表 + 5 数据表 |
| 26 | Redis | ✅ | 队列+心跳+日志 |
| 27 | MinIO | ✅ | 3 buckets |
| 28 | Nginx | ✅ | 静态+API反代+SSL+限流 |
| 29 | Prometheus+Grafana | ✅ | 指标采集正常 |
| 30 | 前端 71 工具 | ✅ | 77/77 路由验证通过 |

---

## 五、监控告警(部分完成)

| # | 项 | 状态 | 说明 |
|---|----|------|------|
| 31 | Prometheus 指标 | ✅ | /metrics 暴露 |
| 32 | 告警规则 | ✅ | alerts.yml(7 类) |
| 33 | 告警通道 | ⏳ | 待配邮件/微信通知 |
| 34 | 日志收集 | ✅ | 前端错误自动上报 |

---

## 六、备份(部分完成)

| # | 项 | 状态 | 说明 |
|---|----|------|------|
| 35 | 数据库备份脚本 | ✅ | backup-db.sh |
| 36 | 完整备份脚本 | ✅ | backup-all.sh |
| 37 | 定时任务 | ⏳ | 待配 crontab |
| 38 | 恢复演练 | ⏳ | 待用户执行 |

---

## 七、上线步骤(用户拿到外部资源后执行)

### 7.1 域名+SSL 就绪后

```bash
# 1. 把证书放到 nginx/certs/
cp fullchain.pem nginx/certs/
cp privkey.pem nginx/certs/

# 2. 改 nginx.conf 的 server_name 为真实域名
sed -i 's/zhiniao.tools/你的域名/g' nginx/nginx.conf

# 3. 改 .env 的 CORS
echo 'CORS_ORIGINS=https://你的域名' >> backend/.env

# 4. 重新部署
bash scripts/deploy_prod.sh
```

### 7.2 支付凭证就绪后

```bash
# 1. 在虎皮椒后台获取 appid/appsecret
# 2. 填入 backend/.env
echo 'XUNHUPAY_APPID=你的appid' >> backend/.env
echo 'XUNHUPAY_APPSECRET=你的appsecret' >> backend/.env

# 3. 在虎皮椒后台配置回调地址:
#    https://你的域名/api/payment/callback/xunhupay

# 4. 实现 _generate_pay_url 真实下单(见 16-payment-integration.md)
# 5. 重建后端
docker compose up -d --build api
```

### 7.3 WPS Worker 就绪后

```bash
# 1. Windows 机器装 WPS 商业版,登录 VIP
# 2. 配置 worker_config.yaml(填 Redis/MinIO 地址)
# 3. 启动 Worker(见 14-wps-worker-integration.md)
nssm install zhiniao-wps-worker "C:\Python311\python.exe" "C:\wps-worker\worker.py"
nssm start zhiniao-wps-worker

# 4. 验证心跳
curl https://你的域名/api/health | python -m json.tool
# 应显示 workers_online: 1
```

---

## 八、Go/No-Go 决策门禁

上线前必须全部满足:

- [ ] 域名解析生效
- [ ] SSL 证书有效(浏览器无警告)
- [ ] 备案完成(国内服务器)
- [ ] `curl https://你的域名/api/health` 返回 200 全绿
- [ ] 71 个工具路由全部可访问
- [ ] 测试卡密可兑换
- [ ] 测试一单支付(0.01 元)
- [ ] WPS Worker 心跳可见
- [ ] 备份脚本测试通过

任一不满足 → 不上线。

---

## 九、相关文档

- [18-go-live-checklist.md](./18-go-live-checklist.md) — 上线手册(详细版)
- [19-runbook.md](./19-runbook.md) — 故障处理 SOP
- [20-backup-strategy.md](./20-backup-strategy.md) — 备份策略
- [16-payment-integration.md](./16-payment-integration.md) — 支付对接
- [14-wps-worker-integration.md](./14-wps-worker-integration.md) — WPS Worker 对接
