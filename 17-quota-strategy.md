# 配额策略

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 配额规则说明
> **依赖**: [16-payment-integration.md](./16-payment-integration.md)

---

## 一、配额设计原则

1. **免费引流**: 标准转换免费,吸引流量
2. **付费转化**: 高质量转换付费,核心收入
3. **防滥用**: 免费有每日限额,防止白嫖
4. **公平性**: 付费用户优先处理(priority 队列)

---

## 二、配额规则

### 2.1 标准转换(免费)

| 项 | 规则 |
|----|------|
| 每日免费次数 | 3 次/天 |
| 计数方式 | Redis `quota:daily:{user_id}`,TTL 到当天 24 点 |
| 次数清零 | 每天 0 点自动重置 |
| 超限处理 | 返回 E202,引导付费 |
| 后端引擎 | LibreOffice(Stirling-PDF) |

**实现**: `backend/app/utils/quota.py`

```python
FREE_DAILY_QUOTA = 3

def check_free_quota(user_id: str) -> bool:
    used = get_today_used(user_id)
    return used < FREE_DAILY_QUOTA

def increment_free_usage(user_id: str) -> int:
    key = f"quota:daily:{user_id}"
    pipe.incr(key)
    pipe.expire(key, ttl_to_midnight)
```

### 2.2 高质量转换(付费)

| 项 | 规则 |
|----|------|
| 计费方式 | 按次扣减余额 |
| 余额来源 | 卡密兑换 / 在线购买 |
| 余额不足 | 返回 E201,引导兑换/购买 |
| 后端引擎 | WPS Worker(中文 OCR 优势) |
| 优先级 | 入 `queue:wps-worker:priority` 队列,优先消费 |

**扣减逻辑**(事务保证):
```python
balance = db.query(Balance).filter(Balance.user_id == user.id).with_for_update().first()
if not balance or balance.credits <= 0:
    raise_error("E201")
balance.credits -= 1
balance.total_consumed += 1
```

### 2.3 套餐对应

| 套餐 | 次数 | 有效期 | 价格 |
|------|------|--------|------|
| single_50 | 50 次 | 90 天 | ¥12 |
| monthly_100 | 100 次 | 30 天 | ¥19 |
| yearly_unlimited | 不限次 | 365 天 | ¥99 |

---

## 三、配额查询 API

### 3.1 查询当前配额

```
GET /api/user/quota
Authorization: Bearer <token>
```

响应:
```json
{
  "free_used_today": 2,
  "free_remaining_today": 1,
  "free_daily_limit": 3,
  "premium_balance": 98,
  "can_use_standard": true,
  "can_use_premium": true
}
```

### 3.2 用户中心首页

```
GET /api/user/dashboard
```

返回余额 + 配额 + 各类记录数,前端用户中心页直接渲染。

---

## 四、配额检查时机

转换 API 创建任务时检查(`backend/app/api/convert.py`):

```python
if quality == "premium":
    # 高质量:检查余额
    if balance.credits <= 0:
        raise_error("E201")
    balance.credits -= 1
else:
    # 标准:检查每日配额
    if not check_free_quota(str(user.id)):
        raise_error("E202")
    increment_free_usage(str(user.id))
```

---

## 五、防滥用

### 5.1 频率限制

| 接口 | 限制 |
|------|------|
| 上传转换 | 60 次/分钟(IP) |
| 卡密兑换 | 5 次/分钟(IP) |
| 注册 | 3 次/小时(IP) |
| 登录失败 | 5 次后锁 15 分钟 |

### 5.2 设备指纹(未来)

防止小号刷免费配额:
- 浏览器指纹(Canvas/WebGL)
- 同设备每日总配额 5 次(跨账号)

### 5.3 文件大小限制

- 单文件最大 50MB
- 防止超大文件拖垮 Worker

---

## 六、配额调整

修改 `backend/app/utils/quota.py`:

```python
FREE_DAILY_QUOTA = 3  # ← 改这里
```

或环境变量化(未来):
```env
FREE_DAILY_QUOTA=5
```

无需重启,Redis 计数自动按新值判断(已用次数不变)。

---

## 七、监控指标

| 指标 | 查询 |
|------|------|
| 免费用户转化率 | 付费用户 / 总用户 |
| 每日免费配额使用率 | avg(free_used_today / FREE_DAILY_QUOTA) |
| 付费用户活跃度 | monthly active paid users |
| 卡密兑换成功率 | redeemed / generated |

---

## 八、相关文档

- [16-payment-integration.md](./16-payment-integration.md) — 支付对接
- [03-backend-architecture.md](./03-backend-architecture.md) — 后端架构
