# 设计令牌 (Design Tokens)

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **文件位置**: `schemas/design-tokens.json`
> **状态**: 设计完成
> **作用**: 视觉一致性的**单一事实来源**,自动生成 CSS 变量

---

## 一、设计令牌的作用

设计令牌是设计系统的**原子单位**,所有视觉属性(颜色/间距/字号/圆角等)集中定义,自动生成各平台变量。

```
design-tokens.json (单一来源)
    │
    ├─→ CSS 变量 (tokens.css) → 前端引用
    ├─→ SCSS 变量 → 主题定制
    ├─→ Tailwind 配置 → 工具类
    └─→ Figma Tokens → 设计师同步
```

**核心原则**:
1. 改令牌 → 自动生成 → 所有页面同步更新
2. 前端不写死颜色值,全部引用变量
3. AI 修改视觉前必须改令牌,改完执行生成命令

---

## 二、令牌定义

### 2.1 完整 JSON

```json
{
  "$schema": "https://design-tokens.github.io/community-group/format/",
  "$description": "知鸟工具箱设计令牌 - 视觉一致性的单一事实来源",
  "version": "1.0.0",
  "last_updated": "2026-07-02",

  "color": {
    "primary": {
      "$value": "#4f46e5",
      "$description": "主色 - 紫色,品牌识别"
    },
    "primary_light": {
      "$value": "#6366f1",
      "$description": "主色浅 - 渐变色"
    },
    "primary_bg": {
      "$value": "#eef2ff",
      "$description": "主色背景 - 浅紫"
    },

    "text": {
      "$value": "#1f2937",
      "$description": "主文本"
    },
    "text_light": {
      "$value": "#6b7280",
      "$description": "次要文本"
    },

    "border": {
      "$value": "#e5e7eb",
      "$description": "边框"
    },

    "success": { "$value": "#10b981", "$description": "成功" },
    "success_bg": { "$value": "#d1fae5" },
    "warning": { "$value": "#f59e0b", "$description": "警告/付费" },
    "warning_bg": { "$value": "#fef3c7" },
    "danger": { "$value": "#ef4444", "$description": "错误" },
    "danger_bg": { "$value": "#fee2e2" },

    "bg": { "$value": "#f9fafb", "$description": "页面背景" }
  },

  "spacing": {
    "xs": { "$value": "4px" },
    "sm": { "$value": "8px" },
    "md": { "$value": "12px" },
    "lg": { "$value": "16px" },
    "xl": { "$value": "24px" },
    "2xl": { "$value": "32px" },
    "3xl": { "$value": "48px" }
  },

  "radius": {
    "sm": { "$value": "4px" },
    "md": { "$value": "8px" },
    "lg": { "$value": "12px" },
    "xl": { "$value": "16px" },
    "full": { "$value": "9999px" }
  },

  "font": {
    "family": {
      "$value": "-apple-system, BlinkMacSystemFont, 'Segoe UI', 'PingFang SC', 'Microsoft YaHei', sans-serif"
    },
    "size": {
      "xs": { "$value": "12px" },
      "sm": { "$value": "13px" },
      "md": { "$value": "14px" },
      "lg": { "$value": "16px" },
      "xl": { "$value": "20px" },
      "2xl": { "$value": "24px" },
      "3xl": { "$value": "32px" }
    },
    "weight": {
      "normal": { "$value": "400" },
      "medium": { "$value": "500" },
      "semibold": { "$value": "600" },
      "bold": { "$value": "700" }
    }
  },

  "shadow": {
    "sm": { "$value": "0 1px 3px rgba(0,0,0,0.04)" },
    "md": { "$value": "0 4px 16px rgba(0,0,0,0.06)" },
    "lg": { "$value": "0 8px 24px rgba(0,0,0,0.08)" },
    "primary": { "$value": "0 6px 20px rgba(79,70,229,0.3)" }
  },

  "transition": {
    "fast": { "$value": "0.2s" },
    "normal": { "$value": "0.3s" }
  },

  "z_index": {
    "dropdown": { "$value": "100" },
    "sticky": { "$value": "100" },
    "modal": { "$value": "1000" },
    "toast": { "$value": "1001" }
  }
}
```

---

## 三、代码生成

### 3.1 生成 CSS 变量

```bash
# 用 Style Dictionary 生成
npm install -D style-dictionary

# 配置 scripts/sd.config.json
npx style-dictionary build
```

输出 `frontend/src/assets/styles/tokens.css`:

```css
:root {
  /* color */
  --color-primary: #4f46e5;
  --color-primary-light: #6366f1;
  --color-primary-bg: #eef2ff;
  --color-text: #1f2937;
  --color-text-light: #6b7280;
  --color-border: #e5e7eb;
  --color-success: #10b981;
  --color-success-bg: #d1fae5;
  --color-warning: #f59e0b;
  --color-warning-bg: #fef3c7;
  --color-danger: #ef4444;
  --color-danger-bg: #fee2e2;
  --color-bg: #f9fafb;

  /* spacing */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 12px;
  --spacing-lg: 16px;
  --spacing-xl: 24px;
  --spacing-2xl: 32px;
  --spacing-3xl: 48px;

  /* radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
  --radius-full: 9999px;

  /* font */
  --font-family: -apple-system, BlinkMacSystemFont, ...;
  --font-size-xs: 12px;
  /* ... */

  /* shadow */
  --shadow-sm: 0 1px 3px rgba(0,0,0,0.04);
  /* ... */
}
```

### 3.2 生成 Tailwind 配置

```javascript
// tailwind.config.js (自动生成片段)
import tokens from './schemas/design-tokens.json'

export default {
  theme: {
    extend: {
      colors: {
        primary: tokens.color.primary.$value,
        'primary-light': tokens.color.primary_light.$value,
        'primary-bg': tokens.color.primary_bg.$value,
        // ...
      },
      spacing: {
        xs: tokens.spacing.xs.$value,
        sm: tokens.spacing.sm.$value,
        // ...
      },
      borderRadius: {
        sm: tokens.radius.sm.$value,
        md: tokens.radius.md.$value,
        // ...
      }
    }
  }
}
```

---

## 四、使用规范

### 4.1 前端使用

**正确**:
```vue
<style scoped>
.card {
  background: white;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  padding: var(--spacing-xl);
  box-shadow: var(--shadow-md);
}
.button {
  background: var(--color-primary);
  color: white;
  padding: var(--spacing-md) var(--spacing-lg);
}
</style>
```

**错误**(禁止硬编码):
```vue
<style scoped>
.card {
  border: 1px solid #e5e7eb;     /* ❌ */
  border-radius: 12px;            /* ❌ */
  padding: 24px;                   /* ❌ */
}
</style>
```

### 4.2 Tailwind 工具类

```vue
<template>
  <!-- 正确: 用 Tailwind 工具类(映射到 tokens) -->
  <div class="p-xl rounded-lg border border-border shadow-md">
    <button class="bg-primary text-white p-md rounded-md">按钮</button>
  </div>
</template>
```

---

## 五、变更流程

1. 改 `schemas/design-tokens.json`
2. 执行 `npm run build:tokens` 生成 CSS/Tailwind
3. 提交 PR,视觉走查
4. 合并后所有页面自动应用新令牌

**禁止** 直接改 `tokens.css`(会被覆盖)。

---

## 六、主题扩展(未来)

设计令牌结构支持多主题:

```json
{
  "themes": {
    "light": { "color": { "bg": "#f9fafb" } },
    "dark": { "color": { "bg": "#1f2937" } }
  }
}
```

通过 `data-theme` 属性切换:
```html
<body data-theme="dark">
  <!-- 自动应用 dark 主题令牌 -->
</body>
```
