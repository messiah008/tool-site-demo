# 在线支付改造设计（易用性 / 可靠性 / 安全性 / 合规）

> **版本**: v1.0
> **最后更新**: 2026-07-20
> **状态**: 设计完成,分阶段落地
> **依赖**: [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md)(契约层 SSOT)、[32-ui-design-spec.md](./32-ui-design-spec.md)(§3.6 舞台模式)、[07-consistency-link.md](./07-consistency-link.md)(SSOT)、[38-placeholder-integration-checklist.md](./38-placeholder-integration-checklist.md)(占位接入)、[27-lessons-learned.md](./27-lessons-learned.md)(#036/#037 微信 V3 踩坑)
> **取代**: 无(补 36 号 §6 未落地部分)
>
> **本文是"在线支付"子系统的落地层 SSOT**。36 号是契约层(已定义 cancel/自动兑换/超时清理),本文诊断现状缺口 + 文件级实现。四主线:**易用性首要(出二维码)/ 可靠性 / 安全性 / 合规(本地规范)**。

---

## 一、现状诊断

### 1.1 易用性硬伤(首要)

| 硬伤 | 文件:行 | 说明 |
|------|---------|------|
| **不出二维码** | `Checkout.vue:37,64-69` | 不引 qrcode 库,`qr_data`(weixin://wxpay/bizpayurl?pr=xxx)当纯文本截断 40 字符,无法扫码。项目已有 `qrcode` 库 + `qrcode-generator` 工具页可复用 |
| pay_url 丢弃 | `Checkout.vue:77` | 解构未取 payUrl,虎皮椒/PayJS 跳转 URL 被丢 |
| Pricing 跳转丢上下文 | `Pricing.vue:15-16` | 裸路径 `/payment/checkout` 不传套餐,需重选 |
| 套餐 id 不一致 | `Pricing.vue:47-49` | 用 `free/monthly/yearly`,后端 `packages.py` 用 `single_50/monthly_100/yearly_unlimited` |
| 无 order store | `stores/`(仅 user.ts) | 刷新/离开丢失进行中订单 |
| 超时不调 cancel | `useOrderPolling.ts:32-37` | TODO 未实现,只 stop 不调后端 |
| 轮询失败静默吞 | `useOrderPolling.ts:45-47` | catch{} 空体,无失败态 UI |
| 非舞台模式 | `Checkout.vue` 全文 | 未按 §3.6:无永远占位扫码舞台,线性切换,金额 28px(应 48px+),圆角 5 种(应≤3),硬编码 `#065f46`/`#047857` |
| 4 组件未抽取 | `components/`(无 payment/) | PackageCard/ChannelTabs/QrCode/OrderStatus 全内联 |

### 1.2 可靠性短板

| 短板 | 文件:行 | 说明 |
|------|---------|------|
| 超时清理全缺 | `main.py:19-28`、`order_service.py` | `expires_at` 列闲置(迁移0004 加了),无定时任务,无 expired 写入,pending 永久悬存 |
| cancel 端点缺 | `payment.py`、`openapi.yaml` | 前后端 + openapi 全缺(36号§4.1 定义未实现) |
| checkout 异常悬空 | `payment.py:344-369` | 无 try/except,create_payment 失败时订单已 commit 悬空 pending |
| 不自动兑换 | `order_service.py:95-144` | pay_order_success 只发卡(unused)不调 redeem_service,需手动 /redeem |
| 回调不落 payment_log | `order_service.py` | 在线回调成功无 INSERT payment_logs,影响对账 |
| 轮询触发 429 | `nginx.conf:118` | 3s/次×30min=600次/单,走 api_limit 60r/m,正常轮询即 429 |
| Order ORM 缺列 | `models/order.py:8-22` | 迁移0004 加了 pay_url/expires_at/refunded_at/refund_amount,ORM 未同步,靠动态属性 |
| 退款占位 | `wechat_native.py:273-279` | refund 未调 V3 退款 API(站内 refunded 但微信侧未退) |

### 1.3 安全性短板

| 短板 | 文件:行 | 说明 |
|------|---------|------|
| 微信验签放行 | `wechat_native.py:185-186,199-201` | 未配公钥/证书时 `return True` 放行,伪造回调可发卡 |
| 虎皮椒/PayJS TODO secret | `payment.py:563,571` | `TODO_XUNHUPAY_SECRET`/`TODO_PAYJS_SECRET` 硬编码占位,验签形同虚设 |
| 无金额校验 | `payment.py:136-185` | 虎皮椒/PayJS 回调只看状态不校验金额 |
| secret 降级 | `xunhupay.py:58`、`payjs.py:51` | `getattr(settings,X,"TODO_...")` 空配降级公开常量,不 fail-fast |
| 回调走限流 | `nginx.conf:118` | 回调走 api_limit,微信重试可能被 429 |
| 基类契约不一致 | `base.py:29` | `verify_callback(request)` 同步 vs 子类 async `verify_callback_data`,基类形同虚设 |
| 重复验签 | `payment.py:560-574` | 内联 `_verify_*_sign` 与渠道适配器 `verify_callback_data` 两套 |

### 1.4 可复用现有

| 资产 | 位置 | 用途 |
|------|------|------|
| qrcode 库 + QRCode.toDataURL→img | `tools/qrcode-generator/index.vue:63,80` | Checkout 二维码渲染直接复用 |
| useOrderPolling composable | `composables/useOrderPolling.ts` | 轮询骨架已有,补 cancel/降频/失败态 |
| 渠道适配器 | `modules/payment/channels/{base,wechat_native,xunhupay,payjs}.py` | 策略模式,补 fail-fast+金额 |
| pay_order_success / refund_order | `services/order_service.py` | 状态机+退款↔卡密联动已实现,补自动兑换+payment_log |
| redeem_service.redeem | `modules/card_key/service.py` | 行锁+幂等(used 抛 E102),自动兑换复用 |
| CSS 变量 | `assets/styles/tokens.css` | --color-*/--radius-*/--shadow-* 齐全 |
| user.ts store 模式 | `stores/user.ts` | Pinia + localStorage + SSR 守卫 |

---

## 二、设计目标

| 维度 | 目标 |
|------|------|
| **易用性(首要)** | 点支付立刻出真二维码;pay_url 渠道一键跳转;套餐一处定义对齐;刷新不丢订单;支付结果即时反馈;§3.6 舞台模式 |
| **可靠性** | 在线支付自动到账(登录用户);30 分钟超时自动 expired;支付发起失败不悬空;回调必落 payment_log;轮询不 429 |
| **安全性** | 验签 fail-fast(未配密钥拒绝而非放行);回调不信任 amount 以本地为准;回调白名单不限流;secret 配置缺失不降级 |
| **合规** | SSOT 先行(openapi/design-tokens);舞台模式 §3.6;CSS 变量不硬编码;模块化(modules/payment 三件套 + 4 前端组件);错误码体系;支付硬约束(36号§6/§10.2) |

---

## 三、整体架构

```
┌─ 前端 ──────────────────────────────────────────────┐
│ pages/payment/Checkout.vue(§3.6 舞台模式)            │
│  ├ components/payment/QrCode.vue(qrcode→img)        │
│  ├ components/payment/PackageCard.vue              │
│  ├ components/payment/ChannelTabs.vue              │
│  └ components/payment/OrderStatus.vue              │
│ composables/useOrderPolling.ts(cancel+降频+失败态) │
│ composables/useBalanceSync.ts(余额刷新)            │
│ stores/order.ts(进行中订单持久化)                   │
└──────────────────────────────────────────────────────┘
          │ checkout / getOrderStatus / cancelOrder
┌─ 后端 ──────────────────────────────────────────────┐
│ app/api/payment.py(旧路由,保留兼容,checkout 兜底)  │
│ modules/payment/api.py(新增 cancel 端点)            │
│ modules/payment/service.py(cancel/超时清理/自动兑换)│
│ modules/payment/schemas.py                          │
│ modules/payment/channels/(适配器,补 fail-fast+金额) │
│ services/order_service.py(自动兑换+payment_log+过期)│
│ main.py(lifespan 超时清理后台任务)                  │
└──────────────────────────────────────────────────────┘
```

**迁移原则**(36号§2.2):新增端点(cancel/超时清理)放 `modules/payment/`,旧 `app/api/payment.py` 保留兼容,渐进重定向,不破坏现有。

---

## 四、模块划分

### 4.1 后端 modules/payment/

```
modules/payment/
├── channels/           # 保留(base/wechat_native/xunhupay/payjs),补 fail-fast+金额校验
├── service.py          # 新增:cancel_order / expire_pending_orders / auto_redeem
├── api.py              # 新增:POST /orders/{no}/cancel(router 自带 /api/payment prefix)
└── schemas.py          # 新增:CancelResponse / ExpireResult
```

### 4.2 前端

```
components/payment/
├── QrCode.vue          # 新增:QRCode.toDataURL→img + 倒计时 + payUrl 跳转按钮
├── PackageCard.vue     # 新增:套餐卡(选中态/价格/次数/描述)
├── ChannelTabs.vue     # 新增:渠道选择(wechat_native/xunhupay/payjs)
└── OrderStatus.vue     # 新增:状态展示(pending/paid/expired/cancelled/失败)
stores/order.ts         # 新增:进行中订单持久化(localStorage)
composables/useBalanceSync.ts  # 新增:支付成功后刷新余额
```

---

## 五、接口契约增量(SSOT 先行)

### 5.1 新增 `POST /api/payment/orders/{order_no}/cancel`

```yaml
/api/payment/orders/{order_no}/cancel:
  post:
    summary: 用户主动取消未支付订单
    security: [BearerAuth]
    responses:
      '200': { description: 已取消, content: { application/json: { schema: { type: object, properties: { order_no: {type: string}, status: {type: string} } } } } }
      '403': { description: E403 无权/E405 已支付, content: { application/json: { schema: { $ref: '#/components/schemas/Error' } } }
      '404': { description: E404 订单不存在, content: { application/json: { schema: { $ref: '#/components/schemas/Error' } } }
```

### 5.2 错误码增量

| 错误码 | 描述 | 用户消息 | HTTP | retryable |
|--------|------|---------|------|-----------|
| E405 | 订单已支付不可取消 | 订单已支付或已交付,无法取消 | 409 | false |

加到 `error_codes.py` + openapi Error schema。

---

## 六、前端舞台模式设计(32号§3.6)

```
┌─────────────────────────────────────────┐
│ 扫码舞台(永远占位,min-height 320px)      │  ← 视觉焦点,顶部
│  ├ 空状态:大💳 56px+"选择套餐开始支付"   │
│  ├ pending:QrCode 组件+金额 48px 700 渐变│
│  │         +倒计时(--color-warning)       │
│  ├ paid/delivered:✅+卡密+余额(成功渐入) │
│  ├ expired:⏰+重新购买                    │
│  └ cancelled/失败:提示+重试              │
├─────────────────────────────────────────┤
│ 配置区(下方)                            │
│  ├ PackageCard 行(3 套餐)               │
│  ├ ChannelTabs(微信/虎皮椒/PayJS)       │
│  └ 立即支付按钮(主色渐变)               │
└─────────────────────────────────────────┘
```

- 金额 48px+ 700 `--color-primary`→`--color-primary-light` 渐变(§3.8.4)
- 圆角 ≤3 种(`--radius-md` 8px / `--radius-lg` 12px / `--radius-xl` 16px)
- 结果渐入 scale+opacity 0.3-0.5s(§3.8.6)
- 空状态大图标比按钮显眼(§3.6)
- 全用 CSS 变量,删 `#065f46`/`#047857` 硬编码

---

## 七、订单状态机(对齐 36号§6.3)

```
pending ──(支付成功,回调)──→ paid/delivered(同事务,卡密就绪 + 自动兑换登录用户)
   │
   ├─(30 分钟未付,定时任务)──→ expired
   ├─(用户主动 cancel 端点)──→ cancelled
   └─(支付发起失败,checkout 兜底)──→ cancelled

paid/delivered ──(退款)──→ refunded(退款↔卡密联动,已有 refund_order)
```

- pending→delivered:`pay_order_success` 同事务生成卡密 + 关联 + (登录用户)自动兑换
- 自动兑换后 delivered 不变(交付=卡密就绪,兑换=余额到账,两件事,36号:780)
- 闲鱼单 user_id=None 不自动兑换,保持手动 /redeem

---

## 八、安全设计

1. **验签 fail-fast**:微信未配公钥/证书 → raise E403(非 return True);虎皮椒/PayJS secret 为空 → raise E403(非 TODO 常量)
2. **回调统一走适配器**:删 `payment.py` 内联 `_verify_*_sign`,全渠道走 `verify_callback_data`
3. **金额校验全渠道**:回调以本地 `order.amount` 为准(微信已有,虎皮椒/PayJS 补)
4. **回调白名单不限流**:nginx 独立 location,无 limit_req(防微信重试被 429)
5. **基类契约修正**:`base.py` 抽象方法统一为 `verify_callback_data(*args)->dict`,删/弃 `verify_callback(request)`
6. **回调红线**(36号§10.2):先验签后处理,失败 raise E403 不落 payment_log,5 秒内响应

---

## 九、可靠性设计

1. **自动兑换**:`pay_order_success` 生成卡密后,`user_id` 非空且 `channel != xianyu` → `auto_redeem` → `redeem_service.redeem`;E102 捕获当幂等;其他异常落 `card_redeem_logs(failed_attempt)` 不回滚(钱已到账),用户仍可手动兑换
2. **payment_log 落库**:回调成功必 INSERT(event=paid,channel,signature_valid,amount)
3. **超时清理**:lifespan 后台任务每 60s 扫 `expires_at<now AND status=pending AND channel IN(在线渠道)`→expired;create_order/checkout 写 `expires_at=now+30min`(闲鱼 None)
4. **checkout 异常兜底**:create_payment 失败→订单标 cancelled+落 log+raise E500(不悬空 pending)
5. **Order ORM 补列**:pay_url/expires_at/refunded_at/refund_amount 映射(对齐迁移0004)
6. **轮询优化**:nginx 独立 poll_limit 30r/m;前端 3s→5s + 指数退避封顶 15s + 失败态 UI;超时主动 cancelOrder

---

## 十、分阶段路线

### P0 易用性(首要,解"不出二维码")
- 4 组件(QrCode/PackageCard/ChannelTabs/OrderStatus)+ Checkout §3.6 重构 + Pricing 传参+套餐对齐 + order store + useOrderPolling 恢复

### P1 可靠性
- openapi cancel + E405 + modules/payment(schema/service/api) + main lifespan + order_service(自动兑换+payment_log+expires_at) + Order ORM + checkout 兜底 + client.cancelOrder + useOrderPolling(cancel+降频) + nginx 限流 + useBalanceSync

### P2 安全加固
- wechat fail-fast + xunhupay/payjs fail-fast+金额 + payment.py 回调走适配器 + base 契约 + config 告警

---

## 十一、验证清单

**自动化**:`verify_all.py`、`check_consistency.py`、`diagnose.py --latest`、`npm run build`、`py_compile`、容器内 import 测试

**端到端**:
1. 选套餐(Pricing→Checkout 预选)→点支付→**真二维码显示**
2. 公网就绪后:扫码→回调→卡密+余额自动到账(登录用户)→用户中心可见
3. 闲鱼单不自动兑换→手动 /redeem
4. 30 分钟不付→expired;主动取消→cancelled
5. 支付发起失败→cancelled 不悬空
6. 轮询 30 分钟不触发 429

**安全**:伪造回调 E403;金额篡改被拒;未配公钥 E403(非放行);secret 空 E403;微信重试不被 429

---

## 十二、风险与回滚

| 风险 | 缓解 |
|------|------|
| 自动兑换重复上分 | redeem_service 对 used 抛 E102 已幂等;auto_redeem 捕获 E102;pay_order_success 对已 delivered 幂等。失败落 failed_attempt log 不回滚 |
| 渐进迁移两套路由并存 | modules/payment/api.py 新增 cancel,旧 payment.py 不动,main include 新 router 不重复定义 |
| 轮询降频感知延迟 | 5s 可接受;指数退避封顶 15s;useBalanceSync 余额刷新作第二反馈 |
| 微信 fail-fast 后未配公钥致回调全失败 | 上线前确认 WECHAT_PUB_KEY_PATH;灰度先保留 return True+告警,公钥就绪再切 fail-fast |
| Order ORM 补列行为差异 | 迁移0004 列已存在,补映射只是显式化,verify_all 兜底 |

---

## 十三、定价方案重新设计(2026-07-20 落地)

### 13.1 4 档定价(诱饵+锚定+提价,人性定价)

保持 credits/valid_days 不变(保护闲鱼存量卡密),只提 price + 加 original_price(SSOT 划线原价):

| 档 | package_id | credits | 现价 | 划线原价 | 单次成本 | 角色 |
|---|---|---|---|---|---|---|
| 1 | free | 3次/天配额 | ¥0 | — | ¥0 | 引流 loss-leader |
| 2 | single_50 | 50 | **¥19** | ~~¥29~~ | ¥0.38/次 | **诱饵**(单次比月度贵→月度显值) |
| 3 | monthly_100 | 100 | **¥29** | ~~¥39~~ | ¥0.29/次 | **主推 popular**(性价比悬崖) |
| 4 | yearly_unlimited | 99999 | **¥199** | ~~¥299~~ | 不限 | **锚**(高价拉参照) |

人性定价:①诱饵(single ¥0.38/次 > monthly ¥0.29/次 让主推超值)②锚(yearly ¥199 让 monthly 显小钱)③分解定价(单次成本对比)④损失规避(划线原价+省额+限时倒计时)⑤默认推荐(monthly popular)。

### 13.2 计费文案对齐(消除 ¥2/¥5 脱节)

后端 convert.py 所有高质量工具固定扣 1 credit,前端文案统一"消耗1点"(原 ¥2/¥5/¥3 是脱节文案)。改 tools.registry.json(5 action_label + 4 faq + 2 seo)+ pdf-to-word/index.vue → gen:tools 重新生成。

### 13.3 引流增长机制(4 项)

1. **注册送 3 点**(auth.py register + IP 防刷 Redis 24h 限 3):新用户即体验高质量,拉首付费率
2. **锚定+限时优惠**(Pricing.vue 划线/省额/单次成本 + localStorage 48h 倒计时)
3. **支付后回工具**(OrderStatus "立即继续转换"→/pdf-to-word,补闭环断点)
4. **邀请/签到裂变**(modules/growth 新模块 + 迁移0008):
   - 邀请:username 作邀请码,注册带 invite_code 双方加点(邀请人+2/被邀+3 叠加注册送),自邀 blocked,UNIQUE(invitee_id)
   - 签到:每日 1 次,连续递增奖励(min(1+(streak-1)//3,5)),断签重置,UNIQUE(user_id,date) 防重
   - 端点:/api/growth/invite-info|checkin|checkin-status;UserCenter 加"福利"tab

### 13.4 SSOT 收敛

套餐价格从 3 处副本(packages.py/Pricing.vue/generate_cards.py)收敛到 1 处 SSOT(packages.py):Pricing.vue 改 paymentApi.getPackages() 拉取,generate_cards.py 改 from app.packages import 读 SSOT。

### 13.5 踩坑(记 27 号)

`_log_action(card_id=None)` 触发 card_redeem_logs.card_key_id NOT NULL 违反(表定义 NOT NULL 但函数设计允许 None,既有 bug,正常兑换格式错分支也会触发)。本次注册赠送/邀请/签到日志改为不落 card_redeem_logs(靠 invitation_records/daily_checkin/balances.total_redeemed 审计)。

---

## 十四、限时优惠双机制 + 低门槛按次套餐(2026-07-20 落地)

### 14.1 低门槛按次套餐(P0)
packages.py 加 2 档极低门槛引流,复用 credits 池零后端逻辑改动:
- trial_1:¥1/1次/30天(原价¥3,¥1.00/次)——1元试水
- pack_5:¥3/5次/30天(原价¥9,¥0.60/次)——体验后复购
阶梯:trial ¥1→pack_5 ¥0.6/次→single_50 ¥0.38/次→monthly ¥0.29/次。已有档 credits/valid_days 不变(保护闲鱼存量卡密)。

### 14.2 限时优惠双机制
- **机制A 后台激活**:新表 promotions(单活动覆盖一档)+ modules/promotion + admin/Promotions.vue。管理员配活动价+起止,全用户可见,过 end_at 自动失效。
- **机制B 新用户专享**:后端按 user.created_at 判定注册48h内,monthly_100 ¥29→¥19(主推档锁定客单价),需登录才享不可绕过。
- **优先级**:后台活动 > 新用户价 > 原价(不叠加)。

### 14.3 优惠真正生效(防绕过核心)
create_order 加 price_tier 参数(normal/promo/new_user),后端 `_resolve_amount` 按 get_active_promotion(活动状态)+ created_at(新用户状态)**重算实收 amount**,tier 仅是提示:
- 伪造 tier=promo 无活动→原价;tier=new_user 非新用户→原价
- 有活动即便 tier=normal→命中活动价(防漏收)
Order 加 price_tier 列持久化,pay_order_success 落 payment_log event=paid:{tier} 审计。

### 14.4 端到端验证
- 按次套餐:trial_1 ¥1→credits 1;pack_5 ¥3→credits 5 ✅
- 新用户:注册48h内→monthly ¥19→下单实收¥19(price_tier=new_user) ✅
- 后台活动:建活动¥15→下单实收¥15(即便tier=normal,后端命中活动) ✅
- **防绕过**:伪造 promo 无活动→原价¥29;伪造 new_user 非新用户→原价 ✅
- verify_all 139/139;迁移0009(promotions表+orders.price_tier列)

### 14.5 配置(config.py)
NEW_USER_HOURS=48 / NEW_USER_PACKAGE_ID="monthly_100" / NEW_USER_PRICE=19.00(可调,改 .env 重启生效)

---

## 十五、相关文档

- [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md) — 三模块契约层 SSOT(§6 支付)
- [32-ui-design-spec.md](./32-ui-design-spec.md) — §3.6 舞台模式
- [38-placeholder-integration-checklist.md](./38-placeholder-integration-checklist.md) — 占位接入
- [27-lessons-learned.md](./27-lessons-learned.md) — #036/#037 微信 V3 踩坑
- [01-progress.md](./01-progress.md) — 迭代进度
