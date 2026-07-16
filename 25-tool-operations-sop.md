# 工具操作 SOP

> **版本**: v1.0
> **最后更新**: 2026-07-03
> **状态**: 标准操作流程,AI 易读
> **依赖**: [24-tool-integration-plan.md](./24-tool-integration-plan.md), [07-consistency-link.md](./07-consistency-link.md)
> **目的**: 新增/修订/下线工具的标准步骤,任何人或 AI 按此操作

---

## 一、操作总览

```
新增工具 → 改 registry → 创建页面 → 加 implementedTools → gen:tools → CI 校验 → 上线
修订工具 → 改 registry(元数据) → 改页面(逻辑) → CI 校验 → 上线
下线工具 → 标 deprecated → 保留路由 30 天 → 删页面
```

**核心原则**: 所有操作从 `schemas/tools.registry.json` 开始,不直接改代码。

---

## 二、新增工具 SOP

### 2.1 前置检查

- [ ] 工具 ID 不与现有重复(`grep '"id":' schemas/tools.registry.json`)
- [ ] 工具 ID 全小写+连字符(如 `random-password`)
- [ ] 选定 category(8 大类之一:life/network/calc/text/health/web/fun/divination)
- [ ] 确认 implementation 类型(frontend/static/db/proxy/worker)

### 2.2 步骤(纯前端工具,典型流程)

**Step 1: 编辑 registry**

在 `schemas/tools.registry.json` 末尾追加:

```json
{
  "id": "random-password",
  "name": "随机密码生成",
  "description": "生成强随机密码,支持自定义长度和字符集",
  "category": "network",
  "icon": "🔑",
  "icon_color": "calc",
  "is_premium": false,
  "backend": "frontend",
  "route": "/random-password",
  "subdomain": "mima",
  "keywords": ["随机密码", "密码生成器", "强密码"],
  "implementation": "frontend",
  "dependencies": ["crypto-js"],
  "difficulty": "low",
  "priority": 3,
  "action_label": "生成密码",
  "input_schema": [
    {
      "name": "length",
      "type": "number",
      "label": "密码长度",
      "default": 16,
      "min": 4,
      "max": 64
    },
    {
      "name": "charset",
      "type": "checkbox-group",
      "label": "字符集",
      "options": ["小写 a-z", "大写 A-Z", "数字 0-9", "符号 !@#$"],
      "default": ["小写 a-z", "大写 A-Z", "数字 0-9"]
    },
    {
      "name": "exclude",
      "type": "text",
      "label": "排除字符",
      "placeholder": "如 iIl1o0O"
    }
  ],
  "related_tools": ["password-strength", "uuid-generator", "hash-calculator"],
  "source": { "type": "original" },
  "about": {
    "title": "关于随机密码生成器",
    "content": "随机密码生成器用于创建高强度密码,防止账号被盗。本工具在浏览器本地生成,不上传任何数据,隐私安全。建议密码长度 ≥ 16 位,包含大小写字母、数字和符号。",
    "features": ["本地生成,不上传", "自定义长度 4-64 位", "可选字符集", "破解耗时估算"],
    "scenarios": ["账号注册", "数据库密码", "API Token", "WiFi 密码"],
    "faq": [
      { "q": "密码安全吗?", "a": "安全。使用浏览器 crypto.getRandomValues,本地生成,不经过网络。" },
      { "q": "多长合适?", "a": "建议 16 位以上,含大小写+数字+符号,破解耗时 > 100 年。" }
    ]
  },
  "seo": {
    "json_ld": {
      "@type": "SoftwareApplication",
      "name": "随机密码生成器",
      "applicationCategory": "UtilityApplication",
      "operatingSystem": "Web",
      "offers": { "@type": "Offer", "price": "0", "priceCurrency": "CNY" }
    },
    "description_for_ai": "免费在线随机密码生成器,浏览器本地生成,不上传数据。支持自定义长度 4-64 位、字符集选择、排除易混字符。生成强密码,防止账号被盗。"
  }
}
```

**Step 2: 校验 schema**

```bash
cd /c/Users/Administrator/tool-site-project
PYTHONIOENCODING=utf-8 python scripts/check_consistency.py
# 应输出: ✓ N 个工具全部通过 schema 校验
```

**Step 3: 创建页面文件**

```
frontend/src/tools/random-password/index.vue
```

用模板(见 24 号文档 4.1),实现核心逻辑:

```vue
<template>
  <ToolLayout :tool="tool">
    <div class="tool-main">
      <!-- 输入区 -->
      <div class="option-group" v-for="field in tool.input_schema" :key="field.name">
        <label class="option-label">{{ field.label }}</label>
        <input v-if="field.type === 'number'" type="number" v-model.number="input[field.name]" :min="field.min" :max="field.max" />
        <div v-else-if="field.type === 'checkbox-group'">
          <label v-for="opt in field.options" :key="opt">
            <input type="checkbox" :value="opt" v-model="input[field.name]" /> {{ opt }}
          </label>
        </div>
        <input v-else v-model="input[field.name]" :placeholder="field.placeholder" />
      </div>

      <button class="convert-btn" @click="generate">{{ tool.action_label }}</button>

      <div v-if="result" class="result-card">
        <code>{{ result }}</code>
        <button @click="copy">复制</button>
      </div>
    </div>
  </ToolLayout>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import ToolLayout from '@/layouts/ToolLayout.vue'
import { getTool } from '@/tools/registry'
import CryptoJS from 'crypto-js'

const tool = getTool('random-password')!
const input = reactive({ length: 16, charset: ['小写 a-z', '大写 A-Z', '数字 0-9'], exclude: '' })
const result = ref('')

function generate() {
  const charsets: Record<string, string> = {
    '小写 a-z': 'abcdefghijklmnopqrstuvwxyz',
    '大写 A-Z': 'ABCDEFGHIJKLMNOPQRSTUVWXYZ',
    '数字 0-9': '0123456789',
    '符号 !@#$': '!@#$%^&*',
  }
  let chars = input.charset.map(c => charsets[c] || '').join('')
  if (input.exclude) {
    chars = chars.split('').filter(c => !input.exclude.includes(c)).join('')
  }
  // 用 crypto-js 的随机数(浏览器环境等同 crypto.getRandomValues)
  let pwd = ''
  const arr = new Uint32Array(input.length)
  crypto.getRandomValues(arr)
  for (let i = 0; i < input.length; i++) {
    pwd += chars[arr[i] % chars.length]
  }
  result.value = pwd
}

function copy() {
  navigator.clipboard.writeText(result.value)
}
</script>
```

**Step 4: 加入 implementedTools**

编辑 `frontend/src/router/index.ts`:

```typescript
const implementedTools = new Set([
  'pdf-to-word',
  'random-password',  // ← 新增
])
```

**Step 5: 重新生成路由**

```bash
cd frontend
npm run gen:tools
# 输出: ✓ registry.ts generated (N tools)
```

**Step 6: 本地验证**

```bash
npm run dev
# 浏览器打开 http://localhost:5173/random-password
# 测试功能正常
```

**Step 7: CI 校验 + 提交**

```bash
cd /c/Users/Administrator/tool-site-project
python scripts/check_consistency.py
# 全部通过后提交
```

**Step 8: 更新进度**

在 `01-progress.md` 记录新增工具。

### 2.3 数据库类工具(db)的额外步骤

除 2.2 的 Step 1-8 外,额外:

**Step A: 导入数据到 PostgreSQL**

```bash
# 假设 idioms.json
docker compose exec -T postgres psql -U zhiniao -d zhiniao -c "
CREATE TABLE IF NOT EXISTS tool_idioms (
  id SERIAL PRIMARY KEY,
  word VARCHAR(64),
  meaning TEXT,
  pinyin VARCHAR(128)
);"

# 用 Python 脚本导入
python scripts/import_data.py --table tool_idioms --file data/idioms.json
```

**Step B: registry 标注**

```json
{
  "implementation": "db",
  "backend": "db-query",
  "data_file": "idioms.json",
  "input_schema": [
    { "name": "keyword", "type": "text", "label": "搜索词", "placeholder": "输入成语" }
  ]
}
```

**Step C: 前端调用 API**

```typescript
// 数据库类工具统一调用 /api/query/{tool_id}
const res = await fetch(`/api/query/idiom-dict?q=${encodeURIComponent(keyword)}`).then(r => r.json())
// res = { tool_id, query, count, results: [...] }
```

**Step D: 后端注册查询(关键)**

数据库工具的核心是后端 `backend/app/api/query.py` 的 `TOOL_QUERIES` 映射。新增数据库工具必须在此注册:

```python
# backend/app/api/query.py
TOOL_QUERIES = {
    "your-tool-id": {
        "table": "tool_your_table",
        "input_field": "keyword",
        "input_label": "查询词",
        "input_max": 64,
        "columns": "col1, col2, col3",        # SELECT 的列
        "match": "col1 LIKE :q",               # WHERE 条件
        "like": True,                           # 是否模糊匹配
    },
}
```

**前端用 QueryResult 组件展示**(避免重复代码):
```vue
<QueryResult :results="results" :loading="loading" :error="error">
  <template #item="{ item }">
    <!-- 自定义每条结果的渲染 -->
    <div>{{ item.word }}</div>
  </template>
</QueryResult>
```

**已验证的数据库工具**(v0.7.0):
- id-card-region(身份证归属,tool_id_cards)
- phone-region(手机归属,tool_phone_numbers)
- idiom-dict(成语,tool_idioms)
- poetry(诗词,tool_poetry)
- history-today(历史今天,tool_history_today)

参考这 5 个的实现作为模板。

### 2.4 后端代理类工具(proxy)

如 DNS 查询,需 FastAPI 代理:

```python
# backend/app/api/proxy.py
@router.get("/api/proxy/dns")
async def dns_query(domain: str, type: str = "A"):
    # 代理 Cloudflare DoH
    async with httpx.AsyncClient() as client:
        r = await client.get(f"https://cloudflare-dns.com/dns-query?name={domain}&type={type}",
                             headers={"accept": "application/dns-json"})
        return r.json()
```

registry 标注:
```json
{ "implementation": "proxy", "backend": "proxy" }
```

---

## 三、修订工具 SOP

### 3.1 改元数据(名称/描述/SEO/互链)

只改 registry,无需碰代码:

```bash
# 1. 编辑 schemas/tools.registry.json,改对应工具字段
# 2. 校验
python scripts/check_consistency.py
# 3. 重新生成(如改了 route/id 才需要)
cd frontend && npm run gen:tools
# 4. 提交
```

### 3.2 改业务逻辑

```bash
# 1. 改 frontend/src/tools/{id}/index.vue
# 2. 本地测试
cd frontend && npm run dev
# 3. 提交
```

### 3.3 改视觉

```bash
# 1. 改 schemas/design-tokens.json(全局视觉)
# 2. 重新生成
cd frontend && npm run build:tokens
# 或改单页样式(在 index.vue 的 <style scoped>)
# 3. 提交
```

---

## 四、下线工具 SOP

### 4.1 软下线(保留路由,不渲染卡片)

```json
// registry 中标 deprecated
{ "id": "old-tool", "deprecated": true }
```

效果:
- 首页不显示卡片
- 路由仍可访问(外链兼容)
- 互链不引用

### 4.2 硬下线(30 天后清理)

```bash
# 1. 确认软下线已 30 天
# 2. 删除 registry 条目
# 3. 删除页面文件
rm -rf frontend/src/tools/{id}/
# 4. 从 implementedTools 移除
# 5. gen:tools 重新生成
# 6. 提交
```

---

## 五、批量合入 SOP(对齐 22 号文档分批计划)

### 5.1 批量新增 30 个工具

```bash
# 1. 准备 30 个工具定义(JSON 片段)
# 2. 追加到 registry
# 3. 批量创建页面(可用脚本生成骨架)
for tool_id in random-password uuid-generator base64-encode ...; do
  mkdir -p frontend/src/tools/$tool_id
  cp frontend/src/tools/_template.vue frontend/src/tools/$tool_id/index.vue
done
# 4. 逐个实现逻辑(人工,每个 30 分钟)
# 5. 全部加入 implementedTools
# 6. gen:tools
# 7. build 验证
cd frontend && npm run build
```

### 5.2 批量校验

```bash
# 1. schema 校验
python scripts/check_consistency.py
# 2. 前端构建
cd frontend && npm run build
# 3. 页面完整性(所有 implementedTools 都有页面)
node scripts/validate.mjs
```

---

## 六、AI 操作规范

AI 接到"加 XX 工具"时:

1. 读 `schemas/tool.schema.json` 了解字段
2. 读 `schemas/tools.registry.json` 确认 ID 不重复
3. 按 2.2 步骤执行
4. 用模板创建页面
5. 更新 `01-progress.md`
6. **禁止跳过 schema 校验直接提交**

详见 [08-ai-collaboration.md](./08-ai-collaboration.md)。

---

## 七、质量检查清单

每次工具变更提交前:

- [ ] schema 校验通过(`check_consistency.py`)
- [ ] 前端构建通过(`npm run build`)
- [ ] implementedTools 与页面文件一致(`validate.mjs`)
- [ ] 互链 related_tools 引用的 ID 都存在
- [ ] SEO 元数据(seo.json_ld)有效
- [ ] about.content ≥ 50 字(SEO)
- [ ] 更新了 01-progress.md

---

## 七点五、批量更新 registry

批量新增/扩展工具时,用手工编辑 JSON 易出错。用 `scripts/update_registry.py` 脚本:

```bash
python scripts/update_registry.py
```

脚本支持:
- **扩展已有工具**:补全 implementation/input_schema/related_tools 等新字段
- **新增工具**:批量追加(每个工具定义完整)
- **标记 deprecated**:避免重复(timestamp 被 timestamp-convert 替代)

**脚本结构**(参考实现):
```python
# 1. 加载 registry
reg = json.loads(...)
# 2. updates 字典:对已有工具补字段
updates = {"uuid-gen": {"implementation": "frontend", ...}}
# 3. new_tools 列表:新工具完整定义
new_tools = [{...}, {...}]
# 4. 应用更新 + 追加 + 保存
```

新增工具时复制脚本改 `new_tools` 内容即可,避免手写 JSON 语法错误。

---

## 七点六、bundle 优化经验

vite.config.ts 的 manualChunks 用**函数式**分割(非对象式),避免引用未安装依赖导致构建失败:

```typescript
manualChunks(id) {
  if (id.includes('node_modules')) {
    // 已安装的大依赖各自独立 chunk
    if (id.includes('zxcvbn')) return 'vendor-zxcvbn'      // 密码强度
    if (id.includes('qrcode')) return 'vendor-qrcode'      // 二维码
    if (id.includes('crypto-js')) return 'vendor-crypto'   // 加密
    if (id.includes('js-base64')) return 'vendor-base64'   // Base64
    if (id.includes('element-plus')) return 'vendor-element' // UI
    if (id.includes('vue')) return 'vendor-vue'            // 核心
    return 'vendor'  // 其他小依赖
  }
}
```

**原则**:
1. 只分割**已安装**的依赖(未装的动态 import 自动分割)
2. 体积大的(zxcvbn 818KB)必须独立,避免拖累主 bundle
3. `chunkSizeWarningLimit` 按最大合理 chunk 设置(zxcvbn 字典 818KB,设 900)

**新增大依赖时**:在 manualChunks 加对应分支,保持按需加载。

---

## 八、常见问题

### Q1: 工具 ID 用中文还是拼音?

**A**: ID 用拼音+连字符(`random-password`),name 用中文(`随机密码生成`)。子域名可用拼音(`mima`)。

### Q2: 纯前端工具的依赖怎么管?

**A**: 加到 registry 的 `dependencies` 字段,前端 `npm install` 后在页面 `import`。大依赖用动态 import 优化 bundle。

### Q3: 数据文件放哪?

**A**: `frontend/public/data/{file}.json`,前端 `fetch('/data/{file}.json')`。大文件(>5MB)确保懒加载。

### Q4: 工具页要不要登录?

**A**: 免费工具不要登录墙(bmcx 模式核心)。付费工具走现有 JWT + 余额检查。

### Q5: 改了 registry 但前端没生效?

**A**: 执行 `cd frontend && npm run gen:tools` 重新生成 `registry.ts`,然后 `npm run dev` 或 `npm run build`。

---

## 九、相关文档

- [24-tool-integration-plan.md](./24-tool-integration-plan.md) — 合入方案总纲
- [22-platform-pivot-strategy.md](./22-platform-pivot-strategy.md) — 平台转型战略
- [06-tool-registry.md](./06-tool-registry.md) — 工具注册表
- [07-consistency-link.md](./07-consistency-link.md) — 一致性链接
- [08-ai-collaboration.md](./08-ai-collaboration.md) — AI 协作规范
