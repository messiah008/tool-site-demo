# 错题集 (Lessons Learned)

> **版本**: v1.0
> **最后更新**: 2026-07-16
> **状态**: 持续更新,每次踩坑必记录
> **目的**: 避免重复犯错,新 AI 接手时先读此文件

---

## 一、使用方法

| 读者 | 用法 |
|------|------|
| 新 AI 接手 | 读此文件了解已踩过的坑,避免重蹈覆辙 |
| 遇到新问题 | 解决后在第四章"问题归档"追加记录 |
| 修改代码前 | 检查是否触碰已记录的雷区 |

**记录原则**: 每个问题必含 ① 现象 ② 根因 ③ 修复 ④ 预防 ⑤ 文件位置

---

## 二、关键雷区速查(高频/高危)

### 雷区 1: vite manualChunks 强制分块破坏循环依赖

**禁止**: 对 Element Plus / Lodash 等内部有循环依赖的库,不要 `return 'vendor-xxx'`

**正确**: 只精确分无循环依赖的小库,大型库让 Vite 自动处理

详见 [问题 #001]

### 雷区 2: Windows 路径在 Docker 与本地不一致

**禁止**: 用 `Path(__file__).parent.parent.parent` 假设目录结构

**正确**: Docker/本地双路径检测

详见 [问题 #002]

### 雷区 3: 数据库 UUID 主键无 server_default

**禁止**: 只在 SQLAlchemy 模型加 `default=uuid.uuid4`(应用层),SQL INSERT 无 id 时失败

**正确**: 迁移加 `server_default=sa.text("gen_random_uuid()")`

详见 [问题 #003]

### 雷区 4: bcrypt 4.1+ 与 passlib 1.7.4 不兼容

**禁止**: `pip install bcrypt`(装最新版)

**正确**: 固定 `bcrypt==4.0.1`

详见 [问题 #004]

### 雷区 5: Docker 挂载嵌套只读目录

**禁止**: `./dist:/html:ro` + `./public:/html/public:ro`(在只读目录下创建子挂载)

**正确**: 把 public 文件复制到 dist 根,移除嵌套挂载

详见 [问题 #005]

### 雷区 6: CSS @import 必须在 @tailwind 之前

**禁止**:
```css
@tailwind base;
@import './tokens.css';  /* ← 失效! */
```

**正确**:
```css
@import './tokens.css';  /* ← 文件顶部 */
@tailwind base;
```

详见 [问题 #006]

### 雷区 7: Docker compose 改代码后需重建镜像

**禁止**: 改了 backend 代码后只 `docker compose restart api`

**正确**: `docker compose up -d --build api`(代码在构建时 baked 进镜像)

详见 [问题 #007]

### 雷区 8: Vue scoped 对动态拼接 class 名样式失效

**禁止**: `:class="['tool-icon', \`icon-${color}\`]"` + scoped CSS

**正确**: 用内联 `:style="{ background: colorMap[color] }"`

详见 [问题 #008]

---

## 三、预防检查清单

每次开发前确认:

- [ ] 新增 npm 依赖 → 检查是否内部循环依赖,是则不进 manualChunks
- [ ] 新增 Docker 挂载 → 检查是否嵌套在只读目录下
- [ ] 新增数据库表 → UUID 主键加 server_default
- [ ] 改 CSS @import → 确保在 @tailwind 之前
- [ ] 改 backend 代码 → 用 `--build` 重建镜像
- [ ] Vue 动态 class → 用内联 style 或 `:deep()`
- [ ] 改完跑 `python scripts/verify_all.py` 全量验证

---

## 四、问题归档

### 问题 #001: Cannot access 'z' before initialization

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-03 |
| **现象** | 首页白屏,Console 报 `vendor-element-xxx.js:1:3836 Cannot access 'z' before initialization` |
| **根因** | vite manualChunks 用 `id.includes('vue')` 误匹配 `@vue/runtime-dom`,且强制把 Element Plus 分到同一 chunk,破坏其内部循环依赖初始化顺序 |
| **修复** | `frontend/vite.config.ts`:精确匹配 `/vue/`,Element Plus 不手动分块 |
| **预防** | 大型库(Element Plus/Lodash)不强制 manualChunks,只分小库(crypto-js/qrcode/zxcvbn) |
| **文件** | `frontend/vite.config.ts` |

### 问题 #002: SCHEMAS_DIR 在 Docker 找不到

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-02 |
| **现象** | API 启动报 `FileNotFoundError: '/schemas/tools.registry.json'` |
| **根因** | `config.py` 用 `Path(__file__).parent.parent.parent / "schemas"`,Docker 中 `/app/app/config.py` parent×3 = `/`,但实际挂载在 `/app/schemas` |
| **修复** | `backend/app/config.py`:Docker 优先 `/app/schemas`,本地回退项目根 |
| **预防** | 路径计算用双检测,不假设目录结构 |
| **文件** | `backend/app/config.py` |

### 问题 #003: UUID 主键导入卡密失败

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-02 |
| **现象** | `cat cards/*.sql \| psql` 报 `null value in column "id" violates not-null constraint` |
| **根因** | SQLAlchemy 模型有 `default=uuid.uuid4`(应用层),但迁移 SQL 无 `server_default`,直接 SQL INSERT 不带 id 时数据库无法自动生成 |
| **修复** | `backend/alembic/versions/0001_initial.py`:所有 UUID 主键加 `server_default=sa.text("gen_random_uuid()")` |
| **预防** | 数据库主键必须有 server_default,不能只靠应用层 |
| **文件** | `backend/alembic/versions/0001_initial.py` |

### 问题 #004: bcrypt 版本不兼容

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-02 |
| **现象** | 注册接口 500,日志 `ValueError: password cannot be longer than 72 bytes` |
| **根因** | passlib 1.7.4 与 bcrypt 4.1+ API 不兼容 |
| **修复** | `backend/requirements.txt`:固定 `bcrypt==4.0.1` |
| **预防** | 密码库固定版本,不装最新 |
| **文件** | `backend/requirements.txt` |

### 问题 #005: Nginx public 目录挂载冲突

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-02 |
| **现象** | Nginx 启动报 `unable to start container: read-only file system` |
| **根因** | docker-compose 同时挂载 `./dist:/html:ro` + `./public:/html/public:ro`,在只读目录下创建子挂载失败 |
| **修复** | 移除 `./public` 挂载,把 robots.txt/llms.txt 复制到 dist |
| **预防** | 不在只读挂载目录下创建子挂载 |
| **文件** | `docker-compose.yml` |

### 问题 #006: CSS 变量未生效(卡片无边框)

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-03 |
| **现象** | 工具卡片外围轮廓透明,无 border 线 |
| **根因** | `main.css` 的 `@import './tokens.css'` 写在 `@tailwind` 之后,违反 CSS 规范(`@import` 必须在文件顶部),导致 `:root` 变量未进入构建 |
| **修复** | `frontend/src/assets/styles/main.css`:`@import` 移到 `@tailwind` 之前 |
| **预防** | CSS `@import` 永远在文件最前 |
| **文件** | `frontend/src/assets/styles/main.css` |

### 问题 #007: Docker 改代码不生效

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-03 |
| **现象** | 加了 logs.py 路由,`docker compose restart api` 后路由不存在 |
| **根因** | Docker 镜像构建时 COPY 代码进去,restart 只重启容器不重建镜像 |
| **修复** | `docker compose up -d --build api`(加 `--build` 重建镜像) |
| **预防** | 改 backend 代码必须 `--build`;改前端代码 `npm run build` 后 `restart nginx` |
| **文件** | (运维操作) |

### 问题 #008: Vue scoped 动态 class 样式失效

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-03 |
| **现象** | 图标背景色不显示,只有光秃秃 emoji |
| **根因** | `:class="['tool-icon', \`icon-${color}\`]"` + `<style scoped>`,scoped 给动态类加 `[data-v-xxx]` 属性,但构建后选择器命中不稳定 |
| **修复** | 用内联 `:style="{ background: colorMap[color] }"` 绕过 scoped |
| **预防** | 动态 class 名避免依赖 scoped,用内联 style 或 `:deep()` |
| **文件** | `frontend/src/pages/Home.vue` |

### 问题 #009: 校验脚本路径假设

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-03 |
| **现象** | `python scripts/check_consistency.py` 在 frontend 目录报 `No such file` |
| **根因** | 脚本用相对路径 `schemas/`,必须在项目根执行 |
| **修复** | 脚本内自动定位项目根,或文档强调"在项目根执行" |
| **预防** | 脚本用 `Path(__file__).parent.parent` 定位根,或文档明确 cd 要求 |
| **文件** | `scripts/*.py` |

### 问题 #010: lunar-javascript API 名不匹配

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-08 |
| **现象** | 万年历查询报 `getShengXiao is not a function` |
| **根因** | lunar-javascript 的生肖方法叫 `getAnimal()`,不是 `getShengXiao()` |
| **修复** | 改用 `lunar.getAnimal()` 获取生肖 |
| **预防** | 使用 lunar-javascript 前先 `node -e` 测试所有方法名,不凭名字猜测 |
| **文件** | `frontend/src/tools/lunar-calendar/index.vue` + `frontend/src/tools/zodiac/index.vue` |

> **复发记录**(2026-07-08): #010 修复时只改了 lunar-calendar,漏改了 zodiac/index.vue(同一 API 调用)。通过日志系统 `diagnose.py --latest` 发现。**教训:修复 API 名变更时,必须 `grep -rn` 全项目搜索所有调用点,不能只改当前文件。**

### 问题 #011: registry input_schema 需随交互模式调整

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-08 |
| **现象** | 万年历从"输入框查单日"改为"月历网格"后,旧的 input_schema 仍残留,导致页面显示无用输入框 |
| **根因** | 改交互模式时只改了 Vue 组件,忘了同步更新 registry 的 input_schema |
| **修复** | `input_schema: []`(空数组,表示无表单输入,纯交互) |
| **预防** | 改工具交互模式时,registry 的 input_schema + action_label 必须同步更新 |
| **文件** | `schemas/tools.registry.json` |

### 问题 #012: API 端口不映射导致本地访问不了

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-08 |
| **现象** | `http://localhost:8000/` 访问不了(connection refused) |
| **根因** | 部署时把 API 的 `ports: 8000:8000` 改成 `expose: 8000`,expose 只对 Docker 内部开放,不映射到宿主机 |
| **修复** | 加回 `ports: "${EXPOSE_API_PORT:-8000}:8000"`,生产用环境变量关闭 |
| **预防** | 开发环境 ports 映射必须有;生产才用 expose 或环境变量关闭。改 compose 端口时想清楚影响 |
| **文件** | `docker-compose.yml` |

### 问题 #013: 前端更新后白屏(浏览器缓存旧 index.html)

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-09 |
| **现象** | 部署新版前端后,用户浏览器白屏无法访问 |
| **根因** | Vite 构建后 JS/CSS 文件名带 hash(如 `index-ABC123.js`),更新后 hash 变了。但 `index.html` 没有 `Cache-Control` 头,浏览器缓存了旧的 `index.html`,其中引用的旧 hash JS 文件已不存在(404),导致白屏 |
| **修复** | nginx 给 `index.html` 加 `Cache-Control: no-cache, no-store, must-revalidate`,确保每次拿到最新版。JS/CSS 保持 `immutable` 长缓存(hash 变了自动失效) |
| **预防** | SPA 部署必须给 `index.html` 设 no-cache;静态资源(hash 文件名)才长缓存。不要让用户手动清缓存,从服务端解决 |
| **文件** | `nginx/nginx.conf` |

### 问题 #014: 修复 API 变更只改当前文件,漏改其他调用点

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-09 |
| **现象** | 修复 #010(`getShengXiao`→`getAnimal`)只改了 lunar-calendar,漏改 zodiac,用户访问生肖查询崩溃 |
| **根因** | 同一 API 调用在多个文件中,修复时只改了报错的文件,没有全局搜索 |
| **修复** | `zodiac/index.vue` 也改为 `getAnimal()` |
| **预防** | **修复 API 名变更/重命名时,必须 `grep -rn` 全项目搜索所有调用点,不能只改当前文件** |
| **文件** | `frontend/src/tools/zodiac/index.vue` |

---

### 问题 #015: lunar-javascript getAnimal() 返回二十八宿动物,非生肖

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-09 |
| **现象** | 开发 solar-to-lunar 时,lunar.getAnimal() 返回"狼",而非预期的生肖"马"(2026 丙午年) |
| **根因** | getAnimal() 返回的是当天的二十八宿对应动物(如"奎木狼"→"狼"),与生肖无关。生肖应使用 getYearShengXiao()。错题集 #010 当年把生肖获取从 getShengXiao 改为 getAnimal,方向有误 |
| **修复** | 全项目 3 处 `getAnimal()` 调用点(经 `grep -rn` 确认无遗漏):solar-to-lunar/zodiac/lunar-calendar 均改为 `getYearShengXiao()`。node 实测 2020-2031 共 12 年返回值均匹配 ZODIAC_DATA 生肖 key(鼠牛虎…猪)。`getAnimal()` 已从代码库彻底移除(grep 复核 0 匹配) |
| **预防** | 取生肖一律用 getYearShengXiao();使用 lunar-javascript 任何方法前先 `node -e` 实测返回值,不凭方法名猜测含义 |
| **文件** | `frontend/src/tools/solar-to-lunar/index.vue`、`zodiac/index.vue`、`lunar-calendar/index.vue` |

> **复发记录(2026-07-09 第2次)**: zodiac/index.vue 在后续"补深度内容"重构时又被改回 `getAnimal()`(可能用了旧模板),导致任意年份都 fallback 显示"鼠"(data.ts key 是生肖,getAnimal 返回二十八宿动物)。**教训:重构工具页时若涉及生肖取值,必须重新用 getYearShengXiao(),不能从旧代码片段复制。建议在 data.ts 顶部加注释标明 key 是 getYearShengXiao() 返回值。**

---

### 问题 #016: ToolInput emit 与父组件监听事件名不匹配,导致改参数后查询无反应(全站 bug)

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-10 |
| **现象** | 用户报"黄道吉日查询一次后按钮无效";精确复现是"改参数后再点查询无反应"。首次查询成功(用初始默认值),改 select/输入后点按钮结果不变 |
| **根因** | `ToolInput.vue` 只 `emit('update:modelValue')`,但 72 个工具页用 `@update="handleUpdate"` 监听(非 `@update:modelValue`)。Vue3 中 `update` 与 `update:modelValue` 是**不同事件**——`@update` 监听不到 `update:modelValue` emit → handleUpdate 不触发 → form 不更新 → 改的参数不生效,query 仍用旧默认值 |
| **验证** | Chrome headless 最小复现:父用 `@update` 监听,子只 emit `update:modelValue` → `form_x` 始终为初始值;修复后同时 emit 两事件 → `form_x` 正确更新 |
| **修复** | `ToolInput.vue` update/toggleCheckboxGroup 同时 emit `update` 和 `update:modelValue`(一处修复,72 工具全生效,无需改父组件) |
| **预防** | Vue3 自定义事件 emit 与监听名必须严格对应;`v-model` 默认 `@update:modelValue`,手写 `@update` 是不同事件。组件设计时 emit 应覆盖所有可能的监听名,或文档统一约定 |
| **文件** | `frontend/src/components/ToolInput.vue` |

---

### 问题 #017: HMAC 验签要求 secret 明文,存哈希(不可逆)无法验签

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 实现 36 号 §7.6.3 对外卡密 Open API 时,设计说「api_secret 只存哈希」,但 HMAC-SHA256 验签需要 secret 明文做密钥,存哈希(不可逆)根本无法验签 |
| **根因** | HMAC 验签 = `HMAC(secret, msg)`,服务端必须持有 secret 明文才能重算签名比对。哈希是单向的,存哈希无法还原 secret。设计文档「只存哈希」与「HMAC 验签」自相矛盾 |
| **修复** | 改为**对称加密存储**(Fernet,key 从 JWT_SECRET 派生):入库前 encrypt_secret(明文),验签时 decrypt_secret(密文)。明文仅在生成时一次性返回给外部系统。仍非明文落库,丢失需 rotate |
| **预防** | 凡需服务端重算 HMAC 的场景,secret 必须**可还原**(对称加密),不能存单向哈希。单向哈希只用于「服务端只验证不持有」的场景(如用户密码登录) |
| **文件** | `backend/app/modules/card_key/auth.py`(encrypt_secret/decrypt_secret) |

### 问题 #018: api_secret_hash 列长度 128 不够装 Fernet 密文

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 建 distributor 时报 `value too long for type character varying(128)`。36 号 §7.6.2 设计 `api_secret_hash String(128)`(按哈希长度估),改 Fernet 加密后密文 ~180 字符超长 |
| **根因** | 列长度按「存哈希」估的 128,改加密存储后 Fernet 密文含 IV+时间戳+HMAC,远超 128 |
| **修复** | `card_key_distributors.api_secret_hash` 改 `String(256)`(models.py + 迁移 0006 + 已跑库 ALTER TYPE 三方同步) |
| **预防** | 设计 String(N) 列时考虑存储内容实际长度;哈希 vs 加密密文长度差很大;迁移已跑过改列要 ALTER + 同步迁移文件 + ORM(#007 三方) |
| **文件** | `backend/app/modules/card_key/models.py` + `alembic/versions/0006_open_api.py` |

### 问题 #019: .env 缺 POSTGRES_HOST 导致 app/alembic 连 localhost 失败

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | `alembic upgrade` 报 `connection to server at 127.0.0.1 port 5432 failed`,但 api 容器显示 healthy |
| **根因** | WSL 侧 `.env` 只有 7 个 key(无 POSTGRES_HOST/PORT/REDIS_HOST/MINIO_ENDPOINT),全用 config.py 默认 `localhost`。Docker 容器内 localhost 是容器自己,连不到 postgres 容器(要服务名 `postgres`)。api「healthy」是假象——healthcheck 只查进程 curl,不查 DB 连接(数据库懒加载) |
| **修复** | `.env` 追加 `POSTGRES_HOST=postgres` `POSTGRES_PORT=5432` `REDIS_HOST=redis` `MINIO_ENDPOINT=minio:9000`(docker 服务名) |
| **预防** | Docker 网络内服务互连必须用**服务名**不是 localhost;healthcheck 应查真实依赖(DB ping)而非只查进程,否则「healthy」会掩盖连接失败;.env 需含全部 host 配置,别全靠默认值 |
| **文件** | `/root/tool-site-project/.env`(WSL 侧,不入 git) + `backend/app/api/health.py`(healthcheck 建议加 DB ping) |

---

### 问题 #020: 错误码 E407 漏定义,raise_error 走 fallback 变 E500

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 非管理员调 admin 接口返回 500(E500「服务异常」),而非预期 403(E407「需要管理员权限」) |
| **根因** | 36 号 §4.2 定义了 E407(管理员权限不足),但补错误码时只加了 E104-E106 + E501-E507,**漏了 E407**。`raise_error("E407")` 因 E407 不在 ERROR_CODES 走 fallback → `error_code="E500"`(500),掩盖了真实语义 |
| **修复** | error_codes.py 补 E407: `("管理员权限不足","需要管理员权限",403,False)` |
| **预防** | 补错误码时按 36 号 §4.2 错误码表**逐条核对**,新错误码定义后用 `grep` 确认所有 raise_error 调用的 code 都已定义;`raise_error` 的 fallback 是 E500,会掩盖「未定义错误码」的 bug,易误判为业务异常 |
| **文件** | `backend/app/error_codes.py` |

---

### 问题 #021: pay_order_success 生成卡密 batch_id 旧值触发 FK 违反

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 闲鱼手动确认收款(confirm-paid)报 500:`ForeignKeyViolation insert on card_keys violates fk_card_keys_batch_id` |
| **根因** | `pay_order_success` 仍用旧逻辑 `batch_id=f"order-{order.order_no}"`,但迁移 0006 已把 batch_id 改为 FK→card_key_batches。`order-ZN2026...` 值在批次表不存在,FK 违反。这是评审硬伤2(batch_id 改 FK)在发卡路径的遗漏——只改了迁移/模型,没改发卡代码 |
| **修复** | `pay_order_success` 生成卡密时 `batch_id=None`(订单发的卡密不归属批次,批次是管理员批量生成用);顺带清 `__import__("datetime")` hack 改正常 import timedelta(硬伤3 部分) |
| **预防** | 改字段语义(如 batch_id 自由 String→FK)后,必须 `grep -rn` 搜所有写该字段的代码(不止兑换,还有发卡/生成),全改;迁移+模型+所有写入点三方同步(#007 延伸) |
| **文件** | `backend/app/services/order_service.py`(pay_order_success) |

---

### 问题 #022: create_order channel 白名单漏新渠道,checkout 报 E404

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | P1 checkout 用 wechat_native 渠道报 E404「支付渠道 wechat_native 不支持」,但渠道适配器 CHANNELS 注册正常(get_channel 直接测 OK) |
| **根因** | `create_order` 的 channel 白名单是旧枚举 `("wechat","alipay","xianyu","pdd")`,P1 新渠道适配器用 `wechat_native/xunhupay/payjs/alipay_face`,白名单没同步扩展。checkout 先调 create_order 校验 channel,在白名单外就 E404,根本到不了 get_channel |
| **修复** | create_order channel 白名单加 `wechat_native/xunhupay/payjs/alipay_face` |
| **预防** | 新增渠道时,渠道适配器 CHANNELS 注册 + create_order 白名单 + openapi.yaml 枚举 三处同步(不只是适配器);channel 枚举是 SSOT,散落多处易漏 |
| **文件** | `backend/app/services/order_service.py`(create_order) |

---

### 问题 #023: 批量补错误码仍漏定义(E302-E305/E408),raise 走 fallback E500

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | P2 用了 E408(不能吊销自己)/E302(验证码错误)/E303(验证码过期),但 error_codes.py 没定义,raise_error 走 fallback E500(同 #020) |
| **根因** | 36 号 §4.2 错误码表分批补(E104-E106/E501-E507/E407),但 E302-E305(验证码)+E408(吊销自己)+E409(注销)这组用户系统错误码漏补。`raise_error` 未定义码 fallback E500 掩盖问题 |
| **修复** | 补 E302-E305(验证码/邮箱/手机)+ E408(不能吊销自己)+ E409(注销冷却期) |
| **预防** | **补错误码时按 36 号 §4.2 全表逐条核对,不能只补当前任务用到的**;每用新错误码前先确认已定义;#020 已犯同样错,本次复发说明需建立「错误码清单 checklist」对照 36 号 §4.2 |
| **文件** | `backend/app/error_codes.py` |

---

### 问题 #024: 运维脚本依赖 app 模块但容器未挂载 scripts 目录

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | cleanup_expired_cards.py / generate_cards.py --from-orders 在容器内跑报 `can't open /app/scripts/...`(文件不存在)或 `No module named 'app'`(路径不对) |
| **根因** | 容器只 COPY backend/app,Dockerfile 不 COPY 项目根 scripts/;脚本既依赖 app 模块(需在 /app 工作目录)又在 scripts/(未进容器),路径冲突 |
| **修复** | 运维逻辑做成 **admin API 端点**(cleanup-expired / xianyu-batch-ship),定时任务调端点,不依赖脚本路径;脚本仍保留供本地/宿主跑 |
| **预防** | 容器内运维任务优先用 API 端点(无路径依赖,有鉴权);脚本仅作本地批处理;需容器跑的脚本要 COPY 进镜像且在正确工作目录 |
| **文件** | `backend/app/modules/card_key/api.py`(cleanup-expired/xianyu-batch-ship 端点) |

---

### 问题 #025: 新增端点用 Query 等依赖但顶部未 import,致 api 启动 crash loop

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | P4 新增 admin/dashboard 等端点后,api 容器反复重启(crash loop),日志 `NameError: name 'Query' is not defined`(payment.py 第 403 行) |
| **根因** | 端点用了 `Query(...)` 参数但 payment.py 顶部 `from fastapi import ...` 没含 Query。import 缺失致模块加载失败,整个 app 启不起来(不只该端点) |
| **修复** | payment.py 顶部 import 补 `Query`(同前 #020/#023 类:新增代码用到的依赖未 import) |
| **预防** | 新增端点用到 Query/Body/Cookie 等 fastapi 依赖时,先确认顶部已 import;**crash loop + NameError 多为 import 缺失**,看日志首行错误即可定位;写完端点先 `python -c "import app.main"` 本地 import 自检再部署 |
| **文件** | `backend/app/api/payment.py` |

---

### 问题 #026: 挂载的 .env 用 os.replace 跨 inode 替换报 Device or resource busy

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 后台支付接入页写 .env 报 `E508 Device or resource busy: '.env.tmp.1' -> '.env'`,os.replace 临时文件替换挂载文件失败 |
| **根因** | .env 通过 docker volume 挂载(./.env:/app/.env),挂载文件不能被 os.replace 跨 inode 替换(Windows/WSL2 挂载层禁止 rename 跨到挂载点),原"写临时文件再 replace"的原子写方案对挂载文件失效 |
| **修复** | 改原地覆盖写:读原内容到内存备份 → open(.env,'w') 直接覆盖写(同 inode,挂载文件可写)→ 失败回写备份。失去跨文件原子性,但挂载文件只能如此,靠内存备份兜底 |
| **预防** | 挂载文件(volume mount)的写入不能用 os.replace/rename 跨文件,只能同 inode 覆盖;原子写要靠"备份+覆盖+回写"而非"临时文件+rename";开发期用本地 .env(非挂载)测不出此坑,部署到挂载环境才暴露 |
| **文件** | `backend/app/modules/config/service.py`(write_env) |

### 问题 #027: Stirling 转换链路从未接通,12 个工具"假就绪"

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | registry 挂 backend=stirling 的 12 个工具(pdf-merge/split/compress/to-image/ocr/decrypt/watermark/scanned/excel-to-pdf/ppt-to-pdf/ebook-convert/audio-convert),用户点转换后任务永远卡 queued,永不完成 |
| **根因** | convert.py 把 stirling 任务推入 queue:stirling(Redis),但 docker-compose 无任何 stirling-worker 消费者,后端也无队列消费代码(grep 全项目零结果),Stirling 容器此前从未启动。P3"Stirling+10工具"一直待做未落地 |
| **修复** | 写 stirling-worker(Python 队列消费者):brpop queue:stirling → 调 Stirling HTTP API(X-API-KEY)→ 结果存 MinIO → 回写 task 状态。设计见 38-competitor-and-conversion-gap.md §五 |
| **预防** | registry 的 backend 字段必须有对应消费者;CI 应校验"backend=X 且无对应 worker 服务"为 fail,避免"挂了字段就算实现"的假就绪 |
| **文件** | `backend/app/api/convert.py`(入队) · `backend/app/redis_client.py`(WORKER_QUEUES) · `docker-compose.yml`(待加 stirling-worker) |

### 问题 #028: Stirling API 需 X-API-KEY 即使 DOCKER_ENABLE_SECURITY=false

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 已配 DOCKER_ENABLE_SECURITY=false,调 Stirling /v1/api-docs 仍 401 Authentication required |
| **根因** | Stirling v2.14.2 新版 enableLogin 默认开,SECURITY=false 只关用户登录增强,API 认证仍要求 key;无预设 globalApiKey |
| **修复** | docker-compose 加环境变量 SECURITY_CUSTOMGLOBALAPIKEY=zhiniao_stirling_key_2026 并重建容器 |
| **预防** | 接 Stirling 必先验认证;SECURITY=false ≠ 无认证,新版默认 enableLogin=true 仍需 key;worker 调用必带 X-API-KEY 头 |
| **文件** | `docker-compose.yml`(stirling.environment) |

### 问题 #029: Stirling 新版 SaaS license 限 5 用户(商业化前必须解决)

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | Stirling 启动日志 License check: requiresPaid=true, hasPaid=false;GRANDFATHERING LOCKED: 5 users |
| **根因** | Stirling v2.14.2 改商业模式,SaaS 模式(对外提供服务)需付费 license,免费版锁 5 用户 |
| **修复** | P0 用当前版本跑通验证;商业化上线前购 license 或锁定旧版/评估替代,详见 38 号 §6.1 |
| **预防** | 引入开源依赖前先核 license 条款(尤其商业模式);开源 ≠ 可任意商用,注意 AGPL/SaaS 限制类条款 |
| **文件** | `docker-compose.yml`(stirling) · `38-competitor-and-conversion-gap.md`(§6.1) |

### 问题 #030: audio-convert 是假工具(Stirling 无音视频端点)

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | registry 的 audio-convert 挂 backend=stirling,实测 Stirling v2.14.2 共 259 端点,grep audio\|video\|mp3\|wav 零命中——Stirling 是纯 PDF/文档工具,无任何音视频能力 |
| **根因** | 加工具时按"Stirling 能转多格式"的笼统认知挂了 backend,未核对具体端点是否支持音视频 |
| **修复** | audio-convert 改 backend=ffmpeg-worker(独立 FFmpeg worker,见 38 号 §五 P2)或暂 deprecated;音频用 MP3/FLAC/OGG 免专利,AAC 有专利池 |
| **预防** | 挂 backend=stirling 前必须确认对应端点真实存在(跑 test_stirling_endpoints.py);Stirling 不碰音视频,音视频走独立 FFmpeg worker |
| **文件** | `schemas/tools.registry.json`(audio-convert) · `scripts/test_stirling_endpoints.py` |

### 问题 #031: deploy_sync.sh rsync --delete 会删除仅存于 WSL 的脚本

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | test_stirling_endpoints.py 最初只写在 WSL;deploy_sync.sh 第 78 行 rsync -a --delete(Windows→WSL)同步 scripts/,下次部署会用 --delete 删掉 WSL 独有脚本 |
| **根因** | deploy_sync 以 Windows 为权威单向同步,--delete 删目标侧(WSL)有而源侧(Windows)无的文件;在 WSL 直接写的脚本没回写 Windows 权威侧 |
| **修复** | 把脚本 cp 回 Windows scripts/(成权威副本),两副本一致后 --delete 不再误删 |
| **预防** | 凡在 WSL 新建/修改的同步范围内文件(scripts/frontend/schemas/nginx)必须同步回 Windows,否则 deploy_sync --delete 丢失;docker-compose 不在同步列表需手动双副本一致 |
| **文件** | `scripts/deploy_sync.sh`(第 78 行) · `scripts/test_stirling_endpoints.py` |

### 问题 #032: httpx 同时传 files= 和 data=(list-of-tuples) 会破坏 multipart

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | stirling-worker 调 Stirling:① `data=[]` 时 Stirling 回 415 `Media type null is not supported...multipart/form-data`(请求没设 multipart Content-Type);② `data=[("dpi","150")]` 时 httpx 抛 `TypeError: sequence item 1: expected bytes-like, tuple found`。改成 `data=dict` 或 omit 即 200 |
| **根因** | httpx 在 `files=` 存在时,`data` 若传 **list-of-tuples** 会走错编码路径(不设 multipart boundary / 崩在 httpcore 拼装);只有 **dict**(含空 dict `{}`)或省略才正确构造 multipart。不能照搬 requests 的 list-of-tuples 写法 |
| **修复** | `call_stirling` 用 `data=dict(form_fields)`(route_map builder 返回 list[(k,v)],转 dict);实测 A-J 验证:dict 200、list 415/崩 |
| **预防** | httpx 同时传 files + 表单字段时,`data` 必须是 dict,不是 list-of-tuples;写完用 `client.build_request(...).headers['content-type']` 自查是 multipart/form-data; boundary |
| **文件** | `workers/stirling/worker.py`(call_stirling) |

### 问题 #033: 后台支付接入页写 .env 字段错位,致 api crash loop

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 重建 api 后 crash loop,日志 `pydantic ValidationError: SMTP_PORT Input should be a valid integer, input_value='admin@bibilabu.cc'`。查 .env:`SMTP_HOST=smtpdm.aliyun.com:465` / `SMTP_PORT=admin@bibilabu.cc` / `SMTP_USER=KvW2zCZqX6WbG3P6`(值整体上移一位,SMTP_PASSWORD 丢失)。旧 api 进程用内存旧 settings 一直没崩,重建重读 .env 才暴露 |
| **根因** | `write_env` 本身忠实按 key=value 逐行写入(无误,见 service.py L125-138);错位源于**提交侧**——后台支付接入页(PaymentConfig.vue/config api)把表单值映射到了错的字段名,提交 dict 已错位(write_env 忠实写入错位值) |
| **修复** | 手动还原 .env SMTP 段为正确值(SMTP_HOST/PORT/USER/PASSWORD),已备份 `.env.bak_stirlingfix`;api 重建后 200 |
| **预防** | 后台配置页表单字段名必须与 config.py/.env 字段名严格一一对应;提交前校验类型(SMTP_PORT 应 int);重建 api 前先验 .env 关键字段;pydantic 启动 ValidationError 多为 .env 类型/错位 |
| **文件** | `.env`(SMTP 段) · `backend/app/modules/config/{service,api}.py`(write_env 无误,待查 api 提交映射) · `frontend/.../PaymentConfig.vue`(待查表单绑定) |
| **闭环(2026-07-16)** | 见下「#033 闭环」段:源码 v-model 按字段名绑定(无误,疑旧部署产物错位)+ 后端加 INT_FIELDS 类型防线 + SMTP 真发接入 |

### #033 闭环(2026-07-16,SMTP 字段错位 + 邮件发送接入)

**根因复核**(本次实地核查源码,非再猜):
- `PaymentConfig.vue` 当前 `v-model="editForm[f.field]"`(L41)+ `saveEdit` 按 `f.field` 收集(L117-119),**已按字段名绑定,无误**。#033 当时的"值上移一位"在当前源码下不会发生 → 极可能来自**旧部署前端构建产物**(曾按位置/index 绑定),源码后改对但 #033 未记录该修正、未重建前端。
- 真正缺失的防线:后端 `write_env` **无类型校验** → 前端(旧版)错位提交的脏数据(`SMTP_PORT='admin@bilibabu.cc'`)直接写进 .env → 重启重读 → `pydantic ValidationError` → api crash loop。旧进程用内存 settings 不崩,重建才暴露(与 #033 现象完全吻合)。

**本次修复**(前后端双重防线 + 接通真发):
1. 后端 `config/service.py` 加 `INT_FIELDS={"SMTP_PORT"}`,`write_env` 写前校验非空 int 字段为数字,否则 `E403` 拒写(**核心防线:即使前端再错位,脏数据也不进 .env,不再 crash loop**)。
2. 新建 `backend/app/utils/email.py`(36 号 §5.4 模块化要求):`parse_host_port` 容错 `host:port` 合体(用户填 `smtpdm.aliyun.com:465` 到 HOST 也能用)、`send_email`(SSL,`SMTP_FROM` 空→回退 `SMTP_USER`,适配阿里云 DM 发信地址即账号)、`send_verify_code`。
3. `auth.py` `_send_email_code` 从 `print` 占位改为调 `send_verify_code` 真发;配置不全/发送失败降级控制台打印不阻断(找回密码防枚举仍统一回"已发送")。**接 38 号 §四占位**。
4. `config/service.py` `_test_smtp` 用 `parse_host_port` 容错读取(测试连通也能识别 host:port 合体)。
5. 前端 `PaymentConfig.vue`:`saveEdit` 提交前校验 `SMTP_PORT` 为数字(前端第一道防线)+ `placeholderFor` 给 `SMTP_HOST` 提示"只填主机名,端口填 SMTP_PORT"。

**部署生效前提**:Windows 副本改 → 同步 WSL 副本(deploy_sync)→ 重建 api(后端改)+ 重建前端(前端改,否则旧产物仍错位)→ 重启。.env SMTP 段当前已手动还原正确(HOST=smtpdm.aliyun.com/PORT=465/USER/PASSWORD,缺 FROM 由回退 USER 兜底)。

**预防追加**:配置写 .env 的 int 字段必须后端类型校验(不能只信前端);占位代码接入时同步核查"源码对但部署旧"的可能(grep TODO + 比对构建产物)。

### 问题 #034: convert.py expires_at 用 now.replace(hour=now.hour+1) 23 点后崩溃

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | 23:00-23:59 UTC 创建转换任务,convert.py 抛 `ValueError: hour must be in 0..23`,API 500,任务建不出(stirling-worker 再通也用不了) |
| **根因** | `now.replace(hour=now.hour + 1)` 当 hour=23 → hour=24 越界;`datetime.replace` 不做进位(不跨日) |
| **修复** | 改 `(now + timedelta(hours=1)).isoformat()`(timedelta 自动进位跨日),补 `timedelta` import |
| **预防** | 时间加减用 `timedelta`,不用 `replace(hour=...)` 手动算;replace 不进位,边界值(23 点/月末)必越界 |
| **文件** | `backend/app/api/convert.py`(L174) |

### 问题 #035: registry 加 stirling 工具须同步 file_validator.INPUT_TYPES,否则 API 拒上传

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-16 |
| **现象** | pdf-to-scanned / excel-to-pdf / ppt-to-pdf / ebook-convert 上传报 `E404 工具 X 不支持文件上传`(任务根本进不了队列) |
| **根因** | convert.py → `validate_upload` 查 `file_validator.INPUT_TYPES.get(tool_id)`,返回 None 则 E404 拒上传。这 4 个工具在 registry 有 backend=stirling 但 INPUT_TYPES 漏登(一致性链断裂:registry ↔ file_validator) |
| **修复** | `file_validator.INPUT_TYPES` 补这 4 个及允许扩展名(pdf-to-scanned=.pdf / excel-to-pdf=.xlsx,.xls / ppt-to-pdf=.pptx,.ppt / ebook-convert=.pdf)。audio-convert 假工具不入 → API 早拒(正确,假工具不该收上传) |
| **预防** | 加工具到 registry 时,若需后端上传,必须同步 `file_validator.INPUT_TYPES`;一致性链:registry.backend ↔ WORKER_QUEUES ↔ file_validator.INPUT_TYPES 三处对齐 |
| **文件** | `backend/app/utils/file_validator.py`(INPUT_TYPES) · `schemas/tools.registry.json` |

### 问题 #036: 微信支付 V3 签名串字段顺序错误(method/path 应在前)

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-17 |
| **现象** | 微信 V3 下单/查单返回 `SIGN_ERROR 签名错误,请检查后再试`,Authorization 头格式/证书序列号/body 字节全核对正确仍失败 |
| **根因** | 签名串字段顺序写错。本地构造 `{timestamp}\n{nonce}\n{method}\n{path}\n{body}\n`,微信官方规范是 `{method}\n{path}\n{timestamp}\n{nonce}\n{body}\n`(method+path 在前,timestamp+nonce 在后)。微信报错 `sign_message_length:66` + `truncated_sign_message` 直接展示了正确顺序,对比即定位 |
| **修复** | `_sign_v3` 改 `f"{method}\n{path}\n{timestamp}\n{nonce}\n{body}\n"`,修后 GET /v3/certificates 立即返回非 SIGN_ERROR(转为业务错误),下单拿到真实 code_url |
| **预防** | 微信 V3 签名串顺序记牢:`{method}\n{url_path}\n{timestamp}\n{nonce}\n{body}\n`;回调验签串不同(无 method/path):`{timestamp}\n{nonce}\n{body}\n`;报 SIGN_ERROR 时看微信返回的 `detail.sign_information.truncated_sign_message` 直接给正确串,对比即定位。回调验签用平台证书/微信支付公钥,与请求签名(商户私钥)方向相反 |
| **文件** | `backend/app/modules/payment/channels/wechat_native.py`(`_sign_v3`) |

### 问题 #037: 微信支付新模式(微信支付公钥)替代平台证书,回调验签需改用公钥

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-17 |
| **现象** | 签名修对后,GET /v3/certificates 返回 `RESOURCE_NOT_EXISTS 无可用的平台证书,请在商户平台-API安全申请使用微信支付公钥`。新商户号无平台证书可下载,旧自动下载证书模式失效 |
| **根因** | 微信 2024+ 推「微信支付公钥」模式替代平台证书。新模式:回调/应答验签用商户平台下载的微信支付公钥(PEM `-----BEGIN PUBLIC KEY-----`),回调头 `Wechatpay-Serial` 值形如 `PUB_KEY_ID_xxx`(以 PUB_KEY_ID_ 前缀,区别平台证书的 hex 序列号)。GET /v3/certificates 在新模式商户下无可用证书 |
| **修复** | `_verify_platform_sig` 改双模式:A) `serial` 以 `PUB_KEY_ID_` 开头→公钥模式,读 `WECHAT_PUB_KEY_PATH`(`load_pem_public_key`)验签;B) 否则→平台证书模式(自动下载回退)。config 加 `WECHAT_PUB_KEY_PATH`;签名串两模式都是 `{ts}\n{nonce}\n{body}\n`(回调无 method/path)。Authorization 头 serial_no 仍填商户 API 证书序列号(不变) |
| **预防** | 接微信 V3 先判模式:看回调头 `Wechatpay-Serial` 前缀(`PUB_KEY_ID_` vs hex);新商户默认公钥模式,需商户平台下载公钥(.pem)+ 记公钥ID 放 secrets;无公钥时容错放行+告警(回调路由二次校验订单状态/金额防伪造),不阻断支付 |
| **文件** | `backend/app/modules/payment/channels/wechat_native.py`(`_verify_platform_sig`) · `backend/app/config.py`(`WECHAT_PUB_KEY_PATH`) |

### 问题 #038: card_redeem_logs.card_key_id NOT NULL 但 _log_action 设计允许 None,触发约束违反

| 项 | 内容 |
|----|------|
| **日期** | 2026-07-20 |
| **现象** | 注册赠送/邀请/签到调 `_log_action(db, None, user_id, "register_gift"/"invite_reward"/"checkin", ...)` 落 card_redeem_logs,报 `NotNullViolation: null value in column "card_key_id"`,注册 500 |
| **根因** | 迁移 0003 `card_redeem_logs.card_key_id` 定义 `nullable=False`(L29),但 `card_key/service.py:_log_action` 设计 `card_id: Optional[str]` 允许 None(注释:"格式错/管理员作废/外部核销无 card_id")。设计与表结构矛盾——既有 bug,正常 redeem 格式错分支(`_log_action(None,...)`)也会触发,只是格式错卡密罕见没暴露 |
| **修复** | 注册赠送/邀请/签到改为**不落 card_redeem_logs**(card_key_id NOT NULL + 非卡密场景);审计靠 invitation_records/daily_checkin 表 + balances.total_redeemed 累计 |
| **预防** | 既有 bug 待彻底修:要么表加 `nullable=True`(迁移),要么 `_log_action` 强制 card_id 非空(改设计)。新增长机制奖励不要塞卡密专用日志表,用各自业务表审计 |
| **文件** | `backend/app/modules/card_key/service.py`(`_log_action`) · `backend/alembic/versions/0003_card_redeem_logs.py`(L29) · `backend/app/api/auth.py`(register) · `backend/app/modules/growth/service.py` |

---

## 五、维护规范

### 5.1 何时记录

- 部署/构建失败
- 功能不工作且排查 > 5 分钟
- 发现反直觉的坑
- 用户报告问题

### 5.2 记录格式

```markdown
### 问题 #NNN: 简短标题

| 项 | 内容 |
|----|------|
| **日期** | YYYY-MM-DD |
| **现象** | 用户可见的症状 |
| **根因** | 技术原因 |
| **修复** | 具体改了什么 |
| **预防** | 如何避免再犯 |
| **文件** | 涉及文件路径 |
```

### 5.3 编号规则

- 顺序编号 #001、#002...
- 不复用编号(即使问题已解决)
- 同类问题可在"关键雷区"合并引用

---

## 六、相关文档

- [21-deployment-validation.md](./21-deployment-validation.md) — 部署验证记录
- [26-log-collection.md](./26-log-collection.md) — 日志收集与 AI 调试
- [19-runbook.md](./19-runbook.md) — 运维故障处理
