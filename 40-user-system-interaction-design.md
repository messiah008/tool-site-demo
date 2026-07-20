# 用户系统交互设计

> **版本**: v1.0
> **最后更新**: 2026-07-20
> **状态**: 设计文档(待评审)
> **依赖**: [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md)(模块 A 用户系统契约)、[39-payment-redesign.md](./39-payment-redesign.md)、[32-ui-design-spec.md](./32-ui-design-spec.md)(§3.6 舞台模式)
>
> **本文是用户系统交互层 SSOT**。36 号是契约层(已定义资料/会话/找回/设备/注销 API),本文诊断前端交互缺口 + 设计完整用户交互链路。涵盖通用功能:登录/注册/退出/切换账号/找回密码/资料/设备/安全。

---

## 一、现状诊断

### 1.1 后端能力(已实现,36 号模块 A)

| 能力 | 文件:行 | 状态 |
|------|---------|------|
| 用户名注册(唯一/赠送/邀请/自动登录) | `auth.py:111-156` | ✅ 完整 |
| 用户名+密码登录+状态校验 | `auth.py:182-207` | ✅ 完整 |
| 邮箱/手机号登录 | `auth.py:81-83` | ❌ 仅用户名 |
| 微信扫码登录 | `auth.py:401-419` | ⚠️ 骨架不可用 |
| JWT(JTI/exp/HS256)+Redis黑名单+库兜底 | `security.py:27-76` | ✅ 完整 |
| 登出吊销 `/api/auth/logout` | `auth.py:247-268` | ✅ 后端完整 |
| 强制下线/全部下线 | `modules/user/api.py:57-98` | ✅ 完整 |
| 设备列表/设备名解析 | `modules/user/api.py:42-54`,`auth.py:60-70` | ✅ 完整 |
| IP 反查城市 | `models.py:25` location 字段 | ❌ 字段空 |
| GET/PUT 资料(昵称/头像/bio) | `modules/user/api.py:101-144` | ⚠️ phone 不可改 |
| 改密码+强制全部重登 | `modules/user/api.py:147-169` | ✅ 完整 |
| 改邮箱端点 | — | ❌ 缺(有验证码基建) |
| 绑定手机号 | — | ❌ 缺(无短信通道) |
| 找回密码(发码/校验/重置+防枚举/冷却/锁定) | `auth.py:284-398` | ✅ 完整 |
| SMTP 真发 | `email.py:43-91` | ⚠️ 配置空时降级 print |
| 注销账号(软删+立即吊销) | `modules/user/api.py:173-191` | ⚠️ 冷静期未落地 |
| 用户中心 dashboard/订单/卡密/任务/配额 | `api/user.py:16-127` | ✅ 完整 |
| 管理员判定(硬编码 ADMIN_USERNAMES) | `security.py:122-130` | ✅ 简化方案 |

### 1.2 前端交互缺口(关键)

| # | 缺口 | 影响 | 优先级 |
|---|------|------|--------|
| G1 | **退出登录链路断裂** | 导航栏/用户中心无退出按钮;`userStore.logout()`(user.ts:80-88)仅清本地,**未调后端 /auth/logout**,token 7 天有效期内可复用,设备列表会话不吊销 | 🔴 关键 |
| G2 | **切换账号缺失** | 无退出即无切换,多账号用户只能改密码/注销触发副作用 logout | 🟠 高 |
| G3 | **找回密码 UI 完全缺失** | 后端+client 封装就绪(client.ts:127-128)零 UI,无独立页/无 Login 入口/无验证码流程 | 🔴 关键 |
| G4 | 头像上传缺失 | API 支持 avatar_url 但无上传/预览组件 | 🟡 中 |
| G5 | 用户中心能力浪费 | orders/card-keys/convert-tasks/quota/dashboard 五个后端能力前端未接,信息密度不足 | 🟠 高 |
| G6 | `/user-center` 未加 requiresAuth | 未登录直接进→onMounted 401→拦截跳登录,体验差+丢 redirect | 🟡 中 |
| G7 | isAdmin 前端硬编码 | `['admin']`(user.ts:7)与后端耦合,加管理员改两处 | 🟡 中 |
| G8 | 注册无邀请码字段 | 福利 Tab 生成邀请链接但注册不采集,裂变闭环断裂 | 🟡 中 |
| G9 | 移动端适配薄弱 | 导航右区/用户中心 Tab 无响应式 | 🟢 低 |
| G10 | 401 拦截丢 redirect | client.ts:29 用 location.href='/login' 未带 redirect | 🟢 低 |
| G11 | 注销无强校验 | 仅点按确认,无密码/验证码二次确认 | 🟢 低 |
| G12 | 登录态过期无提示 | token 7天过期 401 被 client.ts:29 拦截跳 /login,无任何提示,用户莫名被踢 | 🟡 中 |

### 1.3 现有交互链路

```
未登录 → 导航栏[登录][注册] → /login(同页 Tab) → 登录/注册 → setSession → 回 redirect/首页
已登录 → 导航栏[用户名+余额][后台(admin)] → /user-center 4 Tab(资料/福利/设备/安全)
退出   → ⚠ 无入口(仅改密码/注销副作用 logout,且 logout 不调后端)
找回密码 → ⚠ 无 UI
```

---

## 二、设计目标

| 维度 | 目标 |
|------|------|
| **完整性** | 登录/注册/退出/切换账号/找回密码/资料/设备/安全 全链路可用 |
| **安全性** | 退出调后端吊销 token;改密码/找回/注销强制重登;注销强校验 |
| **易用性** | 退出入口显眼(导航栏);切换账号一键;找回密码自助;用户中心信息丰富 |
| **一致性** | 遵循 §3.6 舞台模式 + CSS 变量;错误反馈统一;加载态统一 |
| **合规** | 注销冷静期 + 撤销;设备管理可查;资料可改 |

---

## 三、整体交互架构

```
┌─ 未登录 ────────────────────────────────────────────┐
│ 导航栏: [登录] [注册] [定价]                         │
│   └ /login(登录/注册同页 Tab)                       │
│       ├ 登录 Tab:用户名+密码 → setSession → redirect │
│       ├ 注册 Tab:用户名+密码+邮箱+邀请码 → 自动登录  │
│       └ 忘记密码? → /forgot-password(新增)          │
└──────────────────────────────────────────────────────┘
          ↓ 登录成功
┌─ 已登录 ────────────────────────────────────────────┐
│ 导航栏: [后台(admin)] [用户名+余额▾] [定价]         │
│   └ 用户名下拉菜单(新增):                           │
│       ├ 用户中心 → /user-center                      │
│       ├ 切换账号 → 退出+回 /login?redirect=原页        │
│       └ 退出登录 → 调 /auth/logout → 清 store → /login │
│   └ /user-center 顶部固定区 + 4 Tab(合规 §2.4):     │
│       顶部固定区:余额+配额+统计(接 dashboard)        │
│       ├ 交易记录:订单+卡密(接 orders/card-keys)      │
│       ├ 资料:昵称+bio(+头像 P3.5)                    │
│       ├ 设备:设备列表/下线(已完整)                   │
│       └ 安全:改密码/注销(强校验)+退出登录            │
└──────────────────────────────────────────────────────┘
```

**核心新增**:
1. 导航栏用户名改**下拉菜单**(用户中心/切换账号/退出登录)
2. 找回密码独立页 `/forgot-password`(三步流程)
3. 用户中心**顶部固定区**(余额/配额/统计) + **4 Tab**(交易记录/资料/设备/安全,合规 §2.4 "Tab 2-3个")
4. 注册补邀请码字段;安全补退出登录按钮 + 注销强校验
5. 登录过期提示(401 拦截先提示再跳)

---

## 四、登录/注册交互

### 4.1 登录/注册页 `/login`(改造现有 Login.vue)

**现状**:同页 Tab 切换已可用,缺忘记密码入口+邀请码字段。

**改造**:
- 登录 Tab:用户名 + 密码 + [忘记密码?](链 /forgot-password) + 登录按钮
- 注册 Tab:用户名 + 密码 + 邮箱(选填) + **邀请码(选填,从 query.invite 预填)** + 注册按钮
- 注册调 `authApi.register({username,password,email,invite_code})`(client.ts:39-40 补 invite_code 透传)
- 登录/注册成功 → setSession → 回 redirect 或首页
- 加载态(loading)+错误反馈(ElMessage)已有,保留

**邀请码闭环**(修 G8):
- 福利 Tab 邀请链接 `/?invite=<username>` → 首页读 query.invite 存 sessionStorage → 跳 /login 注册 Tab 预填 invite_code
- 或 Login 注册 Tab 直接读 `route.query.invite` 预填

### 4.2 找回密码页 `/forgot-password`(新增,修 G3)

**后端就绪**:`forgot-password`(发码)/`verify-code`(校验)/`reset-password`(重置)全通,SMTP 真发。

**三步流程**(单页 step 切换,§3.6 舞台模式):
```
步骤1:输入邮箱 → POST /auth/forgot-password → "验证码已发送"(防枚举统一回成功)
步骤2:输入验证码 → POST /auth/verify-code(purpose=forgot_password)→ 校验通过
步骤3:输入新密码(≥8位)+ 确认 → POST /auth/reset-password → "重置成功,请重新登录"
   └ 重置后后端吊销所有会话(_revoke_all_sessions),跳 /login
```

**交互细节**:
- 顶部步骤指示器(①邮箱 → ②验证码 → ③新密码)
- 验证码 60s 重发冷却(后端已限),前端倒计时按钮
- 新密码二次确认 + 强度提示
- 成功后 ElMessage + 跳 /login(带 redirect)
- 防枚举:无此邮箱也回"已发送"(后端已实现,前端不暴露)
- **verify-code 安全**:对不存在邮箱,后端 `auth.py:360-361` 查无记录即 raise E302"验证码错误",与存在邮箱输错码同响应,**不泄露邮箱归属**(已实现,无需改)

### 4.3 登录方式扩展(后端缺口,本期可选)

- 邮箱登录:后端 login 改 `username OR email` 查询(小改)
- 微信扫码:后端骨架需补 state 生成+二维码 URL+回调(大改,P2)
- **本期建议**:补邮箱登录(低成本),微信扫码留后续

---

## 五、退出登录与切换账号(修 G1+G2,核心)

### 5.1 导航栏用户下拉菜单(改造 DefaultLayout.vue)

**现状**:用户名直接链 /user-center,无下拉。

**改造**:用户名+余额徽章 → 点击展开下拉菜单:
```
┌────────────────────┐
│ 👤 username  余额 12 │  ← 点击展开
├────────────────────┤
│ 📊 用户中心         │ → /user-center
│ 🔄 切换账号         │ → 退出 + /login?redirect=/
│ 🚪 退出登录         │ → 退出 + /login
└────────────────────┘
```

- 点击外部收起;CSS 变量样式
- **移动端**:P0 先上桌面下拉菜单;移动端折叠进汉堡菜单在 **P5** 收尾(P0 阶段窄屏用户区改为最简单按钮弹层兜底,避免溢出错位)

### 5.2 退出登录(修 G1 核心)

**现状 bug**:`userStore.logout()`(user.ts:80-88)仅清前端,**未调后端 /auth/logout**。

**修复**:
```ts
// stores/user.ts logout 改造
async logout() {
  try { await authApi.logout() } catch {}  // 调后端吊销 token(即便失败也清本地)
  this.token = null; this.userId = null; ...  // 清 state + localStorage
}
```
- 后端 `/auth/logout`(auth.py:247-268)吊销 JTI + 置 session.revoked_at
- 退出后 token 立即失效,设备列表该会话标已下线
- **退出不带 redirect**(退出就是退出,回原页无意义)→ 跳 `/login`

### 5.3 切换账号(修 G2)

**机制**:退出登录 + 回登录页(带 redirect 回原页)。本质是"重新登录"而非多账号并行——个人工具站无多账号并行场景,有意简化。
```ts
async switchAccount(currentPath: string) {
  await this.logout()  // 调后端吊销+清本地
  router.push({ path: '/login', query: { redirect: currentPath } })  // 切换才带 redirect
}
```
- 用户用新账号登录后回原页
- **退出 vs 切换 redirect 区分**:退出→`/login`(不带 redirect);切换→`/login?redirect=<当前页>`

---

## 六、用户中心改造(修 G5)

### 6.1 布局:顶部固定区 + 4 Tab(合规 §2.4 "Tab 2-3个",评审 H1)

32-ui-design-spec.md §2.4 明确"Tab 2-3个",故**不扩 6 Tab**。改为:**顶部固定区**(余额/配额/统计,非 Tab)+ **4 Tab**(交易记录/资料/设备/安全)。订单+卡密合并为「交易记录」(二级切换或并列)。

**顶部固定区**(始终可见,§3.6 舞台):
- 余额大数字 + 配额进度条(今日免费 X/3)
- 统计行:累计兑换/累计使用/订单数/卡密数(接 dashboard,**字段对齐后端**:balance.credits/total_redeemed/total_consumed + counts.orders/card_keys/convert_tasks)
- 余额不足 banner(已有)+ 快捷操作(兑换/购买/继续转换)

**4 Tab**:

| Tab | 内容 | 后端能力 | 状态 |
|-----|------|---------|------|
| 交易记录 | 订单列表 + 卡密兑换记录(二级 Tab 或并列) | orders + card-keys | 接 getOrders/getCardKeys |
| 资料 | 昵称+bio(+头像,见 §6.4) | profile | 补头像 |
| 设备 | 设备列表/下线 | sessions | ✅ 已完整 |
| 安全 | 改密码+注销+退出登录 | change-password/deactivate | 补退出按钮+强校验 |

### 6.2 顶部固定区(新增,接 dashboard)
- 余额舞台(§3.6 大数字)+ 配额进度条
- 统计:累计兑换/累计使用/订单数/卡密数(接 `userApi.getDashboard`,字段 balance.credits/total_redeemed/total_consumed + counts.orders/card_keys/convert_tasks)
- 快捷操作:兑换卡密/在线购买/继续转换

### 6.3 交易记录 Tab(新增,合并订单+卡密,接 G5)
- 订单列表:订单号/金额/状态徽章/渠道/时间(接 getOrders)
- 卡密兑换记录:脱敏卡密/状态/兑换时间(接 getCardKeys)
- 二级切换(订单|卡密)或上下两区并列;转换记录(getConvertTasks)可选并入

### 6.4 资料 Tab(补头像,修 G4)
- 头像:点击上传(MinIO 已配,需新增上传端点)+ 预览;**头像上传为独立任务(见 P3.5),本期可仅支持 URL 输入兜底**
- 昵称/bio:可编辑(已有)
- 用户名/邮箱:只读(已有)
- 改邮箱:补端点(后端验证码基建已就,需新增 /api/user/change-email)

### 6.5 安全 Tab(补退出+强校验)
- 改密码:已有(改完强制重登)
- 注销账号:补**强校验**(输入密码确认,修 G11)+ 30 天冷静期提示 + 撤销入口(见 §7.2)
- **退出登录按钮**(显眼位置,兜底入口,与导航栏下拉呼应)

---

## 七、安全与合规

### 7.1 会话安全
- 退出调后端吊销(修 G1):token 立即失效,不靠 7 天自然过期
- 改密码/找回/注销:后端 `_revoke_all_sessions` 强制全部重登(已实现)
- 设备管理:可查可下线(已完整)
- **登录过期提示**(修 G12):client.ts 401 拦截时先 `ElMessage.warning('登录已过期,请重新登录')` 再跳,并带 redirect(修 G10)

### 7.2 注销账号冷静期(后端补,评审 H2)

**现状**:软删 status=banned,无定时硬删+无撤销。`security.py:100`/`auth.py:188` 对 `status != "active"` 拒绝登录。
**冲突**:若注销设 deactivating 状态,会被登录拦截,**用户登不进来自撤销** → 30天可撤销形同虚设。

**设计(reactivate 走邮箱验证码,不依赖登录态)**:
- 注销:`status=deactivating`(新增,区分 banned)+ 立即吊销所有会话 + 设 `deactivated_at`
- **撤销不靠登录**:reactivate 复用找回密码的验证码流程——`POST /api/user/reactivate`(凭邮箱验证码,无需 token),校验通过后 `status=active` + 清 deactivated_at
- 30 天后定时任务硬删(lifespan 加任务,扫 deactivated_at<now-30d)
- 前端:注销页显示"X 天后永久删除,期间可凭邮箱验证码撤销";登录页对 deactivating 用户提示"账号已注销,凭邮箱验证码撤销或等待删除"

### 7.3 防刷
- 注册赠送 IP 限(已实现)
- 找回密码防枚举/冷却/锁定(已实现)
- 登录限流(nginx api_limit)

---

## 八、分阶段路线

### P0 退出登录 + 切换账号 + 登录过期提示(修 G1+G2+G12,最高优先)
| 文件 | 改什么 |
|------|--------|
| `stores/user.ts` | logout() 改 async + 调 authApi.logout() 吊销后端 token;加 switchAccount(path) |
| `layouts/DefaultLayout.vue` | 用户名改下拉菜单(用户中心/切换账号/退出登录);移动端最简弹层兜底 |
| `api/client.ts` | 401 拦截:先 ElMessage.warning('登录已过期')再跳,带 redirect(修 G10+G12) |
| `api/client.ts` | authApi.logout 确认封装(已有 client.ts:129) |

### P1 找回密码 UI(修 G3)
| 文件 | 改什么 |
|------|--------|
| `pages/auth/ForgotPassword.vue` | 新增三步流程页(邮箱→验证码→新密码),§3.6 舞台模式 |
| `router/index.ts` | 加 /forgot-password 路由 |
| `pages/Login.vue` | 加"忘记密码?"链接 |
| `api/client.ts` | 确认 forgotPassword/verifyCode/resetPassword 封装(已有) |

### P2 用户中心改造(修 G5,4 Tab + 顶部固定区,合规 §2.4)
| 文件 | 改什么 |
|------|--------|
| `pages/UserCenter.vue` | 顶部固定区(余额/配额/统计,接 dashboard)+ 4 Tab(交易记录/资料/设备/安全);交易记录合并订单+卡密 |
| `api/client.ts` | userApi.getOrders/getCardKeys/getDashboard/getConvertTasks 已封装,接通 |

### P3 邀请码闭环 + 改邮箱(修 G8,小改)
| 文件 | 改什么 |
|------|--------|
| `pages/Login.vue` 注册 Tab | 邀请码字段(query.invite 预填) |
| `api/client.ts` | register 透传 invite_code |
| 后端 | 改邮箱端点 /api/user/change-email(验证码基建已就,purpose=change_email) |
| `pages/UserCenter.vue` 资料 Tab | 改邮箱 UI(验证码流程) |

### P3.5 头像上传(修 G4,独立任务,可后置)
| 文件 | 改什么 |
|------|--------|
| 后端 | 用户头像上传端点(MinIO 用户 bucket + 签名 URL/直传) |
| `pages/UserCenter.vue` 资料 Tab | 上传/裁剪/预览组件 + avatar_url 回写 |
| 注 | 本期资料 Tab 可仅支持 URL 输入兜底,头像上传独立排期 |

### P4 安全与合规(修 G6+G7+G11)
| 文件 | 改什么 |
|------|--------|
| `router/index.ts` | /user-center 加 requiresAuth |
| 后端 | 注销冷静期:deactivating 状态 + reactivate 端点(邮箱验证码,不依赖登录,H2)+ 硬删定时任务 |
| `pages/UserCenter.vue` 安全 Tab | 注销强校验(输密码)+ 撤销入口提示 |
| 后端 + store | isAdmin 改后端返回 role(去硬编码,本期可不做,现有机制可用) |

### P5 移动端(修 G9)
| 文件 | 改什么 |
|------|--------|
| `layouts/DefaultLayout.vue` | 导航右区响应式+汉堡菜单 |
| `pages/UserCenter.vue` | Tab 横排改响应式 |

---

## 九、验证清单

**P0**:点退出登录→后端 session revoked_at 置位→token 立即失效→跳 /login(不带 redirect);切换账号→退出+回登录带 redirect;token 过期 401→先提示"登录已过期"再跳
**P1**:忘记密码→输邮箱→收验证码→重置→重新登录;无此邮箱也回"已发送";verify-code 对不存在邮箱报验证码错误(不泄露)
**P2**:用户中心顶部固定区(余额+配额+统计)+ 4 Tab;交易记录显示订单+卡密列表
**P3**:注册带邀请码→双方加点;改邮箱凭验证码成功
**P3.5**:头像上传成功+预览(独立任务)
**P4**:/user-center 未登录跳登录带 redirect;注销输密码确认;30 天内凭邮箱验证码撤销(无需登录)
**P5**:移动端导航+用户中心可用

---

## 十、相关文档

- [36-user-payment-cardkey-modules.md](./36-user-payment-cardkey-modules.md) — 模块 A 用户系统契约层(§5)
- [32-ui-design-spec.md](./32-ui-design-spec.md) — §3.6 舞台模式
- [39-payment-redesign.md](./39-payment-redesign.md) — 定价/支付
- [27-lessons-learned.md](./27-lessons-learned.md) — 错题集
