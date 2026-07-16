# AI 工具分类引流方案

> **版本**: v1.0
> **最后更新**: 2026-07-10
> **状态**: 设计方案,待确认后实施
> **目的**: 在工具站加 AI 工具分类,引流到自有 New API(海外)+ 发卡(国内)

---

## 一、需求分析

### 1.1 目标

- 在知鸟工具站加一个 **"AI 工具"** 分类
- 该分类下的工具以"介绍 + 跳转"为主(非站内实现)
- 跳转目标: 用户自有的 **New API**(海外部署)+ **发卡系统**(国内部署)
- 工具站作为**流量入口**,把免费工具用户转化为 AI 付费用户

### 1.2 现有分类

| 分类 | 数量 | 说明 |
|------|------|------|
| pdf | 11 | PDF 工具 |
| image | 5 | 图片工具 |
| text | 31 | 文本工具 |
| convert | 5 | 格式转换 |
| calc | 44 | 计算工具 |
| **缺** | - | **无 AI 分类** |

现有 `ai-matting`/`batch-translate` 散落在 image/text,无统一入口。

---

## 二、设计方案

### 2.1 新增 `ai` 分类

**Schema 变更**:
- `schemas/tool.schema.json` 的 `category` 枚举加 `"ai"`
- `frontend/src/pages/Home.vue` 的 `categoryList` 加 AI 分类

### 2.2 AI 工具页面模式:介绍 + 跳转

AI 工具的页面 **不在站内实现功能**,而是:
- 展示工具介绍、功能亮点、使用场景
- 显示"前往使用"按钮,跳转到 New API 或发卡站
- 跳转链接从 registry 的 `external_url` 字段读取(SSOT)

**registry 新增字段**:
```json
{
  "id": "ai-chat",
  "name": "AI 对话",
  "category": "ai",
  "backend": "external",
  "external_url": "https://你的new-api域名",
  "external_desc": "支持 GPT-4/Claude/Gemini,中转 API,按量计费",
  "is_premium": false,
  "route": "/ai-chat",
  "input_schema": [],
  "about": { ... }
}
```

**Schema 扩展**:
- `category` 枚举加 `"ai"`
- `backend` 枚举加 `"external"`
- 新增 `external_url`(跳转地址)
- 新增 `external_desc`(外部服务说明)

### 2.3 AI 工具清单建议

| ID | 名称 | 跳转目标 | 说明 |
|----|------|---------|------|
| ai-chat | AI 对话 | New API | GPT-4/Claude/Gemini 对话 |
| ai-translate | AI 翻译 | New API | 比传统翻译更准 |
| ai-code | AI 编程助手 | New API | 代码补全/审查 |
| ai-image-gen | AI 绘画 | New API(或第三方) | 文生图 |
| ai-doc | AI 文档处理 | New API | PDF 总结/问答 |
| api-key | API Key 购买 | 发卡站 | 购买 API 额度 |
| ai-plus | AI 会员 | 发卡站 | 包月/包年会员 |

### 2.4 前端页面实现

新建通用组件 `ExternalToolCard.vue`:
```vue
<template>
  <ToolLayout :tool="tool">
    <div class="ext-intro">
      <div class="ext-icon">{{ tool.icon }}</div>
      <h2>{{ tool.name }}</h2>
      <p>{{ tool.external_desc }}</p>
      <div class="ext-features">
        <div v-for="f in tool.about?.features" :key="f">✓ {{ f }}</div>
      </div>
      <a :href="tool.external_url" target="_blank" class="ext-btn">
        前往使用 →
      </a>
      <p class="ext-note">将在新窗口打开外部服务</p>
    </div>
  </ToolLayout>
</template>
```

### 2.5 首页 AI 分类入口

Home.vue 分类标签加:
```javascript
{ id: 'ai', name: 'AI', icon: '🤖' }
```

AI 分类的卡片显示"外部服务"角标,区别于站内工具。

---

## 三、合规风险分析

### 3.1 高风险项(必须注意)

| 风险 | 严重度 | 说明 | 对策 |
|------|--------|------|------|
| **New API 涉及境外服务** | 🔴 高 | 海外部署的 AI API 中转,可能涉及"擅自建立信道" | 不在国内服务器部署,仅做链接跳转 |
| **API Key 售卖合规** | 🔴 高 | 倒卖 OpenAI/Anthropic API Key 违反其 ToS | 包装为"API 中转服务"而非"卖 Key",用户买的是额度 |
| **AI 内容合规** | 🟡 中 | AI 生成内容可能涉政/涉黄 | New API 侧做内容过滤,工具站只做跳转不展示内容 |
| **发卡系统国内备案** | 🟡 中 | 发卡站国内部署需备案 | 发卡站独立备案,不与工具站混用 |
| **大模型服务备案** | 🔴 高 | 国内提供生成式 AI 服务需备案 | 工具站不提供 AI 服务,只做链接,服务在海外 |

### 3.2 关键合规原则

**工具站(New API 跳转)侧**:
1. **工具站只做链接,不提供 AI 服务** — 工具站本身不调用大模型,只展示介绍+跳转
2. **不存储 AI 生成内容** — 工具站不存储对话/图片,降低合规风险
3. **免责声明** — 跳转页标注"本服务由第三方提供,知鸟工具箱仅提供信息导航"
4. **不宣传"翻墙"** — 文案用"AI API 中转"而非"科学上网/翻墙"

**New API(海外)侧**:
1. **服务器在海外** — 不受国内大模型备案约束
2. **用户协议明确** — 用户自行承担使用风险
3. **内容过滤** — 接入敏感词过滤,降低涉政风险
4. **不公开宣传** — 仅通过工具站引流,不做公开广告

**发卡系统(国内)侧**:
1. **独立域名+备案** — 不与工具站混用
2. **商品描述规避敏感词** — 不写"GPT-4 Key",写"AI API 额度"
3. **虚拟商品自动发货** — 符合电商平台规则

### 3.3 不可做的事

| 禁止 | 原因 |
|------|------|
| ❌ 工具站直接嵌入 AI 对话框 | 等于工具站提供 AI 服务,需备案 |
| ❌ 工具站存储 AI 对话记录 | 内容合规风险 |
| ❌ 宣传"免费 GPT-4"/"翻墙 API" | 涉嫌违规 |
| ❌ 发卡站和工具站同域名 | 备案风险关联 |
| ❌ 工具站页面展示 AI 生成的实时内容 | 内容合规风险 |

---

## 四、实施步骤

### 4.1 Schema 扩展

1. `tool.schema.json`: category 加 `ai`,backend 加 `external`,新增 `external_url`/`external_desc`
2. `tools.registry.json`: 添加 7 个 AI 工具定义
3. `Home.vue`: 分类标签加 AI

### 4.2 前端

1. 创建 `ExternalToolCard.vue`(通用跳转卡片)
2. AI 工具路由指向 `ExternalToolCard`
3. AI 卡片加"外部"角标

### 4.3 内容

- 每个工具页写介绍文案(功能/场景/价格)
- 跳转按钮指向 New API 或发卡站
- 底部免责声明

### 4.4 引流路径

```
用户搜索"PDF转Word" → 知鸟工具站 → 用免费工具 → 看到"AI 工具"分类
  → 点击"AI 对话" → 跳转 New API → 注册使用
  → 需要额度 → 点击"API Key 购买" → 跳转发卡站 → 购买
```

---

## 五、建议的落地优先级

| 优先级 | 任务 | 说明 |
|--------|------|------|
| P0 | Schema 扩展 + 7 个 AI 工具 registry | 基础设施 |
| P0 | ExternalToolCard 组件 + 路由 | 跳转页面 |
| P0 | 免责声明 + 外部角标 | 合规 |
| P1 | AI 工具介绍文案 | SEO + 转化 |
| P2 | 首页 AI 分类入口优化 | 引流效果 |
| P3 | 跳转统计(utm 参数) | 效果追踪 |

---

## 六、相关文档

- [22-platform-pivot-strategy.md](./22-platform-pivot-strategy.md) — 平台战略
- [31-usability-optimization.md](./31-usability-optimization.md) — 易用性优化
- [06-tool-registry.md](./06-tool-registry.md) — 工具注册表
