# 前端设计方案

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **依赖文档**: [05-design-tokens.md](./05-design-tokens.md), [06-tool-registry.md](./06-tool-registry.md)
> **状态**: 原型已完成,待 Vue 工程化

---

## 一、设计理念

### 1.1 核心原则

1. **极简交互**(借鉴 bmcx): 用户来即用,核心操作一屏完成
2. **工具矩阵互链**(借鉴 bmcx): 每个工具页导流到其他工具
3. **免费+付费双层**: 标准转换免费,高质量转换付费
4. **现代化设计**(不借鉴 bmcx): 渐变色、圆角、阴影,告别 table 布局
5. **移动优先**: 响应式设计,移动端体验优先

### 1.2 对标站点分析

详见 [prototypes/](./prototypes/) 下的实际原型。对标 bmcx.com 的分析:

| 借鉴 | 不借鉴 |
|------|--------|
| 子域名结构(利于 SEO) | 全免费模式 |
| 极简交互(一屏完成) | 纯广告变现 |
| 工具矩阵互链 | 老旧 table 布局 |
| 原创说明内容(SEO) | 广告密集(11 个) |
| localStorage 偏好 | 无用户系统 |
| 历史记录功能 | - |

---

## 二、技术选型

| 组件 | 技术 | 理由 |
|------|------|------|
| 框架 | Vue 3 + Vite | Stirling-PDF 前端也用 Vue,无缝集成 |
| 语言 | TypeScript | 类型安全,配合 OpenAPI 生成 |
| UI 库 | Element Plus | 中文友好,组件丰富 |
| 样式 | Tailwind CSS + Design Tokens | 快速迭代 + 一致性 |
| 路由 | Vue Router | SPA,每工具独立路由 |
| 状态 | Pinia | 轻量,够用 |
| HTTP | Axios + 自动生成 Client | 从 OpenAPI 生成 |
| 部署 | Nginx 静态 + Cloudflare CDN | 全球加速 |
| SEO | vite-ssg (预渲染) | 解决 SPA SEO 问题 |

---

## 三、目录结构

```
frontend/
├─ public/
├─ src/
│  ├─ main.ts                    # 入口
│  ├─ App.vue
│  ├─ router/
│  │  └─ index.ts                # 从 tool-registry 自动生成
│  ├─ stores/                    # Pinia stores
│  │  ├─ user.ts                 # 用户/余额
│  │  └─ tools.ts                # 工具列表(从 registry 加载)
│  ├─ api/                       # 从 OpenAPI 自动生成
│  │  └─ client.ts
│  ├─ components/                # 通用组件
│  │  ├─ UploadBox.vue
│  │  ├─ ToolCard.vue
│  │  ├─ QualitySelector.vue
│  │  └─ ProgressBar.vue
│  ├─ layouts/
│  │  ├─ DefaultLayout.vue       # 主站布局
│  │  └─ ToolLayout.vue          # 工具页布局
│  ├─ tools/                     # 各工具页面(模块化)
│  │  ├─ pdf-to-word/
│  │  │  ├─ index.vue
│  │  │  ├─ api.ts
│  │  │  └─ config.ts            # 工具元数据
│  │  ├─ pdf-merge/
│  │  └─ ...
│  ├─ pages/                     # 非工具页
│  │  ├─ Home.vue
│  │  ├─ Redeem.vue
│  │  ├─ UserCenter.vue
│  │  └─ Pricing.vue
│  ├─ assets/
│  │  └─ styles/
│  │     ├─ tokens.css           # 从 design-tokens.json 生成
│  │     └─ global.css
│  └─ utils/
├─ tokens/
│  └─ design-tokens.json         # 单一来源(参见 schemas/)
├─ tools/
│  └─ tools.registry.json        # 单一来源(参见 schemas/)
└─ openapi.yaml                  # 单一来源(参见 schemas/api/)
```

---

## 四、页面设计

### 4.1 首页(工具目录)

**目标**: 工具发现,引导用户找到需要的工具

**布局**:
```
┌────────────────────────────────────────┐
│ Logo  [搜索]   兑换卡密 我的账户 定价 登录│
├────────────────────────────────────────┤
│         🛠️ 知鸟工具箱                  │
│   50+ 在线工具,免费好用                │
│   [大搜索框 + 热门标签]                │
├────────────────────────────────────────┤
│ 分类标签: 全部 PDF 图片 文本 转换 计算  │
├────────────────────────────────────────┤
│ 📄 PDF 工具 (8)                        │
│  [工具卡片网格 - 每卡含图标/名/描述/徽章]│
│ 🖼️ 图片工具 (6)                        │
│  [工具卡片网格]                        │
│ 🔄 格式转换 (5)                        │
│  [工具卡片网格]                        │
│ ...                                    │
├────────────────────────────────────────┤
│ 价值主张: 本地处理/隐私/高质量/大部分免费 │
├────────────────────────────────────────┤
│ CTA: 包月会员引导                      │
├────────────────────────────────────────┤
│ 页脚: 工具导航 + 备案                  │
└────────────────────────────────────────┘
```

**实现要点**:
- 工具卡片从 `tools.registry.json` 自动渲染
- 分类筛选客户端过滤
- 搜索框支持工具名/关键词搜索

### 4.2 工具页(以 PDF转Word 为例)

**目标**: 极简交互,一屏完成核心操作

**布局**:
```
┌────────────────────────────────────────┐
│ Logo [搜索]               登录 定价    │
├────────────────────────────────────────┤
│ 首页 › PDF工具 › PDF转Word [高质量标签] │
├────────────────────────────────────────┤
│        📄 PDF 转 Word                  │
│   高质量 OCR 转换,支持扫描件           │
│                                        │
│   ┌──────────────────────────────┐    │
│   │  📁 拖拽 PDF 文件到此处       │    │
│   │     或 [点击上传]            │    │
│   └──────────────────────────────┘    │
│                                        │
│   转换质量:                            │
│   ┌────────────┐ ┌────────────┐       │
│   │📋 标准      │ │💎 高质量   │       │
│   │  免费       │ │  ¥2/次     │       │
│   └────────────┘ └────────────┘       │
│                                        │
│   ☑ OCR 识别   格式: [docx ▼]         │
│                                        │
│   [🚀 开始转换 (免费/¥2)]              │
│                                        │
│   [进度条 + 阶段文案]                  │
│   [结果卡片: 下载/重新转换]            │
│   [付费引导卡片(免费转换后)]           │
├────────────────────────────────────────┤
│ 相关工具: PDF合并 · PDF压缩 · OCR ...  │
├────────────────────────────────────────┤
│ 关于 PDF 转 Word(SEO 内容 + FAQ)      │
└────────────────────────────────────────┘
```

**关键交互**:
1. 拖拽/点击上传,支持多文件
2. 质量:标准(免费 LibreOffice)/高质量(¥2 WPS+OCR)
3. 高质量转换前检查余额,不足跳转兑换页
4. 免费转换完成后弹"试试高质量版"引导卡片
5. 进度多阶段文案(上传/调用引擎/转换/OCR/生成)

### 4.3 卡密兑换页

**目标**: 用户输入闲鱼购买的卡密,激活次数

**布局**:
```
┌────────────────────────────────────────┐
│ Logo                  工具首页 我的账户  │
├────────────────────────────────────────┤
│           🎫                            │
│        卡密兑换                        │
│   输入购买后获取的卡密,激活次数        │
│                                        │
│   卡密 (Card Key)                      │
│   [ZN-XXXX-XXXX-XXXX          ]        │
│   ✓ 格式正确,点击兑换                  │
│                                        │
│   [🚀 立即兑换]                        │
│                                        │
│   ┌──────────────────────────────┐    │
│   │ ⚠️ 还没有卡密?                │    │
│   │ 闲鱼/拼多多搜索「PDF转Word」   │    │
│   │ [🐟 闲鱼] [🛒 拼多多] [💬 微信]│    │
│   └──────────────────────────────┘    │
│                                        │
│   📌 卡密使用说明                      │
│   • 不绑定设备,任意浏览器可用          │
│   • 兑换后立即到账                     │
│   • 有效期 90 天                       │
└────────────────────────────────────────┘
```

**交互**:
- 卡密输入实时格式校验(ZN- 开头,19 位)
- 兑换成功弹窗显示余额
- 失败提示"卡密无效或已使用"

### 4.4 用户中心

**目标**: 查看余额、记录,快速操作

**布局**:
```
┌────────────────────────────────────────┐
│ Logo                  工具首页 [头像]   │
├────────────────────────────────────────┤
│ ┌──────────────────────────────────┐  │
│ │ 高质量转换次数                    │  │
│ │ 100 次          [兑换卡密][购买]  │  │
│ │ 最近兑换: 2026-07-02 14:32       │  │
│ └──────────────────────────────────┘  │
│                                        │
│ [⚠️ 余额不足横幅(余额<5时显示)]         │
│                                        │
│ 快速操作:                              │
│ [PDF转Word] [兑换卡密] [购买] [批量]   │
│                                        │
│ Tab: 卡密记录 | 转换记录 | 订单记录    │
│ ┌──────────────────────────────────┐  │
│ │ 🎫 100次包 卡密兑换              │  │
│ │ 2026-07-02 14:32  +100次  已兑换 │  │
│ │ ...                              │  │
│ └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

---

## 五、组件设计

### 5.1 通用组件

| 组件 | 用途 | Props |
|------|------|-------|
| `UploadBox` | 文件上传(拖拽+点击) | accept, multiple, maxSize |
| `ToolCard` | 工具卡片(首页用) | tool(从 registry) |
| `QualitySelector` | 质量(标准/高质量)选择 | v-model, onBalanceCheck |
| `ProgressBar` | 多阶段进度条 | stages, current, percent |
| `ResultCard` | 转换结果展示 | fileName, duration, downloadUrl |
| `UpsellCard` | 付费引导卡片 | targetQuality |

### 5.2 工具页模板

每个工具页遵循统一模板:

```vue
<template>
  <ToolLayout :tool="toolMeta">
    <UploadBox @upload="onUpload" />
    <QualitySelector v-model="quality" :balance="balance" />
    <OptionGroup v-model="options" :schema="toolMeta.options" />
    <ConvertButton :quality="quality" :balance="balance" @click="onConvert" />
    <ProgressBar :stages="stages" v-if="converting" />
    <ResultCard :result="result" v-if="done" />
    <UpsellCard v-if="showUpsell" />
    <RelatedTools :tools="relatedTools" />
    <AboutTool :content="toolMeta.about" />
  </ToolLayout>
</template>
```

---

## 六、状态管理

### 6.1 User Store

```typescript
// stores/user.ts
interface UserState {
  isLogin: boolean
  userId: string | null
  balance: number              // 高质量转换剩余次数
  apiToken: string | null      // 卡密兑换后生成的 token
}

// 关键 action
- redeemCard(key: string)      // 兑换卡密
- checkBalance()               // 查询余额
- consumeBalance(n: number)    // 消耗次数
```

### 6.2 Tools Store

```typescript
// stores/tools.ts
interface ToolsState {
  list: Tool[]                 // 从 registry 加载
  categories: string[]
  filter: { category: string; keyword: string }
}
```

---

## 七、SEO 策略

### 7.1 路由结构

| 工具类型 | 路由 | 示例 |
|---------|------|------|
| 高频工具 | 子域名 | pdf.zhiniao.tools |
| 长尾工具 | 子路径 | zhiniao.tools/tools/uuid |

### 7.2 每页 SEO 元素

```typescript
// 每个工具的 config.ts 自动生成
export const seo = {
  title: 'PDF 转 Word - 高质量 OCR 转换 | 知鸟工具箱',
  description: '免费在线 PDF 转 Word...',
  keywords: ['pdf转word', 'pdf转换', 'ocr'],
  canonical: 'https://zhiniao.tools/pdf-to-word',
  schema: {
    '@type': 'SoftwareApplication',
    name: 'PDF 转 Word',
    offers: { price: '0', currency: 'CNY' }
  }
}
```

### 7.3 预渲染

用 `vite-ssg` 对所有工具页做预渲染,解决 SPA 的 SEO 问题。

---

## 八、与后端对接

前端所有 API 调用从 `openapi.yaml` 自动生成:

```typescript
// 自动生成的 api/client.ts
import { ApiClient } from './generated'

const client = new ApiClient({ baseUrl: '/api' })

// 调用
const result = await client.convert.convert({
  pdfFile: file,
  quality: 'premium',
  ocr: true
})
```

**关键**: 前端不手写 API 调用,全部从 OpenAPI 生成,保证与后端契约一致。

详见 [04-api-contract.md](./04-api-contract.md) 和 [07-consistency-link.md](./07-consistency-link.md)。

---

## 九、原型文件

| 原型 | 路径 | 说明 |
|------|------|------|
| 首页 | `prototypes/index.html` | 工具目录 |
| PDF转Word | `prototypes/pdf-to-word.html` | 核心付费工具页 |
| 卡密兑换 | `prototypes/redeem.html` | 卡密输入激活 |
| 用户中心 | `prototypes/user-center.html` | 余额和记录 |

原型已实现完整闭环演示,可直接浏览器打开查看。

---

## 十、待办

- [ ] 转 Vue3 工程
- [ ] 接入 Design Tokens 自动生成
- [ ] 接入 Tool Registry 自动生成路由
- [ ] 接入 OpenAPI 自动生成 API Client
- [ ] 实现预渲染(SEO)
- [ ] 移动端适配验证
