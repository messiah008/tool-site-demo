# 用户/支付/卡密 三模块功能补全方案

> **版本**: v1.2
> **最后更新**: 2026-07-16
> **状态**: 设计完成,分阶段落地
> **作者**: AI 协作产出
> **依赖**: [03-backend-architecture.md](./03-backend-architecture.md), [04-api-contract.md](./04-api-contract.md), [07-consistency-link.md](./07-consistency-link.md), [16-payment-integration.md](./16-payment-integration.md), [17-quota-strategy.md](./17-quota-strategy.md), [27-lessons-learned.md](./27-lessons-learned.md), [32-ui-design-spec.md](./32-ui-design-spec.md)
> **取代**: 无(新增)
>
> **本文件是这 3 个模块的单一事实来源(SSOT)**。所有实现、测试、迭代都引用此文件。

---

## 〇、文档导航

本文件较长,建议按下面顺序阅读:

| 章节 | 读者 | 用途 |
|------|------|------|
| 一、现状盘点 | 所有接手者 | 理解已有什么、缺什么 |
| 二、模块边界 | 架构师 / AI | 模块化设计原则 |
| 三、数据模型增量 | 后端开发 | DB schema 变更 |
| 四、API 契约增量 | 前后端 | OpenAPI 扩展 |
| 五、模块 A: 用户系统 | 全栈 | 设计 + 实现 |
| 六、模块 B: 支付系统 | 全栈 | 设计 + 实现 |
| 七、模块 C: 卡密系统 | 全栈 + 运营 | 设计 + 实现 |
| 八、前端页面清单 | 前端 | 新增/改造页面 |
| 九、分阶段落地路线图 | PM / 运营 | P0-P5 路线 |
| 十、踩坑预防 | AI / 开发 | 基于错题集的预防 |
| 十一、一致性检查清单 | CI | 上线前必过 |
| 十二、迭代记录 | 所有 | 后续每次迭代追加 |

---

## 一、现状盘点

### 1.1 已有(后端)

| 组件 | 文件 | 状态 |
|------|------|------|
| User 模型 | `backend/app/models/user.py` | ✅ 完整(username/email/password_hash/wechat_openid/status) |
| Balance 模型 | `backend/app/models/balance.py` | ✅ 完整(credits/total_redeemed/total_consumed) |
| CardKey 模型 | `backend/app/models/card_key.py` | ✅ 完整(key_string/package_type/credits/valid_days/status/batch_id) |
| Order 模型 | `backend/app/models/order.py` | ✅ 完整(order_no/channel/amount/status/card_key_id/external_order_no) |
| ConvertTask 模型 | `backend/app/models/convert_task.py` | ✅ 完整 |
| 初始迁移 | `backend/alembic/versions/0001_initial.py` | ✅ 含 5 张表 + 索引 |
| 注册/登录/兑换/余额 API | `backend/app/api/auth.py` | ✅ 完整 + 行锁防重兑 |
| 订单/支付回调 API | `backend/app/api/payment.py` | ⚠️ 占位(虎皮椒/PayJS 回调验签是 TODO) |
| 用户中心 API | `backend/app/api/user.py` | ✅ dashboard/orders/cards/tasks/quota |
| 订单服务 | `backend/app/services/order_service.py` | ✅ create_order/pay_order_success(事务) |
| 套餐配置 | `backend/app/packages.py` | ✅ 3 套餐(single_50/monthly_100/yearly_unlimited) |
| 配额服务 | `backend/app/utils/quota.py` | ✅ Redis 每日免费 3 次 |
| JWT/密码哈希 | `backend/app/utils/security.py` | ✅ bcrypt + jose JWT |
| 错误码 | `backend/app/error_codes.py` | ✅ E001-E500 体系 |
| 卡密生成脚本 | `scripts/generate_cards.py` | ✅ 批量生成 + CSV/SQL 输出 |

### 1.2 已有(前端)

| 组件 | 文件 | 状态 |
|------|------|------|
| 用户 Store | `frontend/src/stores/user.ts` | ✅ Pinia + localStorage |
| API Client | `frontend/src/api/client.ts` | ✅ authApi/paymentApi/userApi 齐全 |
| 登录/注册页 | `frontend/src/pages/Login.vue` | ⚠️ 工具式简陋,违反 §3.8 通用审美 |
| 卡密兑换页 | `frontend/src/pages/Redeem.vue` | ⚠️ 工具式,违反 §3.6 舞台模式 |
| 用户中心 | `frontend/src/pages/UserCenter.vue` | ⚠️ 仅 1 Tab,无订单/卡密记录 |
| 定价页 | `frontend/src/pages/Pricing.vue` | ⚠️ 仅"闲鱼购买"按钮,无在线购买(闲鱼入口保留,P1 加在线购买) |
| 路由 | `frontend/src/router/index.ts` | ✅ 已含 /login /redeem /user-center /pricing |

### 1.3 缺失(必须补全)

**模块 A:用户系统**
- ❌ 用户中心 Tab 化(概览/订单/卡密/转换记录/资料)
- ❌ 找回密码(邮箱验证码)
- ❌ 微信扫码登录(OpenID 绑定)
- ❌ 用户资料(头像/昵称/绑定手机号)
- ❌ 修改密码 / 修改邮箱
- ❌ 注销账号(GDPR 合规)
- ❌ 登录设备列表 / 强制下线
- ❌ 验证码(注册/找回密码防刷)

**模块 B:支付系统**
- ❌ 闲鱼线下收款录入订单 + 管理员手动确认(保留闲鱼为永久渠道,统一进 orders 表统计)
- ❌ 在线支付页(选套餐→选渠道→扫码/跳转→轮询订单)
- ❌ 微信支付(官方 Native 扫码)
- ❌ 支付宝当面付
- ❌ 订单状态轮询(前端 SSE 或定时查询)
- ❌ 订单详情页(金额/时间/卡密/状态)
- ❌ 退款流程
- ❌ 超时订单清理定时任务
- ❌ 对账脚本
- ❌ 后台对账页

**模块 C:卡密系统**
- ❌ 卡密管理后台(管理员,生成/查询/导出/作废)
- ❌ 卡密核销日志(每次兑换记日志)
- ❌ 批量生成可视化(选套餐+数量+批次号→生成→导出 CSV)
- ❌ 卡密发货对接(闲鱼/拼多多订单号关联)
- ❌ 卡密过期清理定时任务
- ❌ 卡密统计(各批次使用率/过期率)

---

## 二、模块边界设计

### 2.1 三模块关系

```
┌─────────────────────────────────────────────────────┐
│ 模块 A: 用户系统 (User Module)                       │
│ ├─ 认证(注册/登录/找回/微信扫码)                    │
│ ├─ 资料(头像/昵称/手机/邮箱/密码)                  │
│ ├─ 会话(JWT/设备列表/强制下线)                    │
│ └─ 余额(查询/明细)                                │
├─────────────────────────────────────────────────────┤
│ 模块 B: 支付系统 (Payment Module)                   │
│ ├─ 套餐/packages SSOT                              │
│ ├─ 订单(创建/查询/状态机)                         │
│ ├─ 渠道(微信官方/支付宝/虎皮椒/PayJS/闲鱼)        │
│ ├─ 回调(验签/幂等)                                │
│ ├─ 退款                                            │
│ └─ 对账(每日/与渠道对账)                          │
├─────────────────────────────────────────────────────┤
│ 模块 C: 卡密系统 (Card Key Module)                  │
│ ├─ 生成(批量/批次/有效期)                         │
│ ├─ 兑换(行锁/事务/幂等)                           │
│ ├─ 核销日志                                        │
│ ├─ 后台管理(管理员,列表/导出/作废)               │
│ ├─ 发货对接(闲鱼/拼多多订单号关联)               │
│ └─ 过期清理                                        │
└─────────────────────────────────────────────────────┘

收款方式(2 种以上并存,详见 §6.0):
  方式① 闲鱼线下(永久):管理员生成卡密(C)→闲鱼售卖→录订单(channel=xianyu,手动确认)→发卡→用户兑换→余额(A)
  方式② 在线支付(P1):用户下单(A→B)→扫码→回调→自动发卡→自动兑换→余额(A)

跨模块调用关系(两种收款方式都经统一订单状态机):
  A → B(用户下单 / 管理员录入闲鱼订单)
  B → C(收款确认后生成或关联卡密:在线自动 / 闲鱼手动)
  A → C(用户兑换卡密 → 余额到账,回到 A)
  C → A(兑换成功更新 Balance,属于 A)

关键:闲鱼与在线支付共用 orders 表 + pay_order_success() 发卡逻辑,仅"确认收款"方式不同(闲鱼=管理员手动,在线=渠道回调),保证后台收入统计统一、发卡逻辑不分叉。
```

### 2.2 模块化目录结构(后端)

```
backend/app/
├── modules/                       ← 新增模块化目录
│   ├── __init__.py
│   ├── user/                      ← 模块 A
│   │   ├── __init__.py
│   │   ├── models.py              ← 复用 models/user.py + 新增 UserSession/UserProfile
│   │   ├── service.py             ← 业务逻辑(注册/登录/资料/会话)
│   │   ├── api.py                 ← HTTP 路由
│   │   └── schemas.py             ← Pydantic DTO
│   ├── payment/                   ← 模块 B
│   │   ├── __init__.py
│   │   ├── models.py              ← 复用 models/order.py + 新增 PaymentLog/Refund
│   │   ├── service.py             ← 创建订单/支付成功/退款
│   │   ├── api.py                 ← 路由(下单/查询/回调)
│   │   ├── channels/             ← 渠道适配器(策略模式)
│   │   │   ├── __init__.py
│   │   │   ├── base.py            ← PaymentChannel 抽象基类
│   │   │   ├── wechat_native.py   ← 微信 Native 扫码
│   │   │   ├── alipay_face.py     ← 支付宝当面付
│   │   │   ├── xunhupay.py        ← 虎皮椒(已部分实现)
│   │   │   └── payjs.py           ← PayJS
│   │   └── schemas.py
│   └── card_key/                 ← 模块 C
│       ├── __init__.py
│       ├── models.py              ← 复用 models/card_key.py + 新增 CardRedeemLog
│       ├── service.py             ← 生成/兑换/作废/统计
│       ├── api.py                 ← 路由(用户兑换 + 管理员管理)
│       └── schemas.py
└── (现有 api/ models/ services/ 保留,渐进迁移)
```

**迁移原则**:不破坏现有 `app/api/auth.py`、`app/api/payment.py`,新功能放 `app/modules/`,旧文件渐进重定向到 modules。详见 §10 踩坑预防。

### 2.3 模块化目录结构(前端)

```
frontend/src/
├── pages/
│   ├── auth/                     ← 新增
│   │   ├── Login.vue             ← 改造(增加微信扫码 Tab)
│   │   ├── Register.vue          ← 新增(独立页)
│   │   ├── ForgotPassword.vue    ← 新增
│   │   └── WechatLogin.vue       ← 新增(扫码回调)
│   ├── user/                     ← 新增
│   │   ├── UserCenter.vue        ← 改造(Tab 化: 概览/订单/卡密/任务/资料)
│   │   ├── Profile.vue           ← 新增(头像/昵称/密码)
│   │   └── Sessions.vue          ← 新增(设备列表/下线)
│   ├── payment/                  ← 新增
│   │   ├── Checkout.vue          ← 新增(选套餐+渠道→下单→扫码)
│   │   ├── OrderDetail.vue       ← 新增
│   │   └── Orders.vue           ← 新增(订单列表)
│   ├── redeem/                   ← 新增
│   │   └── Redeem.vue            ← 改造(舞台模式)
│   └── admin/                   ← 新增(管理员,独立路由)
│       ├── CardKeys.vue          ← 新增(卡密管理)
│       ├── Orders.vue            ← 新增(订单管理)
│       └── Dashboard.vue         ← 新增(数据看板)
├── components/
│   ├── user/                    ← 新增
│   │   ├── UserAvatar.vue
│   │   ├── BalanceCard.vue       ← 余额渐变卡片(可复用)
│   │   └── QuotaCard.vue
│   ├── payment/                 ← 新增
│   │   ├── PackageCard.vue       ← 套餐卡片
│   │   ├── ChannelTabs.vue       ← 支付渠道 Tab
│   │   ├── QrCode.vue            ← 二维码展示+倒计时
│   │   └── OrderStatus.vue      ← 状态徽章
│   └── card-key/                ← 新增
│       ├── CardKeyInput.vue     ← 输入框+实时校验
│       └── RedeemResult.vue     ← 兑换结果卡片
├── composables/
│   ├── useUserAuth.ts           ← 新增(登录/登出/微信扫码状态)
│   ├── useOrderPolling.ts       ← 新增(订单状态轮询,SSE 替代)
│   └── useBalanceSync.ts        ← 新增(余额跨页同步)
└── stores/
    ├── user.ts                  ← 改造
    ├── order.ts                  ← 新增(订单状态)
    └── admin.ts                  ← 新增(管理员状态)
```

---

## 三、数据模型增量

### 3.1 新增表(共 5 张)

#### 3.1.1 user_sessions(登录会话)

```python
class UserSession(Base):
    __tablename__ = "user_sessions"
    id = Column(UUID, primary_key=True, default=uuid.uuid4)
    user_id = Column(UUID, ForeignKey("users.id"), index=True, nullable=False)
    token_jti = Column(String(64), unique=True, index=True, nullable=False)  # JWT ID,用于吊销
    device_name = Column(String(128), nullable=True)        # "Chrome on Windows"
    device_ip = Column(String(45), nullable=True)           # IPv4/IPv6
    location = Column(String(64), nullable=True)           # IP 反查城市
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    last_active_at = Column(DateTime(timezone=True), server_default=func.now())
    expires_at = Column(DateTime(timezone=True), nullable=False)
    revoked_at = Column(DateTime(timezone=True), nullable=True)  # 手动吊销时间
```

**用途**:用户中心 → 设备列表 / 强制下线;JWT 黑名单(JTI 比对)。

#### 3.1.2 user_profiles(用户资料)

```python
class UserProfile(Base):
    __tablename__ = "user_profiles"
    user_id = Column(UUID, ForeignKey("users.id"), primary_key=True)
    nickname = Column(String(64), nullable=True)
    avatar_url = Column(String(256), nullable=True)        # MinIO key 或外链
    phone = Column(String(20), nullable=True, index=True)   # E.164 格式 +8613800000000
    phone_verified = Column(Boolean, default=False)
    bio = Column(String(256), nullable=True)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

**注意**:不直接放 users 表(避免大表 ALTER);放 user_profiles 一对一。

#### 3.1.3 email_verifications(邮箱验证码)

```python
class EmailVerification(Base):
    __tablename__ = "email_verifications"
    id = Column(UUID, primary_key=True, default=uuid.uuid4)
    email = Column(String(128), index=True, nullable=False)
    code = Column(String(8), nullable=False)               # 6 位数字
    purpose = Column(String(32), nullable=False)            # register / forgot_password / change_email
    attempts = Column(Integer, default=0)                  # 错误尝试次数
    expires_at = Column(DateTime(timezone=True), nullable=False)
    used_at = Column(DateTime(timezone=True), nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

**索引**:(email, purpose) 复合索引,验证时高效。

#### 3.1.4 payment_logs(支付流水)

```python
class PaymentLog(Base):
    __tablename__ = "payment_logs"
    id = Column(UUID, primary_key=True, default=uuid.uuid4)
    order_id = Column(UUID, ForeignKey("orders.id"), index=True, nullable=False)
    event = Column(String(32), nullable=False)             # created / paid / callback / refund / dispute
    channel = Column(String(16), nullable=False)
    raw_payload = Column(JSONB, nullable=True)             # 渠道原始回调 JSON(便于对账)
    signature_valid = Column(Boolean, default=True)        # 验签结果
    amount = Column(Numeric(10, 2), nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

**用途**:对账 / 风控 / 纠纷举证。所有渠道回调必落一条。

#### 3.1.5 card_redeem_logs(兑换日志)

```python
class CardRedeemLog(Base):
    __tablename__ = "card_redeem_logs"
    id = Column(UUID, primary_key=True, default=uuid.uuid4)
    card_key_id = Column(UUID, ForeignKey("card_keys.id"), index=True, nullable=False)
    user_id = Column(UUID, ForeignKey("users.id"), index=True, nullable=True)  # 可空(管理员作废时无 user)
    action = Column(String(16), nullable=False)            # redeem / revoke / expire
    ip = Column(String(45), nullable=True)
    user_agent = Column(String(256), nullable=True)
    credits_before = Column(Integer, nullable=True)
    credits_after = Column(Integer, nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

**用途**:卡密核销日志,对账与风控。**重要**:即使兑换失败也落一条(action=failed_attempt)。

### 3.2 现有表追加字段

```sql
-- users 追加
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20) UNIQUE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone_verified BOOLEAN DEFAULT FALSE;
ALTER TABLE users ADD COLUMN IF NOT EXISTS last_login_at TIMESTAMPTZ;
ALTER TABLE users ADD COLUMN IF NOT EXISTS last_login_ip VARCHAR(45);

-- orders 追加
ALTER TABLE orders ADD COLUMN IF NOT EXISTS pay_url TEXT;          -- 微信/支付宝扫码 URL
ALTER TABLE orders ADD COLUMN IF NOT COLUMN expires_at TIMESTAMPTZ; -- 30 分钟超时
ALTER TABLE orders ADD COLUMN IF NOT EXISTS refunded_at TIMESTAMPTZ;
ALTER TABLE orders ADD COLUMN IF NOT EXISTS refund_amount NUMERIC(10, 2);
```

### 3.3 迁移文件命名

```
backend/alembic/versions/
├── 0001_initial.py             ← 已有
├── 0002_user_sessions.py       ← 新增:user_sessions + user_profiles + email_verifications
├── 0003_payment_logs.py        ← 新增:payment_logs + orders 字段
├── 0004_card_redeem_logs.py    ← 新增:card_redeem_logs + users 字段
└── 0005_open_api.py            ← 新增:card_key_distributors + card_key_batches + card_keys 追加 batch_id/valid_from(§7.6.2)
```

**遵守错题集 #003**:UUID 主键必须有 `server_default=sa.text("gen_random_uuid()")`。

---

## 四、API 契约增量

### 4.1 OpenAPI 新增路径

追加到 `schemas/api/openapi.yaml`(下文用伪 YAML 描述,落地时合并到现有文件):

#### 模块 A:用户系统

```yaml
/api/auth/forgot-password:
  post:
    summary: 发起找回密码(发邮箱验证码)
    security: []
    requestBody: { email: string }

/api/auth/verify-code:
  post:
    summary: 校验验证码(找回密码 / 改邮箱)
    security: []
    requestBody: { email: string, code: string, purpose: string }

/api/auth/reset-password:
  post:
    summary: 重置密码(凭验证码)
    security: []
    requestBody: { email: string, code: string, new_password: string }

/api/auth/wechat/qrcode:
  get:
    summary: 获取微信扫码登录二维码 + state
    security: []
    responses: { qrcode_url: string, state: string, expires_in: int }

/api/auth/wechat/callback:
  get:
    summary: 微信扫码回调(redirect_uri)
    security: []
    parameters: [code, state]
    responses: 302 → /login/wechat?token=...

/api/user/profile:
  get:    summary: 查询个人资料
  put:    summary: 修改资料(头像/昵称/bio)
    requestBody: { nickname?, avatar_url?, bio? }

/api/user/change-password:
  post:
    summary: 修改密码(需当前密码)
    requestBody: { current_password, new_password }

/api/user/change-email:
  post:
    summary: 修改邮箱(需验证码)
    requestBody: { new_email, code }

/api/user/sessions:
  get:    summary: 登录设备列表
  delete: summary: 吊销指定会话(by token_jti) / 全部下线

/api/user/deactivate:
  post:
    summary: 注销账号(软删,30 天后硬删)
    requestBody: { reason?: string }
```

#### 模块 B:支付系统

```yaml
/api/payment/checkout:
  post:
    summary: 一键下单+发起支付
    requestBody: { package_type, channel }  # channel: wechat_native / alipay_face / xunhupay
    responses:
      200:
        order_no: string
        pay_url: string            # 扫码 URL 或跳转 URL
        expires_in: int            # 1800 秒
        qr_data: string            # 微信 native 模式返回二维码内容

/api/payment/orders/{order_no}/status:
  get:
    summary: 查询订单状态(前端轮询)
    responses: { status, paid_at?, card_key? (脱敏) }

/api/payment/orders/{order_no}/cancel:
  post:
    summary: 用户主动取消未支付订单

/api/payment/orders/{order_no}/confirm-paid:
  post:
    summary: 管理员手动确认收款(闲鱼等线下渠道,pending→paid→delivered)
    security: [admin]
    requestBody: { card_key_id?, note? }   # 可关联已生成卡密,不传则现生成

/api/payment/refund:
  post:
    summary: 申请退款(管理员)
    requestBody: { order_no, reason }

/api/payment/admin/orders:
  get:
    summary: 管理员订单列表(分页 + 过滤)
    parameters: [status, channel, date_from, date_to, page, page_size]

/api/payment/admin/reconcile:
  get:
    summary: 对账(每日与渠道对账)
    parameters: [date]

/api/payment/admin/dashboard:
  get:
    summary: 数据看板(今日/本周/本月收入、订单数)
```

#### 模块 C:卡密系统

```yaml
/api/card-keys/redeem:
  post:
    summary: 用户兑换卡密(已有,保持兼容,内部调 modules.card_key.service)
    requestBody: { card_key: string }

/api/card-keys/admin/generate:
  post:
    summary: 管理员批量生成卡密
    requestBody: { package_type, count, batch_id?, valid_days?, note? }
    responses: { batch_id, generated: [{ key_string, ... }] }

/api/card-keys/admin/list:
  get:
    summary: 管理员卡密列表(分页 + 过滤)
    parameters: [status, batch_id, package_type, redeemed_by, date_from, date_to, page, page_size]

/api/card-keys/admin/export:
  get:
    summary: 导出 CSV(选批次或状态)
    parameters: [batch_id?, status?]

/api/card-keys/admin/revoke:
  post:
    summary: 作废卡密(管理员,未使用 → revoked)
    requestBody: { key_string, reason }

/api/card-keys/admin/stats:
  get:
    summary: 卡密统计(各批次生成/已用/未用/过期)
    parameters: [batch_id?]

/api/card-keys/admin/bind-order:
  post:
    summary: 卡密关联闲鱼/拼多多订单号
    requestBody: { key_string, external_order_no, channel: "xianyu" | "pdd" }
```

### 4.2 错误码增量

追加到 `backend/app/error_codes.py`:

| 错误码 | 描述 | 用户消息 | HTTP |
|--------|------|---------|------|
| E104 | 卡密已作废 | 该卡密已作废,请联系客服 | 400 |
| E105 | 卡密未到有效期起 | 卡密尚未生效 | 400 |
| E106 | 卡密批次已停用 | 该批次卡密已停用 | 400 |
| E301 (复用) | 验证码发送过频 | 验证码发送过频,请稍后重试 | 429 |
| E302 | 验证码错误 | 验证码错误 | 400 |
| E303 | 验证码已过期 | 验证码已失效,请重新获取 | 400 |
| E304 | 邮箱已注册 | 该邮箱已注册 | 400 |
| E305 | 手机号已绑定 | 该手机号已绑定其他账号 | 400 |
| E401 (复用) | 微信 state 校验失败 | 微信登录失败,请重试 | 401 |
| E403 (复用) | 订单超时 | 订单已超时,请重新下单 | 403 |
| E405 | 订单已支付 | 订单已支付,请勿重复操作 | 400 |
| E406 | 退款金额超限 | 退款金额超过订单金额 | 400 |
| E407 | 管理员权限不足 | 需要管理员权限 | 403 |
| E408 | 不能吊销自己 | 不能吊销当前会话,请用退出登录 | 403 |
| E409 | 注销冷却期 | 注销申请已提交,30 天后生效 | 400 |
| E507 | 退款卡密已消耗 | 卡密已兑换且余额已消耗,需人工处理 | 409 |

**遵守**:每个新错误码必须含 (description, message, status, retryable),见 error_codes.py 现有格式。E501-E506 见 §7.6.5(Open API 专属)。

---

## 五、模块 A:用户系统

### 5.1 设计要点

| 项 | 规则 |
|----|------|
| 登录方式 | 用户名/邮箱/手机号 + 密码;微信扫码;短信验证码(P2) |
| 密码策略 | 至少 8 位,含大小写+数字;bcrypt 4.0.1 固定版本(错题集 #004) |
| JWT | HS256,有效期 7 天,带 JTI;登出 = JTI 加入 Redis 黑名单 |
| 找回密码 | 邮箱 6 位数字验证码,5 分钟有效,错误 5 次锁定 30 分钟 |
| 微信扫码 | OpenID 唯一绑定;新用户首次扫码 = 自动注册(用户名 wx_openid 前 8 位) |
| 设备列表 | 每次登录落一条 user_sessions,IP 反查城市(GeoIP) |
| 注销 | 软删,30 天冷静期,期间可登录取消;满 30 天后定时任务硬删 |

### 5.2 微信扫码登录流程

```
1. 前端 GET /api/auth/wechat/qrcode
   → 后端生成 state(随机 32 字节,存 Redis,key=wechat:state:{state},TTL=300s)
   → 返回 { qrcode_url: "https://open.weixin.qq.com/connect/qrconnect?appid=...&state=...", state }

2. 用户扫码 + 授权 → 微信回调 /api/auth/wechat/callback?code=xxx&state=xxx
   → 后端校验 state 在 Redis 中存在
   → 调微信 API 换 access_token + openid
   → 查 users WHERE wechat_openid = openid:
       存在 → 生成 JWT,302 → /login/wechat?token=xxx
       不存在 → 创建用户(用户名 "wx_" + openid 前 8 位),生成 JWT,302 同上
   → 删除 Redis state(防重放)

3. 前端 /login/wechat 收到 token,存 localStorage,跳 user-center
```

**配置**(`backend/.env`,见 16-payment-integration.md 风格):

```env
WECHAT_APPID=your_appid
WECHAT_APPSECRET=your_secret
WECHAT_REDIRECT_URI=https://zhiniao.tools/api/auth/wechat/callback
```

### 5.3 前端页面要点

#### Login.vue(改造)

按 §3.6 模式:顶部"登录方式选择舞台"(用户名密码 / 微信扫码 / 短信),底部表单。

| Tab | 内容 |
|-----|------|
| 账号密码 | 用户名/邮箱/手机 + 密码 + "记住我" + "忘记密码?" |
| 微信扫码 | 二维码 + 倒计时 + "用微信扫码登录" |
| 短信登录 | 手机号 + 验证码(P2,可选) |

#### UserCenter.vue(改造,Tab 化)

```
┌──────────────────────────────────────────┐
│ 顶部:余额大数字舞台(渐变)+ 配额进度条   │
├──────────────────────────────────────────┤
│ Tab: 概览 | 订单 | 卡密 | 转换记录 | 资料 │
├──────────────────────────────────────────┤
│ 概览:4 卡片(余额/今日免费/累计兑换/累计使用) + 快捷操作
│ 订单:订单卡片列表(状态徽章 + 金额 + 查看详情)
│ 卡密:卡密列表(脱敏 + 状态 + 兑换时间 + 有效期)
│ 转换记录:任务列表(工具 + 质量 + 状态 + 用时 + 下载)
│ 资料:头像/昵称/邮箱/手机/修改密码/管理设备/注销账号
└──────────────────────────────────────────┘
```

遵守 §3.8.1:首屏视觉重心是顶部余额大数字。

### 5.4 关键文件清单

| 文件 | 任务 |
|------|------|
| `backend/app/modules/user/service.py` | 注册/登录/找回/微信扫码/资料/会话 |
| `backend/app/modules/user/api.py` | HTTP 路由(替换 auth.py,保留旧路径兼容) |
| `backend/app/modules/user/schemas.py` | Pydantic DTO |
| `backend/app/utils/email.py` | 新增:邮件发送(SMTP / 阿里云邮件) |
| `frontend/src/pages/auth/Login.vue` | 改造,3 Tab |
| `frontend/src/pages/user/UserCenter.vue` | 改造,5 Tab |
| `frontend/src/pages/user/Profile.vue` | 新增 |
| `frontend/src/pages/user/Sessions.vue` | 新增 |
| `frontend/src/composables/useUserAuth.ts` | 新增 |

---

## 六、模块 B:支付系统

### 6.0 收款方式总览(多渠道并存,核心约束)

**约束:站内同时保留 2 种以上收款方式,闲鱼为永久渠道,不因上线在线支付而下线。**

| 收款方式 | 渠道 channel | 确认收款 | 发卡 | 上线阶段 | 收款入口 |
|---------|-------------|---------|------|---------|---------|
| ① 闲鱼线下(永久) | `xianyu` | 管理员手动确认 | 录单关联已有卡密/或确认时生成 | P0 | 闲鱼平台,站内仅录入订单 |
| ② 微信 Native 扫码 | `wechat_native` | 渠道回调验签 | 自动生成 | P1 | 站内 Checkout 页扫码 |
| ③ 支付宝当面付 | `alipay_face` | 渠道回调验签 | 自动生成 | P1(备选) | 站内 Checkout 页扫码 |
| ④ 虎皮椒 | `xunhupay` | 渠道回调验签 | 自动生成 | P1(已有,迁入) | 站内 Checkout 页跳转 |
| ⑤ PayJS | `payjs` | 渠道回调验签 | 自动生成 | P1(已有,迁入) | 站内 Checkout 页跳转 |

**两方式统一点**:
- 都写入 `orders` 表(`channel` 字段区分),共用订单状态机(§6.3)与 `pay_order_success()` 发卡逻辑
- 后台 Dashboard 收入按 `channel` 分类统计(闲鱼 + 在线 各一目了然),对账分开(闲鱼对闲鱼账,在线对渠道账)

**两方式差异点**:
- 闲鱼:`create_payment` 返回下单指引(非扫码),无回调,`pending→paid` 由管理员手动触发(§6.3)
- 在线:`create_payment` 返回扫码 URL,`pending→paid` 由渠道回调自动触发

### 6.1 渠道适配器(策略模式)

```python
# backend/app/modules/payment/channels/base.py
from abc import ABC, abstractmethod

class PaymentChannel(ABC):
    """支付渠道抽象基类"""
    channel_id: str          # wechat_native / alipay_face / xunhupay / payjs

    @abstractmethod
    def create_payment(self, order: Order) -> dict:
        """发起支付,返回 { pay_url, qr_data?, expires_in }"""

    @abstractmethod
    def verify_callback(self, request: Request) -> dict:
        """验签回调,返回 { order_no, status, amount, raw_payload }"""

    @abstractmethod
    def query_order(self, order: Order) -> dict:
        """主动查询订单状态(对账用)"""

    @abstractmethod
    def refund(self, order: Order, amount: Decimal, reason: str) -> dict:
        """发起退款"""


# 注册表
CHANNELS: dict[str, type[PaymentChannel]] = {
    "xianyu": XianyuManualChannel,      # 线下手动确认(永久渠道)
    "wechat_native": WechatNativeChannel,
    "alipay_face": AlipayFaceChannel,
    "xunhupay": XunhupayChannel,
    "payjs": PayjsChannel,
}

def get_channel(channel_id: str) -> PaymentChannel:
    cls = CHANNELS.get(channel_id)
    if not cls:
        raise_error("E404", f"支付渠道 {channel_id} 不支持")
    return cls()
```

```python
# backend/app/modules/payment/channels/xianyu_manual.py
class XianyuManualChannel(PaymentChannel):
    """闲鱼线下收款:无 API 回调,管理员手动确认(永久渠道,见 §6.0)。"""
    channel_id = "xianyu"
    is_manual = True

    def create_payment(self, order: Order) -> dict:
        # 不产生扫码 URL,返回闲鱼交易说明 + 录单指引
        return {
            "pay_url": None,
            "qr_data": None,
            "expires_in": 0,            # 闲鱼无站内超时,不自动过期
            "manual": True,
            "instruction": "闲鱼线下交易:管理员后台确认收款后自动发卡",
        }

    def verify_callback(self, request: Request) -> dict:
        raise_error("E404", "闲鱼渠道无回调,请用管理员手动确认")  # 不会被调用

    def query_order(self, order: Order) -> dict:
        return {"status": order.status, "manual": True}   # 对账走管理员核对

    def refund(self, order: Order, amount: Decimal, reason: str) -> dict:
        # 闲鱼退款在闲鱼平台操作,站内仅记录 refund 状态
        return {"refunded": True, "external": True, "note": "闲鱼平台已退款,站内同步状态"}
```

### 6.2 微信 Native 扫码(主要渠道)

**为什么 Native**:个人/小企业无 APP ID 也能用(服务号资质),扫码支付 PC 端体验最好。

```python
# backend/app/modules/payment/channels/wechat_native.py
class WechatNativeChannel(PaymentChannel):
    channel_id = "wechat_native"

    def create_payment(self, order: Order) -> dict:
        # 1. 调微信下单 API: https://api.mch.weixin.qq.com/v3/pay/transactions/native
        params = {
            "appid": settings.WECHAT_APPID,
            "mchid": settings.WECHAT_MCHID,
            "description": f"知鸟工具箱 - {order.package_type}",
            "out_trade_no": order.order_no,
            "notify_url": settings.WECHAT_NOTIFY_URL,
            "amount": {"total": int(order.amount * 100), "currency": "CNY"},
        }
        # 签名(微信 V3 用 RSA-SHA256,商户私钥)
        signed = self._sign_v3(params)
        resp = httpx.post("...", json=params, headers=signed, timeout=10)
        return {
            "qr_data": resp.json()["code_url"],   # weixin://wxpay/bizpayurl?pr=...
            "expires_in": 1800,
        }

    def verify_callback(self, request: Request) -> dict:
        # 1. 验微信签名(Wechatpay-Signature header)
        # 2. 解密 resource(微信 V3 用 AES-256-GCM)
        # 3. 返回 { order_no, status: "paid", amount, raw_payload }
```

**配置**:

```env
WECHAT_APPID=...
WECHAT_MCHID=...
WECHAT_API_V3_KEY=...
WECHAT_SERIAL_NO=...               # 商户证书序列号
WECHAT_PRIVATE_KEY_PATH=/run/secrets/wechat_private.pem
WECHAT_NOTIFY_URL=https://zhiniao.tools/api/payment/callback/wechat
```

**踩坑预防**:
- 微信 V3 用 RSA-SHA256,不是 MD5(旧版才是)
- 回调必须 5 秒内响应,验签+解密+落库都要快
- 商户证书是 PEM 文件,放 Docker secrets,不放 .env

### 6.3 订单状态机(强化)

```
  pending ──→ paid ──→ delivered(卡密已交付:在线=自动生成,闲鱼=关联预生成)
     │           │
     │           └──→ refunded(退款,联动卡密作废,见下)
     │
     └──→ expired(30 分钟未付,在线渠道;闲鱼不超时)
     └──→ cancelled(用户主动)

  pending → paid:必经 pay_order_success()(事务):
    1. UPDATE orders SET status='paid', paid_at=now
    2. INSERT INTO card_keys (生成卡密,关联订单)  -- 在线:现生成;闲鱼:可关联预生成
    3. UPDATE orders SET card_key_id=new_card.id
    4. INSERT INTO payment_logs (event='paid')
    5. paid → delivered:支付确认即交付,交付动作 = 卡密已就绪(已生成/已关联)
       → 在线支付:第 2 步生成后即 delivered(同事务内,状态直接 paid+delivered)
       → 闲鱼:管理员确认收款时若关联预生成卡密,pay_order_success 内置 delivered
       → delivered 语义=「卡密就绪可交付用户」,不是冗余:它区分 paid(钱到账但卡密未就绪)的中间态
       → 若第 5 步(可选自动兑换)执行,余额到账后 delivered 保持不变(交付=卡密就绪,兑换=余额到账,两件事)
```

**delivered 的真实驱动**:pay_order_success 事务结束即 delivered(卡密就绪)。该状态用于对账区分「已收款未发卡」(paid)与「已发卡」(delivered);仅当卡密生成异步或预生成未关联时,才会出现 paid 非 delivered,正常路径同事务内一步到位。

**幂等**:回调必须 `if order.status in ('paid','delivered'): return success`(避免重复发卡密)。已在 order_service.py 实现。

**闲鱼手动确认**(对应 §6.0 方式①):
- 闲鱼订单 `pending→paid` 不经回调,由管理员在后台点"确认收款"触发,内部调用同一个 `pay_order_success()`(走上面 5 步),保证发卡逻辑与在线支付完全一致、不分叉。
- 录单时可直接关联已预生成的卡密(`card_key_id`),或确认时现生成(第 2 步)。
- payment_logs 落 `event='paid', channel='xianyu', signature_valid=true(人工确认)`。

**退款↔卡密联动(闭环必补,防双重得利)**:
```
  refund(order) 事务:
    1. 校验:order.status 必须 in ('paid','delivered'),否则 E403(未支付不可退)
    2. 取 order.card_key → 校验卡密状态:
       a) card.status == 'unused'(未兑换):
          → 作废卡密 status='revoked'(§7.1 revoked 终态),落 card_redeem_logs(action='revoke_refund')
          → 退款安全,无余额影响
       b) card.status == 'used'(已兑换,余额已到账):
          → 校验用户余额是否仍 ≥ card.credits(未被消耗):
            余额充足 → 扣减 balance.credits -= card.credits,total_consumed += card.credits,落日志(action='deduct_refund')
            余额不足 → 拒绝自动退款,raise E507「卡密已兑换且余额已消耗,需人工处理」(走人工对账)
          → 卡密保持 used(已用不可逆,§7.1),仅扣余额
       c) card.status in ('revoked','expired'): raise E406/E507
    3. UPDATE orders SET status='refunded', refunded_at=now, refund_amount
    4. INSERT INTO payment_logs (event='refund')
    5. 幂等:已 refunded 直接 return
```
**关键**:退款必须经过卡密状态判断,绝不能"只改订单不处理卡密",否则用户已用余额+退款双重得利。闲鱼退款在闲鱼平台操作,站内仍需走此流程同步卡密/余额状态(否则站内余额虚高)。

### 6.4 前端轮询订单状态

```typescript
// frontend/src/composables/useOrderPolling.ts
export function useOrderPolling(orderNo: string) {
  const status = ref<string>('pending')
  const payUrl = ref<string>('')
  const qrData = ref<string>('')
  const countdown = ref(1800)   // 30 分钟
  const polling = ref(true)

  let timer: number

  async function start() {
    // 1. 创建订单
    const res = await paymentApi.checkout({ package_type, channel })
    payUrl.value = res.pay_url
    qrData.value = res.qr_data
    orderNo = res.order_no

    // 2. 启动倒计时
    timer = window.setInterval(async () => {
      countdown.value--
      if (countdown.value <= 0) {
        // 超时主动取消,避免订单悬空 pending + 用户重扫重复下单(对齐 §6.3)
        try { await paymentApi.cancelOrder(orderNo) } catch {}
        stop(); return
      }
      // 3. 轮询订单状态(每 3 秒)
      const s = await paymentApi.getOrderStatus(orderNo)
      status.value = s.status
      if (s.status === 'paid') {
        stop()
        // 显示成功 + 卡密 + 自动兑换
      }
    }, 3000)
  }

  function stop() {
    polling.value = false
    clearInterval(timer)
  }

  onUnmounted(stop)
  return { status, payUrl, qrData, countdown, start, stop }
}
```

**优化**:后续可改 SSE(Server-Sent Events)替代轮询,减请求量。

### 6.5 关键文件清单

| 文件 | 任务 |
|------|------|
| `backend/app/modules/payment/channels/base.py` | 抽象基类 |
| `backend/app/modules/payment/channels/wechat_native.py` | 微信 Native |
| `backend/app/modules/payment/channels/alipay_face.py` | 支付宝当面付(P1) |
| `backend/app/modules/payment/channels/xunhupay.py` | 虎皮椒(已有,迁入) |
| `backend/app/modules/payment/channels/payjs.py` | PayJS(已有,迁入) |
| `backend/app/modules/payment/service.py` | 创建/查询/退款/对账 |
| `backend/app/modules/payment/api.py` | HTTP 路由 |
| `backend/app/services/order_cleanup.py` | 新增:超时订单定时清理 |
| `backend/app/services/reconcile.py` | 新增:对账脚本 |
| `frontend/src/pages/payment/Checkout.vue` | 新增 |
| `frontend/src/pages/payment/OrderDetail.vue` | 新增 |
| `frontend/src/pages/payment/Orders.vue` | 新增 |
| `frontend/src/composables/useOrderPolling.ts` | 新增 |

---

## 七、模块 C:卡密系统

### 7.1 卡密生命周期

```
管理员生成 → unused → (用户兑换) → used
                ↓
            (过期)→ expired
                ↓
            (作废)→ revoked
```

**状态机**:不可逆。used 不能回到 unused,revoked 是终态。

### 7.2 兑换事务(强化已有 auth.py)

```python
# backend/app/modules/card_key/service.py
def redeem(db: Session, user_id: str, card_key_str: str, ip: str, user_agent: str) -> dict:
    card_key_str = card_key_str.upper().strip()

    # 1. 格式校验
    if not CARD_KEY_PATTERN.match(card_key_str):
        log_action(db, None, None, "failed_attempt", ip, user_agent, reason="format_error")
        raise_error("E101")

    # 2. 行锁查询
    card = db.execute(
        text("SELECT * FROM card_keys WHERE key_string = :k FOR UPDATE"),
        {"k": card_key_str},
    ).fetchone()
    if not card:
        log_action(db, None, None, "failed_attempt", ip, user_agent, reason="not_found")
        raise_error("E102")

    # 3. 状态校验
    if card.status == "used":
        log_action(db, card.id, user_id, "failed_attempt", ip, user_agent, reason="already_used")
        raise_error("E102", "卡密已被使用")
    if card.status == "expired":
        log_action(db, card.id, user_id, "failed_attempt", ip, user_agent, reason="expired")
        raise_error("E103")
    if card.status == "revoked":
        log_action(db, card.id, user_id, "failed_attempt", ip, user_agent, reason="revoked")
        raise_error("E104")

    # 3b. 批次状态校验(激活 E106,关联 §7.6.2 CardKeyBatch)
    if card.batch_id:
        batch = db.execute(text("SELECT status FROM card_key_batches WHERE batch_id=:b"), {"b": card.batch_id}).fetchone()
        if batch and batch.status == "stopped":
            log_action(db, card.id, user_id, "failed_attempt", ip, user_agent, reason="batch_stopped")
            raise_error("E106")

    # 4. 有效期校验
    now = datetime.now(timezone.utc)
    # 4a. 生效起始校验(激活 E105:卡密未到有效期起)
    if card.valid_from and card.valid_from > now:
        log_action(db, card.id, user_id, "failed_attempt", ip, user_agent, reason="not_yet_valid")
        raise_error("E105")
    # 4b. 过期校验(原逻辑)
    if card.expires_at and card.expires_at < now:
        db.execute(text("UPDATE card_keys SET status='expired' WHERE id=:id"), {"id": str(card.id)})
        log_action(db, card.id, user_id, "expire", ip, user_agent)
        db.commit()
        raise_error("E103")

    # 5. 事务:卡密标记 used + 余额增加 + 落兑换日志
    balance = db.query(Balance).filter(Balance.user_id == user_id).with_for_update().first()
    if not balance:
        balance = Balance(user_id=user_id, credits=0, total_redeemed=0, total_consumed=0)
        db.add(balance); db.flush()

    credits_before = balance.credits
    balance.credits += card.credits
    balance.total_redeemed += card.credits

    db.execute(text(
        "UPDATE card_keys SET status='used', redeemed_by=:uid, redeemed_at=:now WHERE id=:id"
    ), {"uid": user_id, "now": datetime.now(timezone.utc), "id": str(card.id)})

    log_action(db, card.id, user_id, "redeem", ip, user_agent,
               credits_before=credits_before, credits_after=balance.credits)

    db.commit()

    return {
        "success": True,
        "credits_added": card.credits,
        "balance": balance.credits,
        "package_type": card.package_type,
        "expires_at": card.expires_at.isoformat() if card.expires_at else None,
    }


def log_action(db, card_id, user_id, action, ip, user_agent, **extra):
    """每次操作落日志,即使失败也落"""
    db.execute(text("""
        INSERT INTO card_redeem_logs (id, card_key_id, user_id, action, ip, user_agent, credits_before, credits_after)
        VALUES (gen_random_uuid(), :cid, :uid, :action, :ip, :ua, :cb, :ca)
    """), {"cid": card_id, "uid": user_id, "action": action, "ip": ip, "ua": user_agent,
           "cb": extra.get("credits_before"), "ca": extra.get("credits_after")})
```

### 7.3 卡密管理后台

| 页面 | 路由 | 角色 |
|------|------|------|
| 卡密生成 | /admin/cards/generate | admin |
| 卡密列表 | /admin/cards | admin |
| 卡密详情 | /admin/cards/:id | admin |
| 卡密作废 | (列表内) | admin |
| 卡密导出 | /admin/cards/export | admin |
| 卡密统计 | /admin/cards/stats | admin |
| 闲鱼订单关联 | /admin/cards/bind | admin |

**管理员鉴权**:

```python
# backend/app/utils/security.py 追加
def get_current_admin(user: User = Depends(get_current_user)) -> User:
    """要求管理员权限"""
    if user.username not in settings.ADMIN_USERNAMES:
        raise_error("E407")
    return user
```

`ADMIN_USERNAMES` 从 `.env` 读取(逗号分隔),不入库(简化起步)。

### 7.4 卡密生成脚本增强(已有 `scripts/generate_cards.py`)

```bash
# 现有:CLI 批量生成 → CSV + SQL
python scripts/generate_cards.py --package monthly_100 --count 50 --batch xianyu-2026-07

# 新增:直接写数据库(避免手动导入 SQL)
python scripts/generate_cards.py --package monthly_100 --count 50 --batch xianyu-2026-07 --db

# 新增:从闲鱼订单 CSV 批量发货
python scripts/generate_cards.py --from-orders orders.csv --batch xianyu-2026-07 --db --output delivery.csv
# → 读 orders.csv(闲鱼订单号),每个订单生成一张卡密,关联 external_order_no,输出 delivery.csv 用于自动发卡
```

### 7.5 关键文件清单

| 文件 | 任务 |
|------|------|
| `backend/app/modules/card_key/service.py` | 生成/兑换/作废/统计 |
| `backend/app/modules/card_key/api.py` | 路由(用户兑换 + 管理员) |
| `backend/app/modules/card_key/open_api.py` | **新增:对外卡密分发 Open API(§7.6)** |
| `backend/app/modules/card_key/auth.py` | **新增:API Key + HMAC 鉴权中间件(§7.6.3)** |
| `backend/app/modules/card_key/schemas.py` | Pydantic DTO |
| `backend/app/services/card_cleanup.py` | 新增:过期卡密定时清理 |
| `scripts/generate_cards.py` | 增强:--db 直写 + --from-orders 闲鱼发货 |
| `frontend/src/pages/admin/CardKeys.vue` | 新增 |
| `frontend/src/pages/redeem/Redeem.vue` | 改造(舞台模式,按 §3.6) |
| `frontend/src/components/card-key/CardKeyInput.vue` | 新增(实时校验 + 自动大写) |

### 7.6 对外卡密分发 Open API(供外部卡密管理系统对接)

> **背景**:卡密在**本站后台生成**,另有**专门的外部卡密分发/管理系统**负责售卖与运营,需要从本站拉取卡密、回传核销、查询状态。本节定义站点对外暴露的机器对机器(M2M)Open API,与 §4.1 的 `admin/*`(人类管理员 JWT 鉴权)严格区分。

#### 7.6.1 设计原则与状态同步模型

- **权威边界**:站点是卡密**生成与兑换余额的权威源**;外部系统是**售卖/分发的权威源**。避免余额双写。
- **同步方向**:站点单向**对外发放**(claim);外部系统卖出后**回传核销**(redeem-notify)标记站点卡密 used。余额到账仍由用户在站点 `/redeem` 闭环完成(§7.2),外部系统不直接改余额。
- **生成归站点后台**:卡密始终在站点 `/admin/cards/generate` 或 `scripts/generate_cards.py` 生成;外部系统**只读申领**,不生成。
- **幂等**:所有写接口必带 `Idempotency-Key`(UUID),同 key 重复请求返回首次结果,不二次发卡/核销。

#### 7.6.2 新增数据模型(2 张表)

```python
class CardKeyDistributor(Base):
    """外部卡密分发系统凭证(M2M 鉴权)"""
    __tablename__ = "card_key_distributors"
    id = Column(UUID, primary_key=True, default=uuid.uuid4)  # server_default=gen_random_uuid()(错题集 #003)
    name = Column(String(64), unique=True, nullable=False)           # "闲鱼发卡机器人" / "自有分发系统"
    api_key = Column(String(48), unique=True, index=True, nullable=False)   # 公开标识, dk_xxx
    api_secret_hash = Column(String(128), nullable=False)            # 存哈希不存明文(argon2/bcrypt)
    permissions = Column(JSONB, default=list)                        # ["claim","query","redeem_notify","revoke","batch_list"]
    ip_whitelist = Column(JSONB, default=list)                       # ["1.2.3.4/32"], 空则不限
    rate_limit_per_min = Column(Integer, default=60)
    revoked_at = Column(DateTime(timezone=True), nullable=True)       # 吊销时间,非空=已禁用
    last_used_at = Column(DateTime(timezone=True), nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class CardKeyBatch(Base):
    """卡密批次(补全批次实体,激活 E106 批次停用 + 申领维度)"""
    __tablename__ = "card_key_batches"
    batch_id = Column(String(64), primary_key=True)                  # "xianyu-2026-07" / "open-2026-07-16"
    package_type = Column(String(32), nullable=False)                # monthly_100 / single_50 ...
    count = Column(Integer, nullable=False)                          # 本批计划生成数
    status = Column(String(16), default="active", nullable=False)    # active / stopped(停用,激活 E106)
    valid_from = Column(DateTime(timezone=True), nullable=True)      # 生效起始(激活 E105)
    valid_days = Column(Integer, nullable=True)                     # 有效天数(算 expires_at)
    created_by = Column(String(64), nullable=True)                   # admin 用户名 / distributor name
    note = Column(String(256), nullable=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

**关联**:card_keys 追加 `batch_id`(FK→card_key_batches)与 `valid_from`(单卡生效起始,激活 §7.2 的 E105 校验)。

#### 7.6.3 鉴权机制(API Key + HMAC-SHA256 签名 + 时间戳防重放)

```python
# backend/app/modules/card_key/auth.py
# 请求头(每次必带):
#   X-Api-Key: dk_xxxxxxxxxxxxxxxx
#   X-Timestamp: 1752768000            (Unix 秒, 与服务端时钟差 ≤ 300s, 否则 E501)
#   X-Nonce: <uuid>                    (5 分钟内不可重复,防重放,Redis 存 openapi:nonce:{key}:{nonce},TTL=300s)
#   X-Signature: hex(HMAC_SHA256(api_secret, f"{method}\n{path}\n{timestamp}\n{nonce}\n{body_sha256}"))

async def verify_distributor(request: Request) -> CardKeyDistributor:
    api_key = request.headers.get("X-Api-Key")
    dist = db.query(CardKeyDistributor).filter_by(api_key=api_key, revoked_at=None).first()
    if not dist: raise_error("E502")                              # 未知分发方
    # IP 白名单
    if dist.ip_whitelist and client_ip not in dist.ip_whitelist: raise_error("E503")
    # 时间戳
    if abs(now - int(ts)) > 300: raise_error("E501")
    # nonce 防重放(Redis SETNX)
    if not redis.set(f"openapi:nonce:{api_key}:{nonce}", 1, nx=True, ex=300): raise_error("E501", "重放请求")
    # HMAC 验签(常量时间比较,防时序攻击)
    expected = hmac_sha256(dist.api_secret, signing_string(request))
    if not hmac.compare_digest(expected, signature): raise_error("E502", "签名错误")
    # 权限校验
    if action not in dist.permissions: raise_error("E504")
    # 更新 last_used_at(异步,不阻塞)
    return dist
```

**踩坑预防**:
- api_secret **只存哈希**,明文仅在生成时一次性展示给外部系统;丢失需重置(rotate),不可找回。
- 验签用 `hmac.compare_digest`(常量时间),禁止 `==` 比较(防时序侧信道)。
- nonce 必须存 Redis 且带 TTL,重启不丢(Redis AOF);时钟漂移容差 300s。
- 签名串含 `body_sha256`,防 body 篡改;GET 请求 body 为空串。

#### 7.6.4 对外 API 清单(独立前缀 `/api/open/v1/`)

| 方法 | 路径 | 权限 | 用途 | 幂等键 |
|------|------|------|------|--------|
| POST | `/api/open/v1/card-keys/claim` | claim | 外部系统**申领一批卡密**(站点后台已生成,按 batch+count 发放) | Idempotency-Key |
| GET | `/api/open/v1/card-keys/{key_string}` | query | 查询单张卡密状态(unused/used/expired/revoked + 关联订单) | - |
| POST | `/api/open/v1/card-keys/redeem-notify` | redeem_notify | 外部系统卖出后**回传核销**:标记卡密 used + 关联 external_order(不直接改余额) | Idempotency-Key |
| POST | `/api/open/v1/card-keys/revoke` | revoke | 外部系统作废卡密(退款联动) | Idempotency-Key |
| GET | `/api/open/v1/batches?status=active` | batch_list | 查可申领批次(批次+剩余可用数) | - |

**claim 响应示例**:
```json
{
  "batch_id": "open-2026-07-16", "claimed": 50,
  "card_keys": [{"key_string": "ZN-ABCD-1234-EFGH", "package_type": "monthly_100",
                 "credits": 100, "expires_at": "2026-08-16T00:00:00Z", "valid_from": null}],
  "idempotency_key": "...", "claimed_at": "2026-07-16T01:52:32Z"
}
```

**redeem-notify 语义(关键,与 §7.2 区分)**:
- 外部系统回传核销**仅标记卡密状态 + 关联 external_order_no + 落 card_redeem_logs(action='external_redeem')**,**不动用户余额**(外部系统无 user 上下文)。
- 用户持卡密到站点 `/redeem` 时:若卡密已被外部 redeem-notify 标记 used,则 §7.2 兑换应**报 E102 已使用**(防止外部已卖+站内再兑的双重使用)。
- **并发安全**:redeem-notify 与站内 redeem 共用 `card_keys FOR UPDATE` 行锁 + 状态机,谁先拿到锁谁先置 used,另一方按已用处理,不会双发。

#### 7.6.5 Open API 错误码增量

| 错误码 | 描述 | 用户消息 | HTTP |
|--------|------|---------|------|
| E501 | Open API 请求过期/重放 | 请求时间戳无效或重复,请重试 | 401 |
| E502 | Open API 鉴权失败 | 鉴权失败(API Key/签名错误) | 401 |
| E503 | IP 未授权 | 该 IP 无权访问 | 403 |
| E504 | Open API 权限不足 | 该分发方无此接口权限 | 403 |
| E505 | 批次无可申领卡密 | 该批次卡密已申领完 | 409 |
| E506 | 幂等冲突 | 重复请求(返回首次结果) | 200 |

**E506 设计**:幂等键命中时**返回 200 + 首次结果**(非错误),便于外部系统重试安全;真正失败用 E5xx。

#### 7.6.6 与 §7.2 站内兑换的一致性约束

- 卡密状态机(§7.1)对外部核销同样适用:外部 redeem-notify 只能 `unused→used`,不可逆。
- 卡密兑换余额**始终在站内 `/redeem` 闭环**;外部系统**不持有也不修改** `balances` 表(避免双写冲突,呼应 §7.6.1 权威边界)。
- 若业务需要"外部卖出即到账"(跳过站内兑换),需另行设计(本期不做,记为 TODO)。

#### 7.6.7 上线前检查(Open API 专属)

- [ ] api_secret 仅存哈希,明文不入库不入 git
- [ ] 所有 `/api/open/v1/*` 必经 `verify_distributor` 中间件,无白名单接口
- [ ] nonce 去重走 Redis(非内存),多实例一致
- [ ] claim/redeem-notify/revoke 必校验 Idempotency-Key
- [ ] 分发方可吊销(revoked_at)+ 可轮换密钥(rotate)
- [ ] Open API 限流(per distributor per_min),超限 E501 风格 429
- [ ] 所有 Open API 操作落 card_redeem_logs(action 标 external_*,含 distributor name)

---

## 八、前端页面清单(完整)

### 8.1 改造现有(6 页)

| 页面 | 改造点 |
|------|--------|
| Login.vue | 3 Tab(账号/微信/短信),按 §3.6 舞台模式 |
| Register.vue | 独立页 + 邮箱验证码 + 防刷 |
| Redeem.vue | 改造为舞台模式,顶部结果卡片,底部输入 |
| UserCenter.vue | 5 Tab(概览/订单/卡密/任务/资料) |
| Pricing.vue | 保留闲鱼入口 + 加"在线购买"按钮(P1)→ 跳 /payment/checkout?pkg=xxx |
| Home.vue | 顶部加"我的余额"入口(已登录时) |

### 8.2 新增页面(11 页)

| 页面 | 路径 | 模块 |
|------|------|------|
| ForgotPassword.vue | /forgot-password | A |
| WechatLogin.vue | /login/wechat | A |
| Profile.vue | /user/profile | A |
| Sessions.vue | /user/sessions | A |
| Checkout.vue | /payment/checkout | B |
| Orders.vue | /payment/orders | B |
| OrderDetail.vue | /payment/order/:no | B |
| admin/Dashboard.vue | /admin | C/B |
| admin/CardKeys.vue | /admin/cards | C |
| admin/Orders.vue | /admin/orders | B |
| admin/Reconcile.vue | /admin/reconcile | B |

### 8.3 路由更新

`frontend/src/router/index.ts` 追加:

```typescript
const staticRoutes = [
  // 已有
  { path: '/', component: () => import('@/pages/Home.vue') },
  { path: '/redeem', component: () => import('@/pages/redeem/Redeem.vue') },
  { path: '/login', component: () => import('@/pages/auth/Login.vue') },
  { path: '/register', component: () => import('@/pages/auth/Register.vue') },
  { path: '/user-center', component: () => import('@/pages/user/UserCenter.vue') },
  { path: '/pricing', component: () => import('@/pages/Pricing.vue') },
  // 新增
  { path: '/forgot-password', component: () => import('@/pages/auth/ForgotPassword.vue') },
  { path: '/login/wechat', component: () => import('@/pages/auth/WechatLogin.vue') },
  { path: '/user/profile', component: () => import('@/pages/user/Profile.vue'), meta: { requiresAuth: true } },
  { path: '/user/sessions', component: () => import('@/pages/user/Sessions.vue'), meta: { requiresAuth: true } },
  { path: '/payment/checkout', component: () => import('@/pages/payment/Checkout.vue'), meta: { requiresAuth: true } },
  { path: '/payment/orders', component: () => import('@/pages/payment/Orders.vue'), meta: { requiresAuth: true } },
  { path: '/payment/order/:no', component: () => import('@/pages/payment/OrderDetail.vue'), meta: { requiresAuth: true } },
  // 管理员
  { path: '/admin', component: () => import('@/pages/admin/Dashboard.vue'), meta: { requiresAdmin: true } },
  { path: '/admin/cards', component: () => import('@/pages/admin/CardKeys.vue'), meta: { requiresAdmin: true } },
  { path: '/admin/orders', component: () => import('@/pages/admin/Orders.vue'), meta: { requiresAdmin: true } },
  { path: '/admin/reconcile', component: () => import('@/pages/admin/Reconcile.vue'), meta: { requiresAdmin: true } },
]

// 路由守卫:requiresAuth / requiresAdmin
router.beforeEach((to, from, next) => {
  const userStore = useUserStore()
  if (to.meta.requiresAuth && !userStore.isLoggedIn) {
    next({ path: '/login', query: { redirect: to.fullPath } })
  } else if (to.meta.requiresAdmin && !userStore.isAdmin) {
    next('/user-center')
  } else {
    next()
  }
})
```

---

## 九、分阶段落地路线图

### 9.1 P0:MVP 闭环(1 周)

**目标**:闲鱼线下收款(永久渠道)+ 卡密闭环的最小可用版本。闲鱼作为第一收款方式,不因后续上线在线支付而下线。

| 任务 | 文件 | 工时 |
|------|------|------|
| 迁移 0002:user_sessions + email_verifications | alembic | 0.5d |
| 迁移 0003:payment_logs + orders 字段 | alembic | 0.5d |
| 迁移 0004:card_redeem_logs | alembic | 0.5d |
| 改造 Login.vue(账号密码 + 邮箱注册验证码) | frontend | 1d |
| 改造 UserCenter.vue(5 Tab,概览+订单+卡密+任务+资料) | frontend | 2d |
| 改造 Redeem.vue(舞台模式) | frontend | 0.5d |
| 改造 Pricing.vue(保留闲鱼入口 + 卡密兑换引导) | frontend | 0.5d |
| 闲鱼渠道接入(orders channel=xianyu + 管理员手动确认收款 + 录单关联卡密) | backend + admin | 1d |
| 卡密管理后台基础版(CardKeys 列表 + 生成 + 导出) | frontend + backend | 2d |
| 卡密生成脚本增强(--db 直写) | scripts | 0.5d |
| 文档:36-user-payment-cardkey-modules.md(本文档) | docs | 0.5d |

**验收**:
- 注册→登录→卡密兑换→余额到账→使用 PDF 转 Word 跑通
- 管理员能批量生成卡密并导出 CSV
- 闲鱼卖家用 CSV 给买家发货,买家在 /redeem 兑换
- 管理员能在后台录入闲鱼订单(channel=xianyu)并手动确认收款,状态 pending→paid→delivered,卡密自动关联
- 后台 Dashboard 能区分显示闲鱼收入(在线支付 P1 上线后并入同表统计)

### 9.2 P1:在线支付(2 周)

**定位**:新增第二种收款方式(在线支付),与闲鱼并存,不下线闲鱼。两种方式共用 orders 表与发卡逻辑。

| 任务 | 工时 |
|------|------|
| 微信 Native 扫码渠道(下单 + 回调 + 验签 V3) | 3d |
| Checkout.vue(选套餐→选渠道→扫码→轮询) | 2d |
| OrderDetail.vue + Orders.vue | 1d |
| 订单超时清理定时任务(crontab) | 0.5d |
| 支付宝当面付(P1 备选,可推到 P2) | 2d |
| 沙箱测试 + 0.01 元真实测试 | 1d |
| 对账脚本 + 后台对账页 | 1d |
| 后台 Dashboard 收入按渠道分类(闲鱼 + 微信 + …) | 0.5d |

**验收**:
- 在线支付 → 自动发卡密 → 自动兑换 → 余额到账 全自动
- 30 分钟未付订单自动 expired
- 后台能看今日/本周/本月收入(按渠道分类:闲鱼 + 在线)

### 9.3 P2:用户系统增强(1 周)

| 任务 | 工时 |
|------|------|
| 找回密码(邮箱验证码) | 1d |
| 微信扫码登录 | 2d |
| 用户资料(头像/昵称) | 1d |
| 设备列表 + 强制下线 | 1d |
| 注销账号(软删 30 天) | 1d |
| 短信验证码登录(对接阿里云短信) | 2d |

### 9.4 P3:卡密运营增强(1 周)

| 任务 | 工时 |
|------|------|
| 卡密统计页(批次使用率/过期率) | 1d |
| 闲鱼订单 CSV 批量发货脚本 | 1d |
| 卡密过期定时清理 | 0.5d |
| 卡密作废 + 退款联动 | 1d |
| 卡密批次管理(批次停用) | 1d |

### 9.5 P4:对账与风控(1 周)

| 任务 | 工时 |
|------|------|
| 每日对账定时任务 + 异常告警 | 2d |
| 退款流程(申请→审核→执行) | 2d |
| 风控:同一 IP 短时多次兑换失败告警 | 1d |

### 9.6 P5:上线运营(持续)

- 真实 HTTPS + 域名 + ICP 备案
- 监控告警(Grafana + 企业微信机器人)
- 客服工单系统(可对接企微)
- 用户增长运营(签到送次数/邀请奖励)

### 9.7 总工时估算

| 阶段 | 工时 | 关键产出 |
|------|------|----------|
| P0 MVP | 9.5 人日 | 闲鱼线下收款(永久)+ 卡密闭环 |
| P1 在线支付 | 11 人日 | 第二种收款方式(在线),与闲鱼并存 |
| P2 用户系统 | 8 人日 | 完整账户体系 |
| P3 卡密运营 | 4.5 人日 | 运营效率 |
| P4 对账风控 | 5 人日 | 安全合规 |
| **合计** | **37.5 人日** | **6-8 周完成全部** |

---

## 十、踩坑预防(基于错题集)

### 10.1 数据库相关

| 雷区 | 预防 |
|------|------|
| UUID 主键无 server_default(错题集 #003) | 所有新表 UUID 主键必加 `server_default=sa.text("gen_random_uuid()")` |
| bcrypt 4.1+ 与 passlib 不兼容(错题集 #004) | `requirements.txt` 固定 `bcrypt==4.0.1` |
| Alembic 迁移修改表后忘更新 ORM(错题集 #007) | 迁移 + 模型 + OpenAPI 三方同步,CI 强制校验 |
| FOR UPDATE 行锁死锁(错题集 #011) | 卡密兑换先锁 card_keys 再锁 balances,顺序统一 |

### 10.2 支付相关

| 雷区 | 预防 |
|------|------|
| 回调验签失败仍处理(导致伪造支付) | 必须先验签再处理,失败 raise E403,不写 payment_log |
| 回调重复(微信会重试) | `pay_order_success` 必幂等,已 paid 直接 return success |
| 订单金额篡改 | 创建订单时锁定 amount,回调不信任 amount 字段,以本地 orders.amount 为准 |
| 回调超时(微信要求 5 秒) | 验签+解密+落库 < 3 秒,主动查询放异步队列 |
| 微信 V3 用 RSA-SHA256 不是 MD5 | 不要复制旧版微信支付代码 |
| 商户证书泄露 | PEM 文件放 Docker secrets,不入 git,不入 .env |
| 测试用 0.01 元 | 上线前必跑小额真实测试,沙箱环境 ≠ 真实环境 |

### 10.3 卡密相关

| 雷区 | 预防 |
|------|------|
| 卡密被并发兑换两次 | `SELECT ... FOR UPDATE` 行锁(已实现) |
| 卡密日志丢失(只记成功不记失败) | log_action 在每个分支调用,即使失败也落日志 |
| 卡密明文存数据库 | key_string 是明文,但生成时用 secrets.choice(密码学安全随机),且只有管理员能看完整,API 脱敏返回 |
| 卡密格式被暴力枚举 | 3 段 × 4 位 = 36^12 ≈ 4.7e18 种,加上限流(同 IP 1 分钟 5 次),实际无法枚举 |
| 闲鱼订单号冲突 | external_order_no 加 unique 索引,重复关联报错 |

### 10.4 前端相关

| 雷区 | 预防 |
|------|------|
| JWT 存 localStorage 易受 XSS | httponly cookie 备选(但 SSR 不便);起步用 localStorage + 严格 CSP |
| 401 跳转死循环(错题集 #012) | 已在 api/client.ts 处理,登录页自身 401 不跳 |
| 余额不同步(多 Tab) | useBalanceSync 用 storage event 跨 Tab 同步 |
| SSG 构建期访问 localStorage | 所有 localStorage 访问必加 `typeof window !== 'undefined'` 守卫(已在 stores/user.ts 实现) |
| 设计规范违反(§3.6/§3.8) | 所有新页面遵守舞台模式 + 视觉重心 + 字号层级 |

### 10.5 一致性相关(遵守 §07)

| 雷区 | 预防 |
|------|------|
| API 变更未同步 OpenAPI | 先改 openapi.yaml,再改代码,跑 `npm run gen:api` |
| 错误码不一致 | 新错误码必加到 `error_codes.py` + openapi.yaml |
| 工具注册表不同步 | 不涉及新工具,但卡密/支付套餐如未来要加 SSOT,放 `packages.py` 不放散落文件 |
| 文档与代码脱节 | 每次迭代更新本文件第十二章"迭代记录" |

---

## 十一、一致性检查清单(上线前必过)

### 11.1 后端

- [ ] alembic upgrade head 成功,无报错
- [ ] `python scripts/check_consistency.py` 通过
- [ ] OpenAPI 路径与代码路由一致(`diff openapi.yaml <(python -c "from app.main import app; import yaml; print(yaml.dump(app.openapi()))")`)
- [ ] 错误码完整(E101-E106, E301-E309, E401-E409)
- [ ] 所有新表 UUID 主键有 server_default
- [ ] bcrypt 版本固定 4.0.1
- [ ] 卡密兑换事务有 FOR UPDATE
- [ ] 支付回调先验签再处理
- [ ] 支付回调幂等(已 paid 直接 return)
- [ ] 管理员鉴权(get_current_admin)应用到所有 /admin/* 路由

### 11.2 前端

- [ ] `npm run build` 成功,无 error/warning
- [ ] `python scripts/verify_all.py` 通过
- [ ] 路由守卫(requiresAuth/requiresAdmin)生效
- [ ] 所有新页面遵守 §3.6 舞台模式 / §3.8 审美规范
- [ ] 所有 localStorage 访问有 SSR 守卫
- [ ] api/client.ts 与 openapi.yaml 一致
- [ ] Pinia store 持久化正常(token/username)

### 11.3 安全

- [ ] HTTPS 证书有效
- [ ] 回调地址在支付渠道后台配置正确
- [ ] .env 不入 git(`.gitignore` 含 .env)
- [ ] 商户证书放 Docker secrets
- [ ] JWT_SECRET 至少 32 字节随机
- [ ] CORS 仅允许前端域名
- [ ] Rate limit:登录 5 次/分钟,兑换 3 次/分钟,注册 3 次/小时

### 11.4 监控

- [ ] /metrics 含支付/兑换/登录指标
- [ ] Grafana 看板:今日订单数/支付成功率/兑换成功率/错误率
- [ ] 告警:回调失败率 > 1%,兑换失败率 > 5%

---

## 十二、迭代记录(每次更新本文件时追加)

### 12.1 v1.0(2026-07-15,本次)

- 初版设计完成
- 3 模块边界、目录结构、数据模型、API 契约、前端页面、路线图、踩坑预防、检查清单
- 待 P0 落地后追加 v1.1 记录实际实现偏差

### 12.2 v1.1(2026-07-16,设计修订:多收款方式并存)

- **变更**:明确站内保留 2 种以上收款方式,闲鱼作为永久渠道,不因上线在线支付而下线
- **新增**:§6.0 收款方式总览表、闲鱼 `xianyu` 手动渠道(XianyuManualChannel)、§6.3 管理员手动确认发卡、§4.1 `/confirm-paid` 接口
- **修订**:§2.1 模块关系补闲鱼路径;§9.1 P0 闲鱼由"验证期"改"永久渠道"+录单手动确认;§9.2 P1 定位为"第二种收款方式";§9.7 工时 36.5→37.5 人日
- **决策**:闲鱼订单进 orders 表统一统计(channel=xianyu),与在线支付共用 pay_order_success 发卡,对账分开(用户确认)
- **下一步**:P0 落地后追加 v1.3 记录实际实现偏差

### 12.3 v1.2(2026-07-16,设计修订:闭环加固 + 对外卡密 API)

- **新增 #1 对外卡密分发 Open API**(用户需求:卡密在站点后台生成,另有专门卡密分发系统对接):
  - §7.6 新增整节:设计原则(站点=生成/兑换权威,外部=售卖权威,避免余额双写)、2 张新表(`card_key_distributors`/`card_key_batches`)、API Key+HMAC+nonce 防重放鉴权(§7.6.3)、5 个 `/api/open/v1/*` 接口(claim/query/redeem-notify/revoke/batch_list)、与 §7.2 站内兑换的一致性约束、E501-E506 错误码、上线检查清单
  - 迁移加 0005_open_api.py;§7.5 关键文件清单加 open_api.py/auth.py
- **修订 #2 订单状态机闭环**(§6.3):
  - 明确 delivered 语义=「卡密就绪可交付」(非冗余,用于对账区分 paid 未发卡 vs delivered 已发卡),给出真实驱动
  - 补退款↔卡密联动事务(防双重得利):unused 卡密→作废 revoke;used 卡密→校验余额充足才扣,余额已消耗→E507 人工
  - 幂等判定扩为 `status in ('paid','delivered')`
  - 前端轮询超时主动调 cancel(§6.4,防订单悬空+重复下单)
- **修订 #3 卡密兑换校验**(§7.2):
  - 第 3b 步补批次状态校验(激活 E106 stopped)
  - 第 4a 步补 valid_from 生效起始校验(激活 E105 未到有效期起)
  - 激活原"死错误码"E105/E106
- **错误码**:加 E507(退款卡密已消耗),注明 E501-E506 见 §7.6.5
- **下一步**:P0 落地后追加 v1.3 记录实际实现偏差

### 12.4 v1.x(后续模板,实际更新时按此格式)

```markdown
### v1.x(YYYY-MM-DD,XXX)

- **完成**:
  - [ ] P0 任务列表(勾选完成的)
  - [ ] ...
- **偏差**:
  - 设计 X,实际改为 Y,原因:...
- **新踩坑**:
  - 问题 N:现象 / 根因 / 修复 / 预防
- **下一步**:
  - P1 任务清单
```

---

## 十三、相关文档

- [03-backend-architecture.md](./03-backend-architecture.md) — 后端架构
- [04-api-contract.md](./04-api-contract.md) — API 契约
- [07-consistency-link.md](./07-consistency-link.md) — 一致性机制
- [16-payment-integration.md](./16-payment-integration.md) — 支付对接(已部分覆盖)
- [17-quota-strategy.md](./17-quota-strategy.md) — 配额策略
- [27-lessons-learned.md](./27-lessons-learned.md) — 错题集
- [32-ui-design-spec.md](./32-ui-design-spec.md) — 产品设计规范 v2.2
- [01-progress.md](./01-progress.md) — 迭代进度(每次 P 完成更新)
- [CLAUDE.md](./CLAUDE.md) — AI 协作上下文

---

## 十四、使用本文件的方式

| 角色 | 用法 |
|------|------|
| AI 接手 | 必读本文件 → 看进度 01-progress.md → 按 P0 任务清单实施 |
| 后端开发 | 看第三/四/五/六/七章 → 改 alembic → 改 modules/ → 跑 verify_all |
| 前端开发 | 看第八章 → 改 router → 新建 pages/components → 跑 build |
| PM | 看第九章 → 按 P0-P5 排期 → 验收按 11 章清单 |
| 运营 | 看第七章 → 用卡密管理后台发货 → 看对账页 |
| 风控 | 看第十章 → 配置告警 → 看监控 |

**本文件是 SSOT,所有变更先改本文件,再改代码。**
