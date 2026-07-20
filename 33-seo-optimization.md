# SEO 优化方案 (SEO Optimization Plan)

> **版本**: v1.1
> **最后更新**: 2026-07-09
> **状态**: T0/T1/T2 已落地,T3 待域名确定后执行
> **适用对象**: 前端 / 架构 / 运营
> **前置阅读**: [22-platform-pivot-strategy.md](./22-platform-pivot-strategy.md)(含 GEO 附加项) · [01-progress.md](./01-progress.md)(当前进度) · [07-consistency-link.md](./07-consistency-link.md)(SSOT 机制)

---

## 一、方案总览

本站是 **bmcx 模式便民工具平台**,SEO 是核心流量来源(子域名/长尾词矩阵)。
当前实现存在 **一个致命缺口 + 若干内容缺口**:技术栈已引入 `vite-ssg` 但未接入,站点仍是纯 CSR 的 SPA,爬虫抓到的是空壳页面;同时缺少每页 title/meta/sitemap,且 110 个工具中仅 3 个填充了 `seo` 字段。

本方案目标:**让每个工具页被搜索引擎和 AI 引擎完整、正确、快速地索引,并长期稳定获流。**

### 1.1 优先级矩阵

| 优先级 | 主题 | 工作量 | 影响 | 对齐阶段 |
|--------|------|--------|------|---------|
| 🔴 T0 | 技术SEO:SSG 接入 + 每页 head + sitemap | 中(1-2 天) | 决定能否被索引 | P1 基础设施 |
| 🔴 T0 | 占位薄页处理(noindex) | 小(2 小时) | 避免被降权 | P1 |
| 🟠 T1 | `seo` 字段全量补全(生成器) | 中(脚本+校对) | 决定排名/CTR | P3 工具矩阵 |
| 🟠 T1 | JSON-LD 自动生成(全部工具) | 小(半天) | 富摘要/GEO | P3 |
| 🟠 T1 | 路由策略统一(子目录) + canonical | 小 | 去重 | P1 |
| 🟡 T2 | llms.txt 重写 + AI 友好内容 | 小 | GEO | P3 |
| 🟡 T2 | 内容补全(FAQ/how_to 扩量) | 大(持续) | 排名/停留 | P3-P5 |
| 🟢 T3 | 外部:域名/备案/收录提交/统计 | 小(等外部) | 收录启动 | 全程 |

---

## 二、现状诊断(Audit)

### 2.1 ✅ 已具备的良好基础

| 项 | 现状 | 评价 |
|----|------|------|
| SSG 依赖 | `vite-ssg ^0.23.0` 已在 devDependencies | 依赖就位,但**未接入** |
| SSOT 生成管线 | `scripts/gen-tools.mjs`(registry→registry.ts) | 可挂载 sitemap/head 生成 |
| 工具页内容骨架 | `ToolLayout.vue`:面包屑 / `<h1>` / about(特点·场景) / FAQ / 70 互链 | 结构优秀 |
| JSON-LD 容器 | ToolLayout 已有 `<script type="application/ld+json">` 渲染位 | 仅对有 `seo.json_ld` 的工具生效 |
| 互链机制 | 同类 35 + 随机 35 = 70 链/页(bmcx 模式) | 权重传递 + 蜘蛛遍历良好 |
| AI 爬虫放行 | `robots.txt` 放行 GPTBot/ClaudeBot/PerplexityBot/CCBot/Google-Extended | GEO 基础 |
| llms.txt | 根目录已存在 | **内容已过时**(见 2.3) |
| 工具内容 | 88/110 工具有 `about`(intro/features/scenarios),24 个有 FAQ | 内容基础好,需扩量 |
| 移动适配 | 响应式已做(见 32 号文档) | 符合移动优先索引 |

### 2.2 ❌ 致命缺口

| # | 缺口 | 现状 | 后果 |
|---|------|------|------|
| G1 | **未接入 SSG** | `vite.config.ts` 无 vite-ssg 插件,`main.ts` 用 `app.mount('#app')`(纯 CSR) | 爬虫抓到空 `<div id="app">`;**百度几乎不渲染 JS**,Google 渲染有延迟且权重低 |
| G2 | **无每页 head** | 无 head 管理库(`@unhead/vue`/`@vueuse/head` 均未安装);全部页面共用 `index.html` 的 `<title>知鸟工具箱 - 50+ 免费在线工具` | 每页 title/description/canonical/OG 全缺 → 无法针对关键词排名 |
| G3 | **无 sitemap.xml** | `robots.txt` 写了 `Sitemap: https://zhiniao.tools/sitemap.xml`,但文件不存在、域名未注册、无生成脚本 | 爬虫发现/收录慢,收录量无法监控 |
| G4 | **`seo` 字段几乎空** | 110 工具中仅 3 个(`pdf-to-word`/等)有 `seo`;JSON-LD 仅这 3 个渲染 | 107 个工具页无独立 title/description/结构化数据 |
| G5 | **占位薄页** | 16 个工具(`pdf-merge`/`pdf-split`/`pdf-compress`/`pdf-ocr`/`caj-to-pdf` 等)路由指向 `_placeholder/index.vue` | 软 404 / 薄内容 → 站点整体被降权风险 |

### 2.3 ⚠️ 内容/一致性问题

| # | 问题 | 证据 |
|---|------|------|
| C1 | llms.txt 路径错误 | 文中用 `/tools/pdf-to-word`,实际路由是 `/pdf-to-word`(无 `/tools/` 前缀) |
| C2 | llms.txt 列了不存在的工具 | `pdf-merge`/`pdf-split`/`pdf-compress`/`pdf-ocr`/`caj-to-pdf` 等是占位页,并非已实现 |
| C3 | 子域名/子目录混用 | registry 21 个工具有 `subdomain` 字段,但路由全部是子目录 `/${id}` | 策略文档(22 号)说免费工具走子域名,实现走子目录 → 若两种都解析会产生重复内容 |
| C4 | `tool.schema.json` 未覆盖 `seo` 完整结构 | `seo` 实际含 `json_ld`/`faq`/`how_to`/`description_for_ai`,但 schema 未约束 → 无法被 `npm run validate` 校验 |
| C5 | 无 canonical | 子目录/子域名混用下无 `<link rel="canonical">` → 重复内容风险 |
| C6 | 无站长验证/统计 | 无百度站长/Google Search Console 验证 meta,无百度统计/GA4 | 收录与流量无法监控与推进 |

---

## 三、技术 SEO(T0 — 决定能否被索引)

### 3.1 G1:接入 vite-ssg,把 SPA 改为预渲染静态站

> 这是全站 SEO 的前提。未做此项,以下所有内容优化对百度基本无效。

**改造步骤**:

1. 新增 `frontend/src/entry.ts`(SSG 入口),`main.ts` 改为 CSR-only 兜底:
   ```ts
   // entry.ts —— vite-ssg 入口
   import { ViteSSG } from 'vite-ssg'
   import App from './App.vue'
   import router from './router'
   import { createPinia } from 'pinia'
   import ElementPlus from 'element-plus'
   import 'element-plus/dist/index.css'
   import './assets/styles/main.css'

   export const createApp = ViteSSG(App, { routes: router.options.routes }, ({ app }) => {
     app.use(createPinia())
     app.use(ElementPlus)
   })
   ```
2. `vite.config.ts` 接入 vite-ssg 插件,并配置 **included routes**(从 registry 生成,避免爬虫式发现):
   ```ts
   import { ViteSSG } from 'vite-ssg' // 或在 entry 中处理
   // ssg: { included: routesFromRegistry, formatting: 'minify' }
   ```
3. `index.html` 的 `<script src="/src/main.ts">` 改为 `entry.ts`。
4. 注意:vite-ssg 在构建期执行 `Math.random()` 互链排序会被**固化**(可接受,反而利于 SEO 稳定);动态 `import()` 工具页由 vite-ssg 处理。

**验收**:`dist/${tool.route}/index.html` 每个工具页生成独立静态 HTML,`view-source` 可见 `<h1>`/about/互链内容(不再是空 `<div id="app">`)。

### 3.2 G2:接入 head 管理,每页输出独立 title/meta/canonical/OG

1. 安装 `@unhead/vue`(vite-ssg 原生集成 unhead):
   ```bash
   npm i @unhead/vue
   ```
2. 在 `ToolLayout.vue` 与 `DefaultLayout.vue` 用 `useHead()` 注入,字段来源见 §四(seo 字段生成):
   ```ts
   useHead({
     title: tool.seo?.title ?? `${tool.name} - ${tool.keywords?.[0] ?? '在线工具'} | 知鸟工具箱`,
     meta: [
       { name: 'description', content: tool.seo?.description ?? tool.description },
       { name: 'keywords', content: (tool.keywords ?? []).join(',') },
       { property: 'og:title', content: tool.name },
       { property: 'og:description', content: tool.seo?.description ?? tool.description },
       { property: 'og:type', content: 'website' },
       { name: 'robots', content: tool.implemented ? 'index,follow' : 'noindex,follow' }, // §3.4
     ],
     link: [{ rel: 'canonical', href: `https://${SITE_DOMAIN}${tool.route}` }],
   })
   ```
3. `Home.vue` 单独配置首页 title/description(品牌词 + 核心关键词)。

**title 公式**:`${name} - ${主关键词} | 知鸟工具箱`(≤30 字,关键词前置)。
**description**:≤120 字,含 1-2 个长尾词 + 价值点,避免堆砌。

### 3.3 G3:自动生成 sitemap.xml

1. 新增 `frontend/scripts/gen-sitemap.mjs`(或并入 `gen-tools.mjs`),读 registry 输出 `public/sitemap.xml`:
   - 仅收录 **非 deprecated + 已实现**(implementedTools)的工具页 + 静态页(首页/redeem 除外)
   - 字段:`<loc>`、`<lastmod>`(取构建日期,见 §七无 Date.now 处理)、`<changefreq>`(daily/weekly)、`<priority>`(付费/高搜索量 0.8,其余 0.6)
   - 同步生成 `public/sitemap-images.xml`(可选,工具截图)与 `public/rss.xml`(可选)
2. `package.json` 加脚本:`"gen:sitemap": "node scripts/gen-sitemap.mjs"`,并入 `build` 流程。
3. `robots.txt` 的 `Sitemap:` 行改为真实域名(域名到位后)。
4. 收录提交时一并提交 sitemap 到百度站长 + Google Search Console + Bing。

**验收**:`public/sitemap.xml` 含全部已实现工具 URL;GSC/Bing 报告 sitemap 状态为"已发现"。

### 3.4 G5:占位薄页处理(防降权)

16 个占位工具(`pdf-merge`/`pdf-split`/`pdf-compress`/`pdf-ocr`/`pdf-decrypt`/`pdf-to-image`/`pdf-watermark`/`ai-matting`/`image-watermark`/`photo-restore`/`caj-to-pdf`/`excel-to-pdf`/`ppt-to-pdf`/`ebook-convert`/`audio-convert`/`batch-translate`):

| 处置 | 做法 |
|------|------|
| ✅ 推荐(短期) | `_placeholder/index.vue` 输出 `<meta name="robots" content="noindex,follow">`;sitemap 排除;首页/互链仍可展示"即将上线"角标,但 a 标签加 `rel="nofollow"` |
| ✅ 推荐(中期) | 按 §22 计划实现后,移除 noindex、加入 sitemap、补 about/seo |

> 关键:占位页**不要**进 sitemap、**不要**被索引。否则 16 个薄页稀释整站质量评分。

### 3.5 C3/C5:路由策略统一 + canonical

**决策**:统一走**子目录** `https://${domain}/${tool.id}`(当前实现即如此)。
- 理由:新域名无权重,**子目录比子域名更利于集中权重**;bmcx 用子域名是因其运营 10 年+。子目录 1 个域名 1 份备案,运维最简。
- 处置:`registry` 的 `subdomain` 字段降级为"未来可选",当前全部以 `route` 为准;canonical 指向子目录 URL。
- 若后期个别工具流量极高,可单独拆子域名并做 301 到主站(高阶,非必需)。

**canonical**:每页 `<link rel="canonical" href="https://${domain}${route}">`,消除子域/子目录混用导致的重复内容(C5)。

### 3.6 nginx 适配 SSG

`nginx.conf` 的 SPA 回退改为支持预渲染 HTML:
```nginx
location / {
  try_files $uri $uri/ /index.html;  # SSG 后 $uri/ 命中 dist 下静态 HTML,无需改
}
```
SSG 产物为 `${route}/index.html`,`try_files $uri $uri/` 已能命中,无需额外改动。补充:
- 开启 **brotli**(可选,优于 gzip,提升 LCP 分);
- `robots.txt`/`llms.txt`/`sitemap.xml` 设 `Cache-Control: public, max-age=3600`;
- 静态资源 `immutable` 缓存(已做)。

---

## 四、内容 SEO(T1 — 决定排名与点击率)

### 4.1 G4:`seo` 字段全量补全(生成器,SSOT 对齐)

**原则**:不手写 107 个,从已有字段**派生 + 校对**。

新增 `scripts/gen_seo_fields.py`:遍历 registry,对缺 `seo` 的工具自动生成:
```python
seo = {
  "title": f"{name} - {keywords[0]} | 知鸟工具箱",          # ≤30 字
  "description": about.content[:118] if about else description[:118],  # ≤120 字
  "keywords": keywords[:5],
}
```
- 生成后由人工/运营校对 title/description(避免公式化同质化),高搜索量工具(个税/房贷/万年历等)重点打磨。
- 写回 `schemas/tools.registry.json`(SSOT),跑 `npm run gen:tools` 同步前端。

### 4.2 JSON-LD 自动生成(全部工具,不依赖手填)

在 ToolLayout 用计算属性**为每个工具自动生成结构化数据**(当前仅 3 个手填):
```ts
const jsonLd = computed(() => [JSON.stringify({
  '@context':'https://schema.org',
  '@type':'WebApplication',
  'name': tool.name,
  'applicationCategory':'UtilityApplication',
  'operatingSystem':'Web',
  'url': `https://${domain}${tool.route}`,
  'offers': {'@type':'Offer','price': tool.is_premium ? '2' : '0','priceCurrency':'CNY'},
  'description': tool.seo?.description ?? tool.description,
})])
// 若有 faq → 追加 FAQPage;若 about.scenarios → 追加 HowTo;breadcrumb → BreadcrumbList
```
- 改 ToolLayout:把 `<script v-if="tool.seo?.json_ld">` 改为 `<script>` 恒输出 `jsonLd`(数组),手填的 `seo.json_ld` 作为覆盖项。
- 效果:富摘要(SoftwareApplication)、FAQ 富结果、面包屑富结果 → 提升 SERP CTR。

### 4.3 C4:扩展 `tool.schema.json`

补充 `seo` 对象 schema,使 `npm run validate` 能校验:
```json
"seo": {
  "type":"object",
  "properties":{
    "title":{"type":"string","maxLength":40},
    "description":{"type":"string","maxLength":130},
    "keywords":{"type":"array","items":{"type":"string"}},
    "faq":{"type":"array","items":{"type":"object","properties":{"q":{"type":"string"},"a":{"type":"string"}}}},
    "how_to":{"type":"string"},
    "description_for_ai":{"type":"string"},
    "json_ld":{"type":"object"}
  }
}
```

### 4.4 内容扩量(持续)

| 字段 | 现状 | 目标 | 方法 |
|------|------|------|------|
| `about` | 88/110 | 100%(已实现工具) | 缺的补 intro/features/scenarios |
| `about.faq` | 24 | ≥60(高搜索量工具必加 3-5 条) | 每工具 3-5 个真实问答(People Also Ask) |
| `seo.how_to` | 极少 | 计算类/转换类工具都加 | "1.上传 2.选参数 3.点开始"步骤文本 |
| `seo.description_for_ai` | 极少 | 全部(高搜索量优先) | 一段客观、可被 AI 引用的功能说明(GEO) |

> FAQ 是性价比最高的内容投入:既喂 FAQPage 富结果,又喂 AI 引用(GEO),还增加长尾关键词密度。

---

## 五、GEO 附加项(T2 — 被 AI 引擎引用)

> 对齐 22 号文档第九章(免费、不专项投入,随内容一起做)。

### 5.1 C1/C2:重写 llms.txt

修正路径(`/tools/x` → `/x`)、删除未实现工具、按分类组织、每工具配 `description_for_ai`:
```
# 知鸟工具箱

知鸟工具箱,100+ 免费在线便民工具集合,涵盖日常查询、电脑网络、计算、文本、文化、健康等。文件类工具浏览器本地处理,隐私安全。

## 日常查询
- 万年历查询: /lunar-calendar — 公历农历对照、黄历宜忌、节气标注
- 老黄历: /old-almanac — 黄道吉日、宜忌冲煞值神
- 身份证归属查询: /id-card-region — 身份证号前 6 位查省市
...

## 关于
本站工具大部分免费;高质量 PDF 转 Word 基于 WPS+OCR(2 元/次,卡密兑换)。
文件处理完成 1 小时自动删除,不存储用户文件。
联系: zhiniao_tools(微信)
```
- 仅列**已实现**工具;占位工具不写,避免 AI 引用无效页。

### 5.2 AI 友好实践清单

| 实践 | 落点 | 状态 |
|------|------|------|
| llms.txt | 根目录 | 重写(§5.1) |
| Schema.org JSON-LD | 每工具页 | 自动生成(§4.2) |
| FAQ 区块 | ToolLayout | 已有容器,扩量(§4.4) |
| robots 放行 AI 爬虫 | robots.txt | ✅ 已做 |
| 语义化 HTML + 清晰 URL | 全站 | route=`/${id}`,语义化标签已有 h1/h2/面包屑 |
| Markdown 友好内容 | about 文本 | description_for_ai(§4.4) |

---

## 六、外部 SEO(T3 — 收录启动,需外部资源)

> 对齐 01-progress.md "待用户准备外部资源"。这些我无法代劳,列出供排期。

| 项 | 动作 | 责任 |
|----|------|------|
| 域名 | 注册含 `tools`/`gongju`/`gongju` 关键词的 `.com`/`.cn`,开泛解析 | 用户 |
| 备案 | 国内服务器必须,20 工作日 | 用户 |
| SSL | Cloudflare 泛证书(免费) | 运维 |
| 站长验证 | 百度搜索资源平台 + Google Search Console + Bing Webmaster,加验证 meta | 运维 |
| sitemap 提交 | 三大平台均提交 `sitemap.xml` | 运维 |
| 主动推送 | 百度普通收录 API 主动推送(发布/更新工具时调) | 后端 |
| 百度自动推送 | 结果页加百度自动推送 JS(可选,百度已弱化) | 前端 |
| 统计 | 百度统计 + GA4,接入 PV/UV/来源/热图 | 运维 |
| 外链 | 闲鱼/公众号/知乎/相关论坛引种子外链 | 运营 |

**验证 meta 接入方式**:在 `index.html` 或用 `useHead` 按环境注入 `baidu-site-verification`/`google-site-verification` meta(避免提交到 git,用 env)。

---

## 七、SSOT 集成(对齐项目规范)

本方案严格遵守"**所有变更从 schema 开始**"(见 CLAUDE.md):

```
改 tool.schema.json(加 seo 结构,§4.3)
  → 改 tools.registry.json(补 seo 字段,§4.1 生成器)
  → gen-tools.mjs(同步前端 + 生成 sitemap,§3.3)
  → ToolLayout(自动 JSON-LD + useHead,§3.2/§4.2)
  → vite-ssg 接入(§3.1)
  → npm run build → dist 预渲染 HTML
  → npm run validate 校验
  → 更新 01-progress.md
```

**新增/改动命令**:
| 命令 | 作用 |
|------|------|
| `npm run gen:sitemap`(新) | registry → public/sitemap.xml |
| `python scripts/gen_seo_fields.py`(新) | 派生缺失 seo 字段,写回 registry |
| `npm run build` | 接入 vite-ssg 后产出预渲染静态站 |
| `npm run validate` | 校验含 seo 的 schema |

---

## 八、落地路线图(对齐 22 号文档 12 周计划)

| 周次 | 阶段 | SEO 任务 | 验收 |
|------|------|---------|------|
| Week 1-2 | P1 基础设施 | **T0 全部**:vite-ssg 接入、head 管理、sitemap、占位页 noindex、canonical、robots 修 | `view-source` 见内容;sitemap 生成;占位页 noindex |
| Week 2 | 上线第 1 批 | 域名/备案启动、站长验证、sitemap 提交 | 三大平台 sitemap 已发现 |
| Week 3-4 | P3 工具矩阵 | T1:`seo` 字段生成器 + 校对、JSON-LD 自动生成、llms.txt 重写 | 全部已实现工具有独立 title/JSON-LD |
| Week 5-8 | 扩量 | T1/T2:FAQ/how_to 扩量、description_for_ai、内容补全 | 高搜索量工具 FAQ≥3 条 |
| Week 9-10 | 付费工具 | 付费工具页 SEO + 转化文案 | PDF 转_word 排名监控 |
| Week 11-12 | 优化监控 | T3:百度普通收录主动推送、统计接入、低流量工具优化/淘汰 | 收录量/UV 达标 |

**时间戳注意**:`gen-sitemap.mjs` 的 `<lastmod>` 不可在脚本内用 `Date.now()`/`new Date()`(项目约束);从构建环境注入或取 registry 的 `updated` 字段。`ToolLayout` 的 `Math.random()` 互链在 SSG 构建期固化,可接受。

---

## 九、验收指标(可勾选)

### 9.1 技术 SEO 验收(T0)
- [ ] `view-source:${route}` 每个工具页可见 `<h1>`/about/互链(非空壳)
- [ ] 每页 `<title>` 独立且 ≤30 字、关键词前置
- [ ] 每页 `<meta description>` 独立、≤120 字
- [ ] 每页 `<link rel="canonical">` 指向子目录 URL
- [ ] `public/sitemap.xml` 生成,仅含已实现工具
- [ ] 16 个占位页 `noindex`
- [ ] `robots.txt` Sitemap 指向真实域名
- [ ] GSC/Bing/百度 sitemap 提交成功

### 9.2 内容 SEO 验收(T1)
- [ ] 100% 已实现工具填充 `seo`(title/description/keywords)
- [ ] 100% 已实现工具自动输出 JSON-LD(WebApplication)
- [ ] 高搜索量工具(≥15 个)有 FAQ ≥3 条 + FAQPage JSON-LD
- [ ] `tool.schema.json` 覆盖 `seo` 结构,`npm run validate` 通过
- [ ] llms.txt 路径正确、仅列已实现工具

### 9.3 外部/监控验收(T3)
- [ ] 域名注册 + 备案完成
- [ ] 百度/Google/Bing 站长验证通过
- [ ] 百度统计 + GA4 接入,可看 PV/UV/来源
- [ ] 百度普通收录主动推送就绪

### 9.4 效果指标(对齐 22 号文档阶段门禁)
| 指标 | Week 2 | Week 4 | Week 6 | Week 12 |
|------|--------|--------|--------|---------|
| 百度收录页数 | ≥30 | ≥60 | ≥80 | ≥100 |
| 月 UV | ≥100 | ≥500 | ≥2000 | ≥10000 |
| sitemap 发现率 | 100% | 100% | 100% | 100% |

---

## 十一、决策记录与域名待办(执行 T3 前必读)

> 本节记录已决断的取舍、需用户拍板的遗留点,以及**域名确定后才执行**的 T3。
> git 回滚基线:`67a4ffb`(SEO T0 基线)。T1/T2 在其后,可 `git reset --hard 67a4ffb` 撤销。

### 11.1 已决断(自行决断,可回滚)

| 决策 | 选择 | 理由 | 落地脚本 |
|------|------|------|---------|
| 重复 id 清理 | 保留字段更完整的条目(有 about.content、关键词更多),删除较不完整的一条 | 5 个重复 id(text-diff/regex-test/url-encode/base-convert/unit-convert)各删 1 条,110→105 | `scripts/dedupe_registry.py` |
| 同质化 title | 按"挑与 name 不重叠的关键词"公式重算;高搜索量工具人工精选 | 消除"个税计算器 - 个税计算器"重复,33 个 title 重算,残留 0 | `scripts/t1_seo_content.py` |
| 高搜索量工具内容 | 为 8 个工具补 how_to/description_for_ai/3-4 条 FAQ | 月搜索 200万+ 工具优先,FAQ 喂 FAQPage JSON-LD + AI 引用 | `scripts/t1_seo_content.py` |
| 路由策略 | 统一子目录 `/{id}`,`subdomain` 字段降级为可选 | 新域名无权重,子目录更利集中权重 + 单域名单备案;vite-ssg 预渲染每页 `${id}.html` | nginx `try_files $uri.html` |
| 域名占位集中化 | 新增 `frontend/src/config/site.json` 为 SSOT,canonical/sitemap/robots/llms 全部读它 | 换域名从"改 4 处"降到"改 1 处 + nginx" | gen-sitemap.mjs / ToolLayout / Home |

### 11.2 需用户拍板的遗留点(域名确定后一并处理)

| 点 | 现状 | 待用户决定 |
|----|------|----------|
| 真实域名 | 占位 `https://zhiniao.tools` | 注册哪个域名(.com/.cn,含 tools/gongju 关键词) |
| 备案 | 未备案 | 国内服务器必须,20 工作日;是否走海外 VPS 兜底 |
| 子域名是否启用 | `subdomain` 字段保留但未用 | 个别高流量工具后期是否拆子域名(301 到主站) |
| 政策类内容准确性 | 个税/退休 FAQ 标注"以官方为准" | 是否需法务/财务复核政策类文案 |

### 11.3 域名占位清单(✅ 已落地 2026-07-17:域名确定 bibilabu.cc)

> **域名已确定**:bibilabu.cc(粤ICP备2026096432号-1,珠海友米网络科技有限公司,网站首页 www.bilibabu.cc)。
> 2026-07-17 备案信息落地:① `site.json` domain→https://www.bilibabu.cc + 加 company/icp 主体字段;② `nginx.conf` server_name→bibilabu.cc;③ 页脚展示 ICP 号(链 beian.miit.gov.cn)+公司名;④ openapi server/tool.schema $id 同步。代码区 `DOMAIN_PLACEHOLDER`/`zhiniao.tools` 占位标记已全部清除。
> 下次换域名仍只需:① 改 `site.json` 的 `domain`(+ company/icp);② 改 `nginx.conf` 的 `server_name`;③ 重 build。前端 canonical/sitemap/robots/llms 自动跟随 site.json。

| # | 位置 | 旧占位值 | 现值 |
|---|------|--------|------|
| 1 | `frontend/src/config/site.json` → `domain` | `https://zhiniao.tools` | ✅ `https://www.bilibabu.cc`(+ company/icp 主体字段) |
| 2 | `nginx/nginx.conf` → `server_name`(2 处 + 子域名 1 处) | `zhiniao.tools` | ✅ `bibilabu.cc`/`www.bibilabu.cc`/`*.bibilabu.cc` |
| 3 | `index.html` | title/description 无域名 | 无需改(各页 useHead 覆盖;仅 SPA 兜底) |
| 4 | `schemas/api/openapi.yaml` servers + `schemas/tool.schema.json` $id | `api.zhiniao.tools`/`zhiniao.tools` | ✅ `https://www.bilibabu.cc` |

> robots.txt 的 `Sitemap:` 行由 `gen-sitemap.mjs` 从 site.json 生成,无需手改(重 build 自动变 bibilabu.cc)。

### 11.4 T3 待执行清单(✅ 域名已确定,其余外部资源待办)

> 以下需外部资源/账号,我无法代劳,域名到位后按序执行。

| 步骤 | 动作 | 验证 |
|------|------|------|
| T3-1 | 注册域名 + Cloudflare 泛解析 + 泛证书(免费) | DNS 解析生效 |
| T3-2 | 备案申请(国内服务器)或海外 VPS 兜底 | 备案号/海外可达 |
| T3-3 | 改 site.json `domain` + nginx `server_name`,重 build 部署 | `curl https://真实域名/sitemap.xml` 200 |
| T3-4 | 百度搜索资源平台:加验证 meta + 提交 sitemap + 普通收录 API | sitemap 已发现 |
| T3-5 | Google Search Console:验证 + 提交 sitemap | sitemap 状态绿色 |
| T3-6 | Bing Webmaster:验证 + 提交 sitemap | 已发现 |
| T3-7 | 百度普通收录主动推送:工具发布/更新时调 API(后端) | 推送返回成功 |
| T3-8 | 百度统计 + GA4 接入 | PV/UV 可见 |
| T3-9 | nginx 开 brotli(可选,提升 LCP) | 响应头 br |
| T3-10 | 收录后监控:收录页数/UV(对齐 22 号文档门禁) | 达门禁阈值 |

**站长验证 meta 接入方式**:用 env 注入 `baidu-site-verification`/`google-site-verification`,通过 useHead 加到首页 head(勿提交真实值到 git,用 `.env`)。

---



## 十、相关文档

- [22-platform-pivot-strategy.md](./22-platform-pivot-strategy.md) — 平台战略 + 第九章 GEO
- [02-frontend-design.md](./02-frontend-design.md) — 前端设计
- [06-tool-registry.md](./06-tool-registry.md) — 工具注册表
- [07-consistency-link.md](./07-consistency-link.md) — SSOT 机制
- [09-deployment.md](./09-deployment.md) — 部署(nginx/Cloudflare)
- [27-lessons-learned.md](./27-lessons-learned.md) — 错题集(避坑)
- [01-progress.md](./01-progress.md) — 进度(本方案落地后同步更新)
