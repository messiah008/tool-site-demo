# 错题集 (Lessons Learned)

> **版本**: v1.0
> **最后更新**: 2026-07-03
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
