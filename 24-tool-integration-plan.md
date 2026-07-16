# 120 工具合入方案

> **版本**: v1.0
> **最后更新**: 2026-07-03
> **状态**: 合入方案总纲(平台转型 22 号文档的执行落地)
> **依赖**: [22-platform-pivot-strategy.md](./22-platform-pivot-strategy.md), [07-consistency-link.md](./07-consistency-link.md), [06-tool-registry.md](./06-tool-registry.md)
> **目的**: 把 120 个便民工具融入现有部署框架,保证兼容/易用/可维护

---

## 一、合入目标与约束

### 1.1 目标

把 22 号文档的 120 工具清单,合入现有 `tool-site-project` 框架,做到:

| 目标 | 衡量标准 |
|------|---------|
| **兼容** | 新工具不破坏现有 PDF→Word / 卡密 / 用户中心 |
| **易操作** | 加新工具 ≤ 30 分钟(纯前端) |
| **可维护** | 改一处 schema,路由/卡片/SEO/互链全同步 |
| **一致性** | 遵循 07 号文档四大 link,CI 强制校验 |

### 1.2 硬约束(不可违背)

1. **不重写现有架构** — 复用已有的 Vue3 + FastAPI + Docker Compose
2. **不破坏 SSOT** — 所有工具元数据只在 `tools.registry.json`
3. **不改 backend 业务逻辑** — 纯前端工具不碰后端
4. **不引入新基础设施** — 用现有 Redis/PostgreSQL/MinIO,不另起 SQLite(注:22 号文档提到 SQLite,但现有框架已用 PostgreSQL,沿用即可,见 4.3)
5. **工具数量上限 120** — 超过维护崩溃(22 号文档 8.3 反模式)

### 1.3 与 22 号文档的差异(务实调整)

| 22 号文档建议 | 本方案调整 | 理由 |
|--------------|----------|------|
| SQLite 数据库 | 沿用 PostgreSQL | 现有框架已部署,不重复造轮子 |
| 5 类 Worker 简化 | 保留 wps-worker + stirling | 22 号 8.3 反模式:微服务过早 |
| Prometheus+Grafana | 保留但默认关闭 | 22 号 8.3:月收入<700 时运维成本>收益 |
| Cloudflare 泛解析 | 保留 | 免费且必要 |
| 子域名 SEO | 保留(高频工具) | 见 3.2 URL 策略 |

---

## 二、合入核心机制:复用现有 link

120 工具的合入**完全依赖 07 号文档的四大 link**,不另建机制:

```
┌─ 链接 1: 设计一致性 ─────────────────────────────┐
│  design-tokens.json → tokens.css → 所有工具页     │
│  (新工具自动用相同色板/间距/圆角,视觉统一)        │
├─ 链接 2: 工具一致性 ─────────────────────────────┤
│  tools.registry.json → 路由/卡片/SEO/互链 自动生成│
│  (加工具只改 JSON,前端自动渲染)                  │
├─ 链接 3: 接口一致性 ─────────────────────────────┤
│  openapi.yaml → 纯前端工具不涉及;后端工具走契约   │
├─ 链接 4: AI 上下文一致性 ───────────────────────┤
│  CLAUDE.md → 所有 AI 按 25 号 SOP 操作工具        │
└──────────────────────────────────────────────────┘
```

**关键**: 现有框架的 `gen-tools.mjs` 已能从 registry 生成路由和卡片,120 工具合入**只需扩展 registry + 创建页面文件**,框架代码零改动。

---

## 三、兼容性设计

### 3.1 工具分类与现有 backend 映射

22 号文档的 120 工具按实现方式分 5 类,映射到现有 `backend` 字段:

| 22 号标识 | 含义 | 现有 backend 值 | 处理方式 |
|----------|------|----------------|---------|
| 🟢 纯前端 | 浏览器运行 | `frontend` | 只写 Vue 页面,不碰后端 |
| 🟡 静态数据 | 前端 + JSON | `frontend` | JSON 放 `public/data/`,前端 fetch |
| 🟠 内置数据库 | 查询类 | `stirling`(借用)或新增 `db-query` | 走 FastAPI 查 PostgreSQL |
| 🔴 需后端 API | 第三方代理 | 新增 `proxy` | FastAPI 代理层 |
| ⚠️ 合规风险 | 涉隐私 | **不合入** | 见 3.5 |

### 3.2 URL 策略(混合,与 22 号 5.2 一致)

| 工具类型 | URL | 路由示例 | Nginx |
|---------|-----|---------|-------|
| 高频免费(密码/UUID/Base64) | 子域名 | `mima.zhiniao.tools` | 通配 server_name |
| 普通免费 | 子路径 | `zhiniao.tools/tools/uuid` | 主站 server |
| 付费工具 | 子路径 | `zhiniao.tools/pdf-to-word` | 主站 server |
| 用户中心 | 子路径 | `zhiniao.tools/user-center` | 主站 server |

**实现**: Nginx 已有 `*.zhiniao.tools` 通配 server,只需 Cloudflare 泛解析 `*` → VPS。

### 3.3 路由兼容(零冲突)

现有 `frontend/src/router/index.ts` 已从 registry 自动生成路由。120 工具合入后:

```typescript
// 现有逻辑(无需改)
const implementedTools = new Set(['pdf-to-word'])  // ← 只加已实现的 ID

const toolRoutes = tools
  .filter(t => !t.deprecated)
  .map(tool => ({
    path: tool.route,
    name: tool.id,
    component: implementedTools.has(tool.id)
      ? () => import(`@/tools/${tool.id}/index.vue`)
      : () => import('@/tools/_placeholder/index.vue'),  // 未实现兜底
    meta: { tool },
  }))
```

**合入新工具流程**:
1. registry 加工具定义
2. 创建 `frontend/src/tools/{id}/index.vue`
3. 把 ID 加入 `implementedTools` Set
4. `npm run gen:tools` 重新生成路由

### 3.4 互链机制(bmcx 模式核心)

22 号文档 5.3 的互链算法,在现有 `ToolLayout.vue` 基础上扩展:

```vue
<!-- frontend/src/layouts/ToolLayout.vue 现有相关工具 -->
<div class="related-grid">
  <RouterLink v-for="t in relatedTools" :key="t.id" :to="t.route" class="related-card">
```

扩展为 bmcx 模式(每页 70 个互链):

```typescript
// 新增 computed: 70 个互链(35 同类 + 35 随机)
const relatedTools = computed(() => {
  const sameCategory = activeTools
    .filter(t => t.category === props.tool.category && t.id !== props.tool.id)
    .slice(0, 35)
  const others = activeTools
    .filter(t => t.id !== props.tool.id && !sameCategory.includes(t))
    .sort(() => Math.random() - 0.5)
    .slice(0, 35)
  return [...sameCategory, ...others]
})
```

**好处**: 加新工具自动进入所有页面互链,无需手动维护。

### 3.5 不合入的工具(合规/过时)

按 22 号文档 2.2,以下不合入:

| 工具 | 不合入理由 |
|------|----------|
| 同 IP 域名查询 | 涉隐私,合规风险 |
| 服务器系统识别 | 受众小,工具站长才用 |
| 在线五笔输入法 | 已过时 |
| 论坛转贴工具 | BBS 时代产物 |
| 迅雷/快车链接加密 | 下载工具没落 |
| Google PR 查询 | PR 已停用 |

**registry 中可标记 `deprecated: true`** 保留外链兼容,但不渲染卡片。

---

## 四、易操作性设计

### 4.1 工具页面模板(标准化)

所有工具页遵循统一模板,降低开发门槛:

```vue
<!-- frontend/src/tools/_template.vue -->
<template>
  <ToolLayout :tool="tool">
    <!-- 1. 输入区(标准化) -->
    <ToolInput v-model="input" :schema="tool.options" />

    <!-- 2. 操作按钮 -->
    <ToolButton @click="process" :loading="loading">
      {{ tool.action_label || '执行' }}
    </ToolButton>

    <!-- 3. 结果区(标准化) -->
    <ToolResult v-if="result" :result="result" :copyable="true" />

    <!-- 4. 选项区(可选) -->
    <ToolOptions v-if="tool.options" v-model="options" :schema="tool.options" />
  </ToolLayout>
</template>
```

**通用组件**(已有/新增):
- `ToolInput` — 文本框/文件上传/选择器,根据 schema 自动渲染
- `ToolButton` — 统一按钮样式
- `ToolResult` — 结果展示 + 一键复制
- `ToolOptions` — 选项面板

### 4.2 registry 字段扩展(见 5.1)

为支持 120 工具的多样性,扩展 schema:

```json
{
  "id": "random-password",
  "name": "随机密码生成",
  "category": "network",
  "icon": "🔑",
  "icon_color": "calc",
  "is_premium": false,
  "backend": "frontend",
  "route": "/random-password",
  "subdomain": "mima",
  "keywords": ["随机密码", "密码生成器"],
  "implementation": "frontend",
  "dependencies": ["crypto-js"],
  "difficulty": "low",
  "priority": 3,
  "action_label": "生成密码",
  "input_schema": [
    { "name": "length", "type": "number", "default": 16, "label": "密码长度" },
    { "name": "charset", "type": "checkbox-group", "options": ["a-z","A-Z","0-9","符号"], "default": ["a-z","A-Z","0-9"] }
  ],
  "related_tools": ["password-strength", "uuid-generator"],
  "source": { "type": "original" },
  "about": { "title": "关于随机密码生成", "content": "..." },
  "seo": { "json_ld": {...}, "faq": [...] }
}
```

### 4.3 数据文件管理

22 号文档 4.2 的数据文件,统一放 `frontend/public/data/`:

```
frontend/public/data/
├─ phone.json          # 手机归属(5MB)
├─ admin-divisions.json # 行政区划(2MB)
├─ holidays.json       # 节假日(5KB)
├─ idioms.json         # 成语(3MB)
├─ poetry.json         # 诗词(20MB)
├─ food-calorie.json   # 食物卡路里(1MB)
└─ elements.json       # 元素周期表(50KB)
```

**前端 fetch**(纯前端工具):
```typescript
const data = await fetch('/data/idioms.json').then(r => r.json())
```

**注意**: 大文件(poetry.json 20MB)用懒加载,只在工具页加载,不进首页 bundle。

### 4.4 依赖管理

22 号文档 4.1 的 npm 依赖,统一加到 `frontend/package.json`。为控制 bundle 体积:

| 依赖 | 体积 | 加载策略 |
|------|------|---------|
| dayjs/lunar-javascript | 中 | 按需 import |
| crypto-js | 中 | 工具页动态 import |
| poetry.json 数据 | 大(20MB) | fetch,不 import |
| prettier/terser | 大 | 动态 import,仅格式化工具用 |

**vite.config.ts** 配置 manualChunks 分割(解决之前 1MB 警告):
```typescript
build: {
  rollupOptions: {
    output: {
      manualChunks: {
        'vendor-vue': ['vue', 'vue-router', 'pinia'],
        'vendor-element': ['element-plus'],
        'vendor-crypto': ['crypto-js'],
        'vendor-lunar': ['lunar-javascript'],
      }
    }
  }
}
```

---

## 五、可维护性设计

### 5.1 schema 扩展(单一事实来源)

详见 [25-tool-operations-sop.md](./25-tool-operations-sop.md) 和 `schemas/tool.schema.json`。

扩展字段:

| 字段 | 用途 | 必填 |
|------|------|------|
| `implementation` | 实现方式(frontend/static/db/proxy) | 是 |
| `dependencies` | npm 依赖列表 | 否 |
| `difficulty` | 难度(low/medium/high) | 是 |
| `priority` | 优先级(1-3 ★) | 是 |
| `action_label` | 按钮文案 | 否 |
| `input_schema` | 输入字段定义 | 否 |
| `related_tools` | 指定互链(覆盖自动) | 否 |
| `source` | 来源(original/fork) | 是 |
| `subdomain` | 子域名(高频工具) | 否 |

### 5.2 CI 校验(强制一致性)

现有 `scripts/check_consistency.py` 扩展校验:

```python
# 新增校验项
def check_tool_pages():
    """校验 implementedTools 与实际页面文件一致"""
    registry = load_registry()
    for tool in registry:
        if tool.get('implementation') == 'frontend' and not tool.get('deprecated'):
            page = f"frontend/src/tools/{tool['id']}/index.vue"
            if not exists(page):
                # 检查是否在 implementedTools Set 中
                ...
```

CI 失败场景:
- registry 有工具但页面文件不存在 → 失败
- 页面文件存在但 registry 没声明 → 失败
- schema 字段不合规 → 失败

### 5.3 工具生命周期管理

```
新增 → registry 加定义 → 创建页面 → gen:tools → CI 校验 → 上线
修订 → 改 registry(元数据/SEO) → 改页面(逻辑) → CI 校验 → 上线
下线 → registry 标 deprecated:true → 保留路由(外链兼容) → 30 天后删页面
```

### 5.4 文档与代码同步

每次工具变更,必须更新:

| 变更类型 | 更新文档 |
|---------|---------|
| 新增工具 | `01-progress.md` 记录 + registry |
| 修订工具 | `01-progress.md` 记录 + registry |
| 下线工具 | `01-progress.md` 记录 + registry 标 deprecated |
| 批量上线 | `01-progress.md` 记录批次 + 22 号文档进度 |

---

## 六、分批合入计划(对齐 22 号文档第六章)

### 6.1 第 1 批(Week 1-2):30 个纯前端工具

**选品**(22 号文档 6.1):
- 电脑网络纯前端 15 个(#21-35 前 15)
- 实用计算 8 个(#41-50)
- 文科工具纯前端 7 个(#56-62)

**合入工作量**:
- registry 加 30 条定义
- 创建 30 个 Vue 页面(用模板,每个 30 分钟)
- 总计约 15 工时

**验收门禁**:
- [ ] 30 工具可访问
- [ ] CI 校验通过
- [ ] 互链机制生效(每页 70 链接)
- [ ] GEO 元数据自动渲染

### 6.2 第 2 批(Week 3-4):30 个纯前端 + 静态数据

**选品**(22 号文档 6.2):
- 日常生活纯前端 8 个(#11-20)
- 网站建设 15 个(#81-95)
- 休闲测算纯前端 7 个(#111-120 前 7)

**新增工作**:
- 数据文件放 `public/data/`
- 大文件懒加载配置

### 6.3 第 3 批(Week 5-6):30 个数据库工具

**选品**(22 号文档 6.3):
- 日常生活数据库 8 个(#1-10)
- 文科工具数据库 6 个(#63-68)
- 休闲娱乐数据库 8 个(#96-103)
- 健康医疗 4 个(#71-74)
- 休闲测算数据库 4 个(#114-120)

**新增工作**:
- PostgreSQL 导入数据(phone/admin-divisions/idioms/poetry)
- FastAPI 加 `/api/query/{tool_id}` 通用查询端点
- 前端 fetch API

### 6.4 第 4 批(Week 7-8):30 个后端 API + 补齐

**选品**(22 号文档 6.4):
- 电脑网络后端 4 个(#37-40)
- 休闲娱乐后端 4 个(#104-110)
- 其他补齐 22 个

**新增工作**:
- FastAPI 代理层(DNS/IP 查询)
- 剩余工具补齐

### 6.5 付费工具(Week 9-10)

**选品**(22 号文档 6.5):
- PDF 转 Word(已有)
- 证件照制作
- 图片批量压缩
- 简历模板生成

**新增工作**:
- 证件照/压缩/简历 工具页 + backend
- 卡密系统已就绪,直接复用

---

## 七、与现有部署的对接

### 7.1 Docker Compose 不变

现有 `docker-compose.yml` 的 9 个服务(nginx/api/postgres/redis/minio/stirling/prometheus/grafana/loki/promtail)**完全保留**,120 工具合入不增减服务。

### 7.2 Nginx 配置微调

新增子域名通配(已有,确认):
```nginx
server {
  listen 443 ssl http2;
  server_name *.zhiniao.tools;  # ← 已有,支持子域名
  ...
}
```

### 7.3 前端构建

```bash
cd frontend
npm install  # 装新依赖(crypto-js/lunar-javascript 等)
npm run gen:tools   # 重新生成 registry.ts
npm run build:tokens
npm run build
```

### 7.4 后端不变

纯前端工具不碰后端。数据库类工具走现有 FastAPI,加通用查询端点:

```python
# backend/app/api/query.py(新增)
@router.get("/api/query/{tool_id}")
async def query_data(tool_id: str, q: str = ""):
    """通用数据查询(数据库类工具)"""
    tool = get_tool(tool_id)
    if tool.get('implementation') != 'db':
        raise_error("E404")
    # 路由到对应数据表
    return await query_from_db(tool_id, q)
```

---

## 八、风险与缓解

| 风险 | 缓解 |
|------|------|
| bundle 体积爆炸 | manualChunks 分割 + 数据文件懒加载 |
| 工具页风格不一 | 用 _template.vue 模板 + design-tokens |
| 互链失效 | CI 校验 related_tools 引用存在 |
| 数据文件过大 | public/data/ 懒加载,不进 bundle |
| 数据库类工具性能 | PostgreSQL 加索引 + Redis 缓存 |
| 工具数量超 120 | 22 号文档 8.3 反模式,严格控量 |

---

## 九、立即可执行的下一步

1. **扩展 schema**(`schemas/tool.schema.json` 加 platform_pivot 字段)— 见任务 #49
2. **合入 3 个示例工具**(密码生成/UUID/Base64)— 见任务 #48,验证方案
3. **编写操作 SOP**(`25-tool-operations-sop.md`)— 见任务 #50
4. **按第六章分批合入** — 对齐 22 号文档 12 周计划

---

## 十、相关文档

- [22-platform-pivot-strategy.md](./22-platform-pivot-strategy.md) — 平台转型战略
- [23-quick-start-operations.md](./23-quick-start-operations.md) — 每日操作清单
- [06-tool-registry.md](./06-tool-registry.md) — 工具注册表
- [07-consistency-link.md](./07-consistency-link.md) — 一致性链接(核心)
- [25-tool-operations-sop.md](./25-tool-operations-sop.md) — 工具操作 SOP
