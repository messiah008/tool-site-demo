# 支付对接指南

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 接入指南
> **依赖**: [03-backend-architecture.md](./03-backend-architecture.md), [13-operations.md](./13-operations.md)

---

## 一、支付通道选择

### 1.1 通道对比

| 通道 | 资质要求 | 费率 | 到账 | 适合 |
|------|---------|------|------|------|
| **虎皮椒(寻虎)** | 个人即可 | 1-2% | T+1 | ✅ 个人起步期推荐 |
| **PayJS** | 个人即可 | 1-2% | T+1 | ✅ 个人微信支付 |
| **码支付** | 个人即可 | 1-2% | T+1 | 个人支付宝 |
| 微信支付官方 | 营业执照 | 0.6% | T+1 | 企业 |
| 支付宝官方 | 营业执照 | 0.6% | T+1 | 企业 |

**起步期建议**: 虎皮椒(微信+支付宝都支持,个人可用),费率略高但零资质门槛。

### 1.2 收款流程

```
用户下单 → API 创建订单(pending) → 调用虎皮椒下单 API → 返回支付链接
   ↓
用户支付 → 虎皮椒回调 /api/payment/callback/xunhupay → 验签 → pay_order_success()
   ↓
生成卡密 + 订单标记 paid + 关联 card_key_id → 用户可在"我的订单"查看卡密
   ↓
用户用卡密兑换 → /api/auth/redeem → 余额到账
```

---

## 二、虎皮椒接入

### 2.1 注册配置

1. 注册: https://xunhupay.com
2. 实名认证(个人身份证)
3. 创建应用,获取:
   - `appid`
   - `appsecret`
4. 配置回调地址: `https://zhiniao.tools/api/payment/callback/xunhupay`

### 2.2 环境变量

在 `.env` 追加:

```env
# 虎皮椒支付
XUNHUPAY_APPID=your_appid
XUNHUPAY_APPSECRET=your_appsecret
XUNHUPAY_API_URL=https://api.xunhupay.com/payment/do.html
XUNHUPAY_CALLBACK_URL=https://zhiniao.tools/api/payment/callback/xunhupay
```

### 2.3 实现下单

替换 `backend/app/api/payment.py` 的 `_generate_pay_url`:

```python
import httpx

def _generate_pay_url(order_no: str, amount: float, channel: str) -> str:
    """调用虎皮椒下单 API"""
    params = {
        "version": "1.1",
        "appid": settings.XUNHUPAY_APPID,
        "trade_order_id": order_no,
        "total_fee": f"{amount:.2f}",
        "title": "知鸟工具箱 - 高质量转换套餐",
        "time": str(int(time.time())),
        "notify_url": settings.XUNHUPAY_CALLBACK_URL,
        "return_url": f"https://zhiniao.tools/user-center?order={order_no}",
        "nonce_str": uuid.uuid4().hex,
        "type": "WAP" if channel == "wechat" else "WAP",
        "wap_url": "https://zhiniao.tools",
        "wap_name": "知鸟工具箱",
    }
    # 签名
    params["hash"] = _xunhupay_sign(params)

    response = httpx.post(settings.XUNHUPAY_API_URL, data=params, timeout=10)
    result = response.json()
    if result.get("errcode") == 0:
        return result["url"]  # 支付链接
    raise Exception(f"虎皮椒下单失败: {result.get('errmsg')}")


def _xunhupay_sign(data: dict) -> str:
    """虎皮椒签名:按 key 排序拼接 + appsecret,MD5"""
    sorted_items = sorted(k for k in data.keys() if data[k])
    sign_str = "&".join(f"{k}={data[k]}" for k in sorted_items)
    sign_str += settings.XUNHUPAY_APPSECRET
    return hashlib.md5(sign_str.encode()).hexdigest()
```

### 2.4 回调验签

`backend/app/api/payment.py` 已实现回调验签(占位),实际接入时:

```python
def _verify_xunhupay_sign(data: dict) -> str:
    """虎皮椒回调验签"""
    app_secret = settings.XUNHUPAY_APPSECRET
    # 虎皮椒签名规则:非空参数按 key 升序 + appsecret,MD5
    filtered = {k: v for k, v in data.items() if v}
    sorted_items = sorted(filtered.keys())
    sign_str = "&".join(f"{k}={filtered[k]}" for k in sorted_items) + app_secret
    return hashlib.md5(sign_str.encode()).hexdigest()
```

### 2.5 测试

虎皮椒提供测试模式:
- 沙箱环境: `https://sandbox.xunhupay.com`
- 测试支付: 0.01 元
- 测试回调: 在后台手动触发

---

## 三、PayJS 接入(备选)

### 3.1 注册配置

1. 注册: https://payjs.cn
2. 获取 `mchid` 和 `key`
3. 配置回调: `https://zhiniao.tools/api/payment/callback/payjs`

### 3.2 环境变量

```env
PAYJS_MCHID=your_mchid
PAYJS_KEY=your_key
PAYJS_API_URL=https://payjs.cn/api/native
PAYJS_CALLBACK_URL=https://zhiniao.tools/api/payment/callback/payjs
```

### 3.3 实现

PayJS 用扫码支付(返回二维码),适合 PC 端:

```python
def _generate_payjs_url(order_no: str, amount: float) -> str:
    params = {
        "mchid": settings.PAYJS_MCHID,
        "out_trade_no": order_no,
        "total_fee": int(amount * 100),  # 分
        "body": "知鸟工具箱套餐",
        "notify_url": settings.PAYJS_CALLBACK_URL,
    }
    params["sign"] = _payjs_sign(params)
    response = httpx.post(settings.PAYJS_API_URL, data=params, timeout=10)
    return response.json().get("code_url")  # 二维码内容
```

---

## 四、订单状态机

```
pending(未支付) → paid(已支付) → delivered(已交付卡密)
                 ↘ refunded(已退款)
                 ↘ expired(超时关闭,30 分钟未支付)
```

### 4.1 超时关闭

定时任务扫描 30 分钟未支付的订单,标记为 expired:

```python
# backend/app/services/order_cleanup.py
from datetime import datetime, timezone, timedelta
from sqlalchemy.orm import Session
from app.models.order import Order

def cleanup_expired_orders(db: Session):
    """关闭超时订单(30 分钟未支付)"""
    threshold = datetime.now(timezone.utc) - timedelta(minutes=30)
    expired = db.query(Order).filter(
        Order.status == "pending",
        Order.created_at < threshold,
    ).all()
    for order in expired:
        order.status = "expired"
    db.commit()
    return len(expired)
```

定时执行(crontab):
```bash
*/10 * * * * curl -X POST http://localhost:8000/api/admin/cleanup-orders -H "X-Admin-Key: $ADMIN_KEY"
```

---

## 五、对账

### 5.1 每日对账

```bash
# 查询当日已支付订单
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT order_no, amount, channel, paid_at FROM orders
   WHERE status='paid' AND paid_at::date = CURRENT_DATE
   ORDER BY paid_at;"

# 统计当日收入
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT channel, count(*), sum(amount) FROM orders
   WHERE status='paid' AND paid_at::date = CURRENT_DATE
   GROUP BY channel;"
```

### 5.2 与虎皮椒对账

虎皮椒后台导出交易明细,与本地 orders 表比对:
- 金额一致
- 订单号一致
- 时间一致

不一致 → 手动核查或联系虎皮椒客服。

---

## 六、安全要点

| 风险 | 防范 |
|------|------|
| 伪造支付回调 | 验签(必须验证 hash/sign) |
| 重复支付 | 回调幂等(已 paid 直接返回) |
| 订单金额篡改 | 创建订单时锁定金额,回调不信任金额 |
| 回调超时 | 虎皮椒会重试,确保回调 < 5 秒响应 |
| 数据库事务失败 | `pay_order_success` 用事务,失败回滚 |

---

## 七、闲鱼/拼多多对接

闲鱼和拼多多不走在线支付,而是用户在电商平台购买后,卖家**手动发货**(卡密)。

### 7.1 流程

```
闲鱼下单 → 卖家收到订单 → 用 generate_cards.py 生成卡密 → 闲鱼发卡密给买家
   ↓
买家拿到卡密 → 在 /redeem 页兑换 → 余额到账
```

### 7.2 卡密发货脚本

```bash
# 批量生成卡密(闲鱼用)
python scripts/generate_cards.py --package monthly_100 --count 10 --batch xianyu-2026-07

# 导入数据库
cat cards/monthly_100-*.sql | docker compose exec -T postgres psql -U zhiniao -d zhiniao

# 查看未使用卡密(发货用)
docker compose exec postgres psql -U zhiniao -d zhiniao -c \
  "SELECT key_string, credits FROM card_keys
   WHERE status='unused' AND batch_id='xianyu-2026-07';"
```

### 7.3 订单关联(可选)

闲鱼订单号可作为 `external_order_no` 创建本地订单:

```bash
curl -X POST http://localhost:8000/api/payment/order \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"package_type":"monthly_100","channel":"xianyu"}'
```

这样既使用卡密发货,也在系统留订单记录,便于统计。

---

## 八、上线检查清单

- [ ] 虎皮椒/PayJS 注册完成,拿到 appid/appsecret
- [ ] `.env` 配置支付环境变量
- [ ] 回调地址在虎皮椒后台配置正确
- [ ] `_generate_pay_url` 实现真实下单
- [ ] `_verify_xunhupay_sign` 实现真实验签
- [ ] 测试小额支付(0.01 元)跑通
- [ ] 回调幂等测试(重复回调不重复发卡密)
- [ ] 超时订单清理任务配置
- [ ] 对账脚本测试
- [ ] HTTPS 证书有效(回调必须 HTTPS)

---

## 九、相关文档

- [03-backend-architecture.md](./03-backend-architecture.md) — 后端架构
- [13-operations.md](./13-operations.md) — 运维手册
- [17-quota-strategy.md](./17-quota-strategy.md) — 配额策略
