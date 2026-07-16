# 数据备份策略

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 备份方案
> **依赖**: [13-operations.md](./13-operations.md), [19-runbook.md](./19-runbook.md)

---

## 一、备份策略

### 1.1 备份内容

| 数据 | 重要性 | 频率 | 保留期 |
|------|--------|------|--------|
| PostgreSQL | 高(用户/订单/卡密/任务) | 每日 + 增量 | 7 天 |
| Redis | 中(队列状态) | 每日 | 3 天 |
| MinIO output | 中(转换结果) | 每日 | 7 天 |
| MinIO input | 低(原始文件,1h 自动删) | 不备份 | - |
| 配置文件 | 高(.env/registry) | 每次变更 | 永久(git) |
| 代码 | 高 | 每次提交 | 永久(git) |

### 1.2 备份脚本

| 脚本 | 用途 | 频率 |
|------|------|------|
| `scripts/backup-db.sh` | PostgreSQL 备份 | 每日 3:00 |
| `scripts/backup-minio.sh` | MinIO bucket 备份 | 每日 4:00 |
| `scripts/backup-all.sh` | 完整备份(全部) | 每周日 2:00 |

---

## 二、定时任务配置

### 2.1 crontab

```bash
# 编辑 crontab
crontab -e

# 添加以下任务(路径按实际部署调整):
# 每日 3:00 备份数据库
0 3 * * * cd /opt/zhiniao && bash scripts/backup-db.sh >> logs/backup.log 2>&1

# 每日 4:00 备份 MinIO
0 4 * * * cd /opt/zhiniao && bash scripts/backup-minio.sh zhiniao-output >> logs/backup.log 2>&1

# 每周日 2:00 完整备份
0 2 * * 0 cd /opt/zhiniao && bash scripts/backup-all.sh >> logs/backup.log 2>&1

# 每小时检查磁盘,超 80% 告警
0 * * * * df -h | awk 'NR>1 && int($5)>80 {print "DISK FULL: "$0}' | mail -s "Disk Alert" ops@zhiniao.tools
```

### 2.2 异地备份(可选)

每日备份后同步到异地(防服务器故障):

```bash
# rsync 到另一台服务器
0 5 * * * rsync -avz --delete /opt/zhiniao/backups/ backup@remote:/backups/zhiniao/

# 或上传到 OSS
0 5 * * * ossutil cp -r /opt/zhiniao/backups/ oss://zhiniao-backups/$(date +%Y%m%d)/
```

---

## 三、恢复演练

每月做一次恢复演练,验证备份可用。

### 3.1 数据库恢复演练

```bash
# 1. 在测试环境恢复
docker compose -f docker-compose.test.yml up -d postgres
docker compose -f docker-compose.test.yml exec -T postgres psql -U zhiniao -c "DROP DATABASE IF EXISTS zhiniao_test;"
docker compose -f docker-compose.test.yml exec -T postgres psql -U zhiniao -c "CREATE DATABASE zhiniao_test;"

# 2. 恢复
gunzip -c backups/zhiniao-latest.sql.gz | \
  docker compose -f docker-compose.test.yml exec -T postgres psql -U zhiniao -d zhiniao_test

# 3. 验证
docker compose -f docker-compose.test.yml exec postgres psql -U zhiniao -d zhiniao_test -c \
  "SELECT count(*) FROM users; SELECT count(*) FROM orders WHERE status='paid';"
```

### 3.2 恢复检查清单

- [ ] 备份文件可解压
- [ ] 数据库可恢复
- [ ] 表结构完整
- [ ] 关键数据正确(用户/订单/卡密)
- [ ] MinIO 文件可访问
- [ ] 应用可连接恢复后的数据

---

## 四、备份监控

### 4.1 备份成功检查

```bash
# 检查今日备份是否存在
ls -la backups/$(date +%Y%m%d)*.sql.gz

# 如果不存在,告警
if [ ! -f "backups/zhiniao-$(date +%Y%m%d)-*.sql.gz" ]; then
  echo "BACKUP FAILED" | mail -s "Backup Alert" ops@zhiniao.tools
fi
```

### 4.2 备份完整性验证

```bash
# 数据库备份可读
gunzip -t backups/zhiniao-latest.sql.gz

# MinIO 备份可解压
tar -tzf backups/minio-latest.tar.gz | head
```

---

## 五、灾备方案

### 5.1 单机故障

| 场景 | 恢复步骤 | 预计时间 |
|------|---------|---------|
| VPS 重启 | docker compose up -d | 5 分钟 |
| 数据库损坏 | restore-db.sh | 15 分钟 |
| MinIO 损坏 | 从备份恢复 | 30 分钟 |
| 全盘故障 | 新 VPS + 恢复完整备份 | 2 小时 |

### 5.2 数据丢失

最坏情况:丢失最近一次备份后的数据(最多 24 小时)。

减少丢失:
- 增加备份频率(每日 → 每 6 小时)
- 启用 PostgreSQL WAL 归档(可恢复到任意时间点)
- Redis 开启 AOF 持久化

---

## 六、备份保留策略

| 类型 | 保留期 | 数量 |
|------|--------|------|
| 每日备份 | 7 天 | 7 个 |
| 每周完整备份 | 4 周 | 4 个 |
| 每月完整备份 | 12 月 | 12 个 |

脚本已自动清理旧备份(保留最近 7 个),月度备份需手动归档:

```bash
# 每月 1 号归档上月备份
0 2 1 * * mv /opt/zhiniao/backups/full-$(date -d 'last month' +%Y%m%d)*.tar.gz /archive/monthly/
```

---

## 七、敏感数据处理

### 7.1 备份加密(推荐)

```bash
# 加密备份
gpg --symmetric --cipher-algo AES256 backups/full-latest.tar.gz
# 输入密码,生成 .gpg 文件

# 解密恢复
gpg --decrypt backups/full-latest.tar.gz.gpg | tar -xzf -
```

### 7.2 卡密脱敏

备份中含卡密明文,需:
- 备份文件权限设为 600
- 异地传输用加密通道(HTTPS/rsync over SSH)
- 不要把备份提交到 git

---

## 八、相关文档

- [13-operations.md](./13-operations.md) — 运维手册
- [19-runbook.md](./19-runbook.md) — 故障处理
- [18-go-live-checklist.md](./18-go-live-checklist.md) — 上线手册
