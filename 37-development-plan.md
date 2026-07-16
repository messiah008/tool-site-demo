# 三模块详细开发计划(36 号方案落地执行版)

> **版本**: v1.0
> **最后更新**: 2026-07-16
> **状态**: 待执行(P0 即将启动)
> **依据**: [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md) v1.2(SSOT 设计)
> **执行规范**: [07-consistency-link.md](./07-consistency-link.md)(SSOT 四链接) + [12-implementation-playbook.md](./12-implementation-playbook.md)(三律一循/8步循环) + [27-lessons-learned.md](./27-lessons-learned.md)(踩坑预防) + [32-ui-design-spec.md](./32-ui-design-spec.md)(前端规范)
> **作者**: AI 协作产出
>
> **本文件是「怎么做」的执行计划,36 号是「做什么」的设计 SSOT。两者冲突以 36 号为准,本文件负责把 36 号拆成可执行任务、对齐代码现状、排依赖、定验收。**

---

## 〇、怎么读本文件

| 读者 | 看哪章 |
|------|--------|
| 执行者(开干) | 第二章任务清单 + 第三章每阶段验收 + 第六章踩坑红线 |
| PM/排期 | 第四章依赖图 + 第五章里程碑 |
| 架构/AI 接手 | 第一章现状对齐 + 第六章踩坑 + 第七章文件清单 |

---

## 一、现状对齐(设计 vs 代码真实差异,必读)

> 36 号 §一盘点基于文档,本节是**代码实地核对**结果。三律一循「律一 Schema 优先」要求先认清这些差距,否则计划会落空。

### 1.1 后端代码现状(实地核对)

| 维度 | 36 号设计 | 代码现状 | 差距 → 计划动作 |
|------|----------|---------|----------------|
| 目录结构 | `app/modules/{user,payment,card_key}` 模块化 | 扁平 `app/api/{auth,payment,user,convert,query,proxy,logs}` | **modules 目录不存在**,P0 先建模块骨架(§2.1) |
| 数据库迁移 | 0002_user_sessions / 0003_payment_logs / 0004_card_redeem_logs / 0005_open_api | 已有 0001_initial + **0002_guest_token**(convert_tasks 加 guest_token 列) | **0002 槽位被占**,36 号命名需顺延:新迁移从 **0003** 起(§2.2 重编号) |
| 卡密兑换 | §7.2 含批次校验/valid_from/兑换日志 | `auth.py redeem`:行锁✅ 余额✅,但**无日志、无 valid_from、无批次校验** | P0 强化兑换逻辑 + 建日志表(§2.3) |
| 兑换日志表 | card_redeem_logs | **不存在** | P0 迁移建表 |
| OpenAPI 契约 | 36 号新增 24+5 接口 | `schemas/api/openapi.yaml` 仅 **20 路径/8 基础接口**,36 号新接口一个都没 | **严重滞后**,是 P0 第一道坎(§2.0,律一 Schema 优先) |
| 错误码 | E104-E507 | `error_codes.py` 有 E001-E500 体系,缺 E104-E507 增量 | P0 批量补(§2.4) |
| packages.py | 3 套餐 SSOT | ✅ 已有 | 复用,不改 |

### 1.2 前端代码现状

| 维度 | 现状 | 计划动作 |
|------|------|---------|
| Login/Redeem/UserCenter/Pricing | 已有但「工具式简陋」,违反 §3.6/§3.8 | P0 按 32 号规范改造 |
| auth/payment/card-key 组件目录 | 不存在 | P0/P1 新建 |
| 路由守卫 requiresAuth/requiresAdmin | **未实现**(router 现为静态) | P0 加守卫 |
| API Client | 现手写 `api/client.ts` | 律一:36 号新接口先入 openapi.yaml → gen:api 生成 |

### 1.3 关键决策(基于现状修订 36 号)

1. **迁移重编号**:0002 已被 guest_token 占用 → 36 号的 0002/0003/0004/0005 顺延为 **0003/0004/0005/0006**(详见 §2.2)。这是 36 号 v1.2 未发现的真实冲突,本计划修正。
2. **modules 目录渐进迁移**:36 号说「不破坏现有 app/api/auth.py,新功能放 modules」。P0 先建 `modules/card_key`(承载对外 API + 兑换增强),`auth.py` 兑换逻辑渐进迁入;`payment` 模块 P1 建。
3. **OpenAPI 优先**:律一 Schema 优先。P0 每个新接口必须**先改 openapi.yaml → gen:api → 再写后端路由**,不允许后端先写路由后补契约(CI 会拦)。

---

## 二、P0 任务清单(MVP,目标 9.5 人日,1-2 周)

> 36 号 §9.1 的 P0,按三律一循 8 步拆解。每个任务标注【SSOT】(改 schema 优先)/【代码】/【校验】/【进度】。

### 2.0【SSOT 先行】OpenAPI 契约补全(P0 卡点,最先做)

**为什么最先**:律一 Schema 优先。后端路由、前端 Client 都依赖契约,不做后面全卡。

- [ ] 把 36 号 §4.1 的 **模块 C 卡密管理接口**(admin generate/list/export/revoke/stats/bind-order + 兑换)+ §7.6.4 **对外 Open API 5 接口**写入 `schemas/api/openapi.yaml`
- [ ] 定义 `card_key_distributors` / `card_key_batches` 的 schema components
- [ ] `npm run gen:api` 生成前端 Client + 类型
- [ ] 后端导出 OpenAPI → `compare_openapi.py` 比对(CI 门禁)
- [ ] **红线**:openapi.yaml 的接口定义必须与 36 号 §4.1 + §7.6.4 一字不差(枚举值/错误码)

### 2.1 模块 C 后端骨架(modules/card_key)

- [ ] 建 `backend/app/modules/__init__.py` + `modules/card_key/{__init__,models,service,api,schemas,open_api,auth}.py`
- [ ] `models.py`:复用 `models/card_key.py` + 新增 `CardKeyDistributor` / `CardKeyBatch`(36 号 §7.6.2)
- [ ] `card_keys` 模型追加字段:`batch_id`(FK)、`valid_from`(激活 E105)
- [ ] **红线**:#003 新表 UUID 主键必 `server_default=sa.text("gen_random_uuid()")`

### 2.2 数据库迁移(顺延重编号)

> ⚠️ 36 号写 0002-0005,但 0002 已被 guest_token 占用,本计划顺延。**执行前先 `alembic current` 确认 head**。

- [ ] **0003_card_redeem_logs.py**:card_redeem_logs 表 + users 追加字段(phone/last_login)
  - 对应 36 号原 0004
- [ ] **0004_payment_logs.py**:payment_logs 表 + orders 追加字段(pay_url/expires_at/refunded_at/refund_amount)
  - 对应 36 号原 0003
- [ ] **0005_user_sessions.py**:user_sessions + user_profiles + email_verifications
  - 对应 36 号原 0002(P0 可仅建表,接口 P2 再上)
- [ ] **0006_open_api.py**:card_key_distributors + card_key_batches + card_keys 追加 batch_id/valid_from
- [ ] 每个迁移遵守 #003(UUID server_default);迁移后同步更新 ORM 模型(#007 雷区:迁移+模型+OpenAPI 三方同步)
- [ ] `alembic upgrade head` 无错 → 36 号 §11.1 门禁

### 2.3 卡密兑换强化(改 auth.py,强化为 §7.2)

- [ ] `redeem` 增加:第 3b 步批次状态校验(激活 E106)、第 4a 步 valid_from 校验(激活 E105)
- [ ] 落 `card_redeem_logs`(每个分支,含失败):成功 redeem / 失败 failed_attempt / 过期 expire
- [ ] 兑换逻辑渐进迁入 `modules/card_key/service.py`,`auth.py /redeem` 调用它(保留旧路径兼容)
- [ ] **红线**:#011 行锁顺序「先 card_keys 再 balances」统一(防死锁);`log_action` 失败也落

### 2.4 错误码补全

- [ ] `error_codes.py` 追加 E104-E106(卡密)+ E501-E507(Open API/退款),每个含 (description,message,status,retryable)
- [ ] 同步入 openapi.yaml 的 Error 响应

### 2.5 对外卡密 Open API(§7.6,P0 必上——你的核心需求)

> 用户需求:卡密在站点后台生成,外部卡密分发系统对接。这是 P0 不可延后的核心。

- [ ] `modules/card_key/auth.py`:API Key + HMAC-SHA256 + nonce 防重放中间件(§7.6.3)
  - api_secret **只存哈希**;`hmac.compare_digest` 常量时间比较;nonce 走 Redis
- [ ] `modules/card_key/open_api.py`:5 接口 claim/query/redeem-notify/revoke/batch_list(§7.6.4)
  - redeem-notify 只标卡密 used + 关联 external_order,**不动余额**;与站内 redeem 共用 FOR UPDATE 行锁
- [ ] 所有写接口校验 Idempotency-Key;E506 幂等命中返 200 首次结果
- [ ] 独立前缀 `/api/open/v1/`,区别 admin;`main.py` 注册路由
- [ ] **红线**:无白名单接口;所有操作落 card_redeem_logs(action 标 external_*)

### 2.6 卡密管理后台(站点生成卡密入口)

- [ ] 后端 `modules/card_key/api.py` admin 接口:generate/list/export/revoke/stats(§4.1)
- [ ] 管理员鉴权 `get_current_admin`(§7.3,ADMIN_USERNAMES 从 .env)
- [ ] 前端 `pages/admin/CardKeys.vue`(§3.6 舞台 + §3.8 审美):生成(选套餐+数量+批次) / 列表 / 导出 CSV / 作废
- [ ] 路由 `/admin/cards` + 守卫 requiresAdmin(§2.7)
- [ ] `scripts/generate_cards.py` 增强 `--db` 直写(§7.4)

### 2.7 前端改造(4 页 + 路由守卫)

- [ ] `Login.vue`:3 Tab(账号/微信占位/短信占位),§3.6 舞台模式(P0 先账号密码 Tab,扫码 P2)
- [ ] `UserCenter.vue`:5 Tab(概览/订单/卡密/任务/资料),§3.8 余额大数字为视觉重心
- [ ] `Redeem.vue`:舞台模式,顶部结果卡片 + 底部输入 + CardKeyInput 实时校验
- [ ] `Pricing.vue`:保留闲鱼入口 + 卡密兑换引导(§8.1,P1 加在线购买)
- [ ] `router/index.ts`:加守卫 requiresAuth/requiresAdmin;新路由见 §7
- [ ] **红线**:#016 Vue 事件 emit/监听名对应;#008 动态 class 用内联 style;#006 @import 在 @tailwind 前;localStorage 加 SSR 守卫(§10.4)

### 2.8 校验 + 部署(每阶段必做)

- [ ] 后端:`python scripts/check_consistency.py`(schema 一致)+ compare_openapi(契约一致)
- [ ] 前端:`npm run build`(SSG)+ `node scripts/validate.mjs`(SEO 护栏)+ `python scripts/verify_all.py`(应全绿)
- [ ] 改 backend 代码:`docker compose up -d --build api`(#007 必须 --build)
- [ ] 部署:`bash scripts/deploy_sync.sh` 同步前端 → Wsl nginx
- [ ] 更新 `01-progress.md` 记录 P0(律二 文档同步)

### 2.9 P0 验收(36 号 §9.1 + 本计划门禁)

- [ ] 注册→登录→卡密兑换→余额到账→PDF 转 Word 端到端跑通
- [ ] 管理员后台能批量生成卡密 + 导出 CSV
- [ ] 外部系统凭 API Key 能 claim 卡密 / redeem-notify 回传核销(§7.6 闭环)
- [ ] 闲鱼订单 channel=xianyu 录入 + 管理员手动确认 → 发卡(§6.3)
- [ ] 卡密兑换覆盖 E105(valid_from)/E106(批次停用)/兑换日志
- [ ] OpenAPI 契约一致(compare_openapi 通过)
- [ ] verify_all 全绿 + Docker 全服务 Up

---

## 三、P1-P4 任务概要(36 号 §9.2-9.5,本计划补执行要点)

### P1 在线支付(11 人日,2 周)——第二种收款方式,不下线闲鱼

- [ ] 渠道适配器 `payment/channels/`(base + wechat_native + xunhupay 迁入 + payjs 迁入),闲鱼 xianyu 已在 P0
- [ ] `pay_order_success()` 事务:paid→delivered,幂等 `status in ('paid','delivered')`
- [ ] **退款↔卡密联动**(§6.3 我加的闭环):unused→revoke;used→校验余额才扣,已消耗→E507
- [ ] Checkout.vue(选套餐→选渠道→扫码→轮询,超时调 cancel §6.4)/ Orders.vue / OrderDetail.vue
- [ ] 订单超时清理定时任务;对账脚本;Dashboard 收入按 channel 分类
- [ ] 微信 V3 **RSA-SHA256 非 MD5**(§10.2);回调先验签后处理;商户证书进 Docker secrets
- [ ] 上线前 0.01 元真实测试(沙箱≠真实)

### P2 用户系统(8 人日,1 周)

- [ ] 找回密码(邮箱验证码)/ 微信扫码 / 资料 / 设备列表 / 注销
- [ ] **本计划补**:JWT 黑名单 Redis + user_sessions 库双源兜底(评审断点1);注销时吊销所有 session
- [ ] **本计划补**:access token 15min-2h + refresh 7d(评审建议,P2 可选)

### P3 卡密运营(4.5 人日,1 周)

- [ ] 卡密统计页 / 闲鱼 CSV 批量发货 / 过期清理 / 作废退款联动 / 批次管理

### P4 对账风控(5 人日,1 周)

- [ ] 每日对账 + 退款流程 + 风控(同 IP 多次兑换失败告警)

---

## 四、任务依赖图(关键路径)

```
P0 关键路径(必须串行):
  2.0 OpenAPI 契约 ──┬─→ 2.1 模块C骨架 ──→ 2.3 兑换强化 ──→ 2.6 后台
                     └─→ 2.5 对外API(依赖2.1骨架+2.2迁移)
  2.2 迁移(0003→0006 串行) ──→ 2.3/2.5 依赖表
  2.4 错误码 ──→ 2.3/2.5 依赖
  2.7 前端改造 ──→ 依赖 2.0 的 gen:api Client(并行做,路由后接)
  2.8 校验部署 ──→ 所有任务完成后

可并行:
  2.1 骨架 ∥ 2.2 迁移 ∥ 2.4 错误码(2.0 完成后)
  2.7 前端(组件) ∥ 2.5/2.6 后端(gen:api 后)

P1 依赖 P0 的模块C骨架 + pay_order_success + 卡密后台
P2 依赖 P0 的 user_sessions 迁移(0005)
P3/P4 依赖 P1 的订单/卡密数据
```

**关键路径**:2.0 → 2.1+2.2 → 2.3 → 2.5 → 2.6 → 2.8。**2.0 OpenAPI 是全链卡点**,务必先通关。

---

## 五、里程碑(对齐 36 号 §9.7 总工时 37.5 人日)

| 里程碑 | 内容 | 人日 | 验收门禁 |
|--------|------|------|---------|
| M0 契约通 | 2.0 openapi.yaml 补全 + gen:api | 1.5 | compare_openapi 通过 |
| M1 卡密闭环 | 2.1-2.6 后端 + 后台 | 5 | 兑换/生成/对外API 端到端 |
| M2 MVP | 2.7 前端 + 2.8 部署验收 | 3 | P0 §2.9 全绿 |
| M3 在线支付 | P1 | 11 | 在线+闲鱼双渠道 |
| M4 用户系统 | P2 | 8 | 完整账户体系 |
| M5 卡密运营 | P3 | 4.5 | 统计/发货/清理 |
| M6 对账风控 | P4 | 5 | 对账/退款/风控 |

---

## 六、踩坑红线(执行必看,基于 27 号错题集 + 36 号 §10)

### 6.1 数据库(高频)

| 红线 | 来源 |
|------|------|
| 新表 UUID 主键必加 `server_default=sa.text("gen_random_uuid()")` | #003 |
| `bcrypt==4.0.1` 固定,不装最新 | #004 |
| 迁移改表后必同步 ORM 模型 + OpenAPI(三方同步) | #007 |
| 卡密兑换行锁顺序:先 card_keys 再 balances,统一顺序 | #011 |
| 迁移重编号:0002 已占,新迁移从 0003 起 | 本计划 §1.3 |

### 6.2 支付/卡密

| 红线 | 来源 |
|------|------|
| 支付回调先验签后处理,失败 raise E403 不落 payment_log | 36 §10.2 |
| 回调幂等:已 paid/delivered 直接 return success | 36 §6.3 |
| 退款必经卡密状态判断,不可只改订单(防双重得利) | 36 §6.3(本计划加) |
| 微信 V3 用 RSA-SHA256,非 MD5 | 36 §10.2 |
| 对外 API api_secret 只存哈希,验签用 hmac.compare_digest | 36 §7.6.3 |
| 对外 API nonce 走 Redis(非内存),多实例一致 | 36 §7.6.7 |

### 6.3 前端

| 红线 | 来源 |
|------|------|
| Vue emit 与 @监听名严格对应(update ≠ update:modelValue) | #016 |
| 动态 class 用内联 :style 或 :deep(),别靠 scoped | #008 |
| CSS @import 永远在 @tailwind 之前 | #006 |
| localStorage 访问加 typeof window 守卫(SSG) | 36 §10.4 |
| 改 API 名/重命名必 grep -rn 全项目搜调用点 | #014/#015 |
| 改工具交互模式必同步 registry input_schema | #011 |

### 6.4 部署

| 红线 | 来源 |
|------|------|
| 改 backend 代码必 `docker compose up -d --build api` | #007 |
| index.html 设 Cache-Control no-cache(hash 资源长缓存) | #013 |
| WSL 发行版名 Ubuntu-24.04(Ubuntu 不存在) | 双副本记忆 |
| 部署用 deploy_sync.sh,绝不同步 .env/certs/node_modules | 双副本记忆 |

---

## 七、关键文件清单(P0 新增/改造)

### 7.1 后端(新增)
- `schemas/api/openapi.yaml` — 补 24+5 接口(2.0,SSOT)
- `backend/app/modules/__init__.py` + `card_key/{models,service,api,schemas,open_api,auth}.py`
- `backend/alembic/versions/0003_card_redeem_logs.py` / `0004_payment_logs.py` / `0005_user_sessions.py` / `0006_open_api.py`
- `backend/app/error_codes.py` — 补 E104-E106 + E501-E507
- `backend/app/utils/security.py` — 追加 get_current_admin

### 7.2 后端(改造)
- `backend/app/api/auth.py` — redeem 强化 + 迁入 modules/card_key/service
- `scripts/generate_cards.py` — 加 --db

### 7.3 前端(新增)
- `frontend/src/pages/admin/CardKeys.vue`
- `frontend/src/components/card-key/CardKeyInput.vue`
- (P1)`pages/payment/{Checkout,Orders,OrderDetail}.vue` + `composables/useOrderPolling.ts`

### 7.4 前端(改造)
- `Login.vue` / `UserCenter.vue` / `Redeem.vue` / `Pricing.vue`
- `router/index.ts` — 守卫 + 新路由
- `api/client.ts` — 从 gen:api 生成(律一,不手写)

---

## 八、执行节奏(三律一循 8 步)

每个 P0 任务都走(12 号 §2.2):
```
1 识别需求 → 2 改 Schema(openapi/registry) → 3 生成代码(gen:api/build:tokens)
→ 4 实现业务 → 5 本地校验(verify_all/compare_openapi) → 6 更新 01-progress.md
→ 7 CI 校验 + 提交 → 8 部署(deploy_sync.sh)+ 监控
```

- **小步**:P0 内部按 2.0→2.1→2.2→...小步提交,每步可校验
- **门禁**:P0 §2.9 全过才进 P1(律三 小步快跑,不跳门禁)
- **文档同步**:每完成一个任务更新 01-progress.md(律二)

---

## 九、风险与外部依赖

| 风险/依赖 | 影响 | 应对 |
|----------|------|------|
| openapi.yaml 滞后严重 | P0 卡点 | 2.0 最先,1.5 人日集中通关 |
| 外部卡密系统对接规格未定 | 2.5 接口字段待确认 | 先按 36 §7.6 实现,预留字段,对接时微调 |
| 微信支付资质 | P1 | P1 才需,P0 闲鱼先行不阻塞 |
| 工作区未提交(死机风险) | 全部 | **执行前先 git commit 保命**(见下) |

### 🚨 执行第一步(必须先做)

**开干前先把工作区提交保命**——死机已两次,大量改动(含 36 号 v1.2 + 本计划)未进 git。
建议:`git add -A && git commit` 至少把 36/37 号文档 + 现有改动分组提交,再启动 P0。

---

## 十、与 36 号的差异说明

本计划相对 36 号 v1.2 的执行性修订(不改变设计,只对齐代码现状):
1. **迁移重编号 0002→0003 起**(0002 已被 guest_token 占用,36 号未察觉)
2. **P0 必含对外 Open API**(用户核心需求,不可延后)
3. **modules 渐进迁移**(现有扁平 api/ 不破坏,新功能入 modules)
4. **openapi.yaml 补全列为 P0 第一卡点**(律一 Schema 优先)
5. 补 JWT 双源兜底 / refresh token(P2 评审建议)

设计本身以 36 号 v1.2 为 SSOT,本计划只解决「怎么落到现有代码」。

---

## 十一、相关文档

- [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md) — 三模块设计 SSOT(v1.2)
- [07-consistency-link.md](./07-consistency-link.md) — SSOT 四链接
- [12-implementation-playbook.md](./12-implementation-playbook.md) — 三律一循/8步循环/6阶段门禁
- [27-lessons-learned.md](./27-lessons-learned.md) — 错题集 16 坑
- [32-ui-design-spec.md](./32-ui-design-spec.md) — 前端设计规范 v2.2
- [03-backend-architecture.md](./03-backend-architecture.md) — 后端架构/目录
- [04-api-contract.md](./04-api-contract.md) — OpenAPI 契约(待补全)
- [01-progress.md](./01-progress.md) — 迭代进度(每步更新)
