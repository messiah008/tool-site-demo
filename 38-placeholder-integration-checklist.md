# 占位代码接入清单(外部资源到位后逐一替换)

> **版本**: v1.0
> **最后更新**: 2026-07-16
> **状态**: 三模块方案 P0-P4 已全部落地,占位代码就位待真实接入
> **用途**: 外部资源(微信支付/SMTP/企业微信/虎皮椒/PayJS)到位后,按本清单逐一替换占位实现
> **配套**: [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md) v1.8 + [37-development-plan.md](./37-development-plan.md) v1.1
>
> **本文件是「接入替换地图」。每个占位含:文件:行 / 当前实现 / 替换目标 / 所需资源 / 接入步骤。**

---

## 〇、占位总览

| # | 占位 | 文件 | 所需外部资源 | 优先级 |
|---|------|------|------------|--------|
| 1 | 微信 Native 下单 + 回调验签 | modules/payment/channels/wechat_native.py | 微信支付 V3 商户资质 | P1 上线必需 |
| 2 | 虎皮椒下单 | channels/xunhupay.py | 虎皮椒账号+密钥 | P1 备选渠道 |
| 3 | PayJS 下单 | channels/payjs.py | PayJS 账号+密钥 | P1 备选渠道 |
| 4 | 邮件验证码发送 | api/auth.py `_send_email_code` | SMTP/阿里云邮件 | P2 找回密码必需 |
| 5 | 微信扫码登录 | api/auth.py wechat_qrcode/callback | 微信开放平台 AppID | P2 可选(有账号密码登录兜底) |
| 6 | 风控告警通道 | api/payment.py `_send_risk_alert` | 企业微信机器人 webhook | P4 上线推荐 |
| 7 | 支付回调路由接线 | api/payment.py callback_xunhupay/payjs | 渠道回调 URL 配置 | P1 上线必需 |
| 8 | 环境配置项 | app/config.py + .env | 各渠道密钥 | 所有接入前 |

> **核心原则**:占位代码已实现完整结构(接口签名/数据模型/状态机/幂等),接入只需替换「真实 API 调用」部分,不动业务逻辑。每个占位处有 `TODO`/`占位` 注释,grep 即可定位。

---

## 一、微信 Native 扫码支付(✅ 已接入 2026-07-17,V3 真实下单)

> **已接入**:商户私钥入 `backend/secrets/`(compose 挂载 /app/secrets),`.env` 配 V3 全字段,`wechat_native.py` 重写为真实 V3 下单(RSA-SHA256)+回调验签解密,`payment.py` 加 `callback/wechat`,后台测试连通=真连下单。真连验证拿到真实 `code_url`。踩坑见 27 号 #036(签名顺序)/#037(微信支付公钥模式)。
> **待办**:① 下载微信支付公钥(.pem)+公钥ID 放 secrets 配 WECHAT_PUB_KEY_PATH 启严格回调验签;② 公网 DNS+SSL 就绪微信回调方能触达。下文为原占位说明(留档对照)。

### 1.1 下单 — `backend/app/modules/payment/channels/wechat_native.py` · ✅ 已接入

**原占位**:返回占位二维码 `weixin://wxpay/bizpayurl?pr=test_{order_no}`(测试用)

**替换目标**:调微信 V3 下单 API,拿真实 `code_url`

**所需资源**(写入 .env,见 §八):
```
WECHAT_APPID=         # 服务号/小程序 AppID
WECHAT_MCHID=         # 商户号
WECHAT_API_V3_KEY=    # API V3 密钥(32字节)
WECHAT_SERIAL_NO=     # 商户证书序列号
WECHAT_PRIVATE_KEY_PATH=/run/secrets/wechat_private.pem  # 商户私钥 PEM(Docker secrets)
WECHAT_NOTIFY_URL=https://zhiniao.tools/api/payment/callback/wechat
```

**接入步骤**:
1. `create_payment` 实现:构造 params(appid/mchid/out_trade_no=order.order_no/notify_url/amount{total:int(order.amount*100)}) → `_sign_v3(params)`(RSA-SHA256,商户私钥) → httpx.post V3 native → 返回 `{qr_data: resp["code_url"], expires_in:1800}`
2. 实现 `_sign_v3`:用商户私钥对 `timestamp\nnonce\n...` 签名(V3 签名规范)
3. **踩坑红线**(36 §10.2):V3 用 RSA-SHA256 **不是 MD5**;商户私钥 PEM 放 Docker secrets 不入 git

### 1.2 回调验签 — `wechat_native.py:40-52` (`verify_callback`)

**当前**:`raise_error("E404", "微信回调待接入真实 V3 验签")`

**替换目标**:验签 + 解密回调,返回 `{order_no, status:"paid", amount, raw_payload}`

**接入步骤**:
1. 验 `Wechatpay-Signature` header(微信平台证书)
2. AES-256-GCM 解密 resource(用 WECHAT_API_V3_KEY)
3. 取 out_trade_no + 金额
4. **红线**:先验签再处理,失败 raise E403 **不落 payment_log**(§10.2);5 秒内响应

### 1.3 回调路由 — `backend/app/api/payment.py` callback(§七)

当前 payment.py 有 `callback/xunhupay`、`callback/payjs`,**缺 `callback/wechat`**。接入时加:
```python
@router.post("/callback/wechat")
async def callback_wechat(request: Request, db: Session = Depends(get_db)):
    channel = get_channel("wechat_native")
    data = channel.verify_callback(request)  # 验签解密
    order = get_order_by_no(db, data["order_no"])
    if order and data["status"] == "paid":
        pay_order_success(db, order)  # 同一发卡逻辑
    return {"code": "SUCCESS"}  # 微信要求
```

---

## 二、虎皮椒渠道(P1 备选)

### 2.1 下单 — `channels/xunhupay.py:18-27`

**当前**:返回占位 `payjs跳转URL`(未调下单 API)

**接入**:调虎皮椒下单 `POST https://api.xunhupay.com/payment/do.html`,拿 `pay_url`(跳转支付页)

**所需资源**:`XUNHUPAY_APPID` + `XUNHUPAY_SECRET`(写 .env)

**验签已实现且加固(2026-07-20,39 号 §八)**:`verify_callback_data`(MD5)从旧 `_verify_xunhupay_sign` 迁入,app_secret 读 `settings.XUNHUPAY_SECRET`(**fail-fast:空→E403,删 TODO_SECRET 占位**)+ 补金额校验(回调 total_amount vs 本地 order.amount)。回调路由 `callback_xunhupay` 已改走 `XunhupayChannel().verify_callback_data`。

### 2.2 旧函数清理 — ✅ 已完成(2026-07-20)

旧 `_verify_xunhupay_sign`/`_verify_payjs_sign`(曾用 `TODO_SECRET` 占位)已从 payment.py 删除,回调路由改用 `channel.verify_callback_data`(39 号 §八,消除重复验签)。`_generate_pay_url` 保留(旧 `/order` 端点用)。

---

## 三、PayJS 渠道(P1 备选)

### 3.1 下单 — `channels/payjs.py:18-27`

**当前**:占位跳转 URL

**接入**:调 PayJS 下单 `POST https://payjs.cn/api/native`

**所需资源**:`PAYJS_MCHID` + `PAYJS_SECRET`

**验签已实现且加固(2026-07-20,39 号 §八)**:`verify_callback_data`(MD5),app_secret 读 `settings.PAYJS_SECRET`(**fail-fast:空→E403,删 TODO_SECRET 占位**)+ 补金额校验(回调 total_fee 分→元 vs 本地 order.amount)。回调路由 `callback_payjs` 已改走 `PayjsChannel().verify_callback_data`。

---

## 四、邮件验证码(P2 找回密码必需)

### 4.1 发送 — `backend/app/api/auth.py` (`_send_email_code`) · ✅ 已接入(2026-07-16)

**已接入**:新建 `backend/app/utils/email.py`(`send_verify_code`→`send_email`,SSL,`SMTP_FROM` 空→回退 `SMTP_USER`,适配阿里云 DM 发信地址即账号),`_send_email_code` 调用之,配置不全/发送失败降级控制台打印不阻断(找回密码防枚举仍统一回"已发送")。`parse_host_port` 容错用户在「服务器」填 `host:port` 合体。配置写 .env 由后台「支付接入」页配置(见 §七)。下文为原占位说明(留档对照)。

**原占位**:`print(f"[EMAIL][{purpose}] to={email} code={code}")`(控制台输出,开发可见)

**替换目标**:真实发邮件(SMTP 或阿里云邮件 API)

**所需资源**(写 .env):
```
SMTP_HOST=smtp.qq.com
SMTP_PORT=465
SMTP_USER=        # 发件邮箱
SMTP_PASSWORD=    # 授权码(非邮箱密码)
SMTP_FROM=知鸟工具箱 <noreply@zhiniao.tools>
# 或阿里云邮件:ALIYUN_DM_*
```

**接入步骤**:
1. 实现 `_send_email_code`:用 `smtplib`(SSL)或 httpx 调阿里云邮件 API 发送
2. 邮件内容:6 位验证码 + 有效期 5 分钟提示
3. 业务逻辑(forgot-password/verify-code/reset-password)**已完整**,只换发送通道

> 找回密码全流程(发码/校验/重置/防枚举/锁定)已实现并验证,接入只需替换 `_send_email_code` 一处。
>
> **验证发信能力(已落地,2026-07-16)**:后台「支付接入」页 SMTP 渠道新增「发测试邮件」——管理员填收件邮箱,后端 `POST /api/admin/config/test-email`(`config/service.py:send_test_email`)真发一封测试邮件,验证登录+发信+收件全链路(区别于「测试连通」仅登录)。用运行时 settings,刚改配置需先重启生效。openapi 已补 config 模块 5 端点。

---

## 五、微信扫码登录(P2 可选)

### 5.1 二维码 — `api/auth.py:355-366` (`wechat_qrcode`)

**当前**:无 WECHAT_APPID 时 raise E404,有则返回 `TODO`

**所需资源**:`WECHAT_APPID` + `WECHAT_APPSECRET` + `WECHAT_REDIRECT_URI`

**接入步骤**:
1. 生成 state(随机 32 字节,存 Redis `wechat:state:{state}`,TTL=300s)
2. 返回 `https://open.weixin.qq.com/connect/qrconnect?appid=...&state=...`
3. 前端 Login.vue 加微信 Tab 展示二维码(当前只有账号密码 Tab)

### 5.2 回调 — `api/auth.py:369-372` (`wechat_callback`)

**当前**:`raise_error("E404", "微信扫码回调未接入")`

**接入步骤**:
1. 校验 state 在 Redis
2. 调微信 API 换 access_token + openid
3. 查 users WHERE wechat_openid=openid:存在→签 token;不存在→建用户(wx_+openid前8位)→签 token
4. 302 → /login/wechat?token=xxx(§5.2 流程)

> 有账号密码登录兜底,微信扫码可后置。用户系统其他功能(资料/设备/改密/注销)已完整。

---

## 六、风控告警通道(P4 上线推荐)

### 6.1 告警发送 — `backend/app/api/payment.py:505-510` (`_send_risk_alert`)

**当前**:`print(f"[RISK-ALERT] ...")`(控制台日志)

**替换目标**:发企业微信机器人 webhook

**所需资源**:`WECOM_WEBHOOK`(企业微信群机器人 webhook URL,写 .env)

**接入步骤**:
1. 实现 `_send_risk_alert`:httpx.post WECOM_WEBHOOK,发 markdown 告警(IP/失败次数/原因/级别)
2. 风控逻辑(risk-alerts 端点,IP 聚合/超阈值/分级)**已完整**,只换发送通道

> 风控查询/聚合已实现验证,接入只需替换 `_send_risk_alert` 一处。

---

## 七、环境配置项接入(所有接入前)

### 7.1 缺失的配置项 — `backend/app/config.py`

当前 config.py **只有**基础配置 + ADMIN_USERNAMES,缺所有渠道/邮件/告警配置。接入时在 Settings 类加:

```python
# 微信支付 V3
WECHAT_APPID: str = ""
WECHAT_MCHID: str = ""
WECHAT_API_V3_KEY: str = ""
WECHAT_SERIAL_NO: str = ""
WECHAT_PRIVATE_KEY_PATH: str = ""
WECHAT_NOTIFY_URL: str = ""
# 微信登录(开放平台,与支付 AppID 不同)
WECHAT_LOGIN_APPID: str = ""
WECHAT_LOGIN_APPSECRET: str = ""
WECHAT_LOGIN_REDIRECT_URI: str = ""
# 虎皮椒/PayJS
XUNHUPAY_APPID: str = ""
XUNHUPAY_SECRET: str = ""
PAYJS_MCHID: str = ""
PAYJS_SECRET: str = ""
# 邮件
SMTP_HOST: str = ""
SMTP_PORT: int = 465
SMTP_USER: str = ""
SMTP_PASSWORD: str = ""
SMTP_FROM: str = ""
# 风控告警
WECOM_WEBHOOK: str = ""
```

> 加完后 .env 填真值(.env 在 .gitignore,不入库)。渠道代码用 `getattr(settings, "XUNHUPAY_SECRET", "TODO_...")` 已兼容空值。

### 7.2 .env 位置(WSL 部署)

- WSL 侧 `/root/tool-site-project/.env`(根目录)
- `backend/.env` 软链到根 .env(已建,compose env_file 能读)
- 生产用 `.env.prod`(从 .env.prod.example 复制),含强密码/JWT/CORS

---

## 八、外部资源准备清单(我无法代劳)

| 资源 | 用于 | 备注 |
|------|------|------|
| 微信支付商户号 + V3 证书 | §一 | 国内收款必需,需企业资质 |
| 虎皮椒/PayJS 账号 | §二/三 | 个人可用支付聚合,无需企业资质 |
| SMTP 邮箱(QQ/阿里云) | §四 | 找回密码,用授权码 |
| 微信开放平台 AppID | §五 | 扫码登录(可选) |
| 企业微信群机器人 | §六 | 风控告警(可选,免费) |
| 域名 + SSL + 备案 | 全站 | nginx server_name + site.json domain |
| Linux VPS | 部署 | 2C4G |
| WPS 商业版 License | PDF 转 Word | 付费工具后端 |

---

## 九、接入顺序建议

1. **域名+SSL+备案+VPS**(上线前提)
2. **SMTP 邮件**(§四,找回密码,低成本)
3. **虎皮椒/PayJS**(§二/三,个人可用,无需企业资质,P1 快速上线)
4. **企业微信告警**(§六,免费,运维)
5. **微信支付 V3**(§一,需企业资质,正式在线收款)
6. **微信扫码登录**(§五,可选,有账号密码兜底)

> 每个 §独立接入,互不阻塞。接入后跑 verify_all + 对应端到端验证(36 号 §11 检查清单)。

---

## 十、相关文档

- [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md) — 三模块设计 SSOT v1.8(§5.2 微信/§6.1 渠道/§10 踩坑)
- [37-development-plan.md](./37-development-plan.md) — 开发计划 v1.1(已全部落地)
- [27-lessons-learned.md](./27-lessons-learned.md) — 错题集(§10.2 支付踩坑/V3 RSA-SHA256)
- [01-progress.md](./01-progress.md) — 进度(P0-P4 完成 + 待外部资源清单)
