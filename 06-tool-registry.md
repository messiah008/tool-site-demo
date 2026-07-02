# 工具注册表 (Tool Registry)

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **文件位置**: `schemas/tools.registry.json`
> **Schema**: `schemas/tool.schema.json`
> **状态**: 设计完成
> **作用**: 工具元数据的**单一事实来源**,前端/后端共享

---

## 一、工具注册表的作用

工具注册表是所有工具元数据的**唯一真相**,前端和后端都从这份文件读取:

```
tools.registry.json (单一来源)
    │
    ├─→ 前端: 自动生成路由、首页卡片、SEO 元数据
    ├─→ 后端: 自动注册 API 路由、校验工具 ID
    └─→ AI: 通过 schema 校验工具定义是否合法
```

**核心原则**:
1. 新增工具**只改** `tools.registry.json`,不改代码
2. 前端路由、首页卡片从 registry 自动渲染
3. 后端 API 校验 `tool_id` 必须在 registry 中存在
4. AI 添加工具前必须读 schema,改完必须更新 registry

---

## 二、工具 Schema

### 2.1 JSON Schema 定义

文件位置: `schemas/tool.schema.json`

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Tool Definition",
  "description": "知鸟工具箱 - 工具定义 Schema",
  "type": "object",
  "required": ["id", "name", "category", "backend", "is_premium"],
  "additionalProperties": false,
  "properties": {
    "id": {
      "type": "string",
      "pattern": "^[a-z0-9-]+$",
      "description": "工具 ID,小写+连字符"
    },
    "name": { "type": "string", "description": "工具显示名" },
    "description": { "type": "string" },
    "category": {
      "type": "string",
      "enum": ["pdf", "image", "text", "convert", "calc", "query"]
    },
    "icon": { "type": "string", "description": "Emoji 图标" },
    "icon_color": {
      "type": "string",
      "enum": ["pdf", "image", "text", "convert", "calc", "query"]
    },
    "is_premium": {
      "type": "boolean",
      "description": "是否付费工具"
    },
    "backend": {
      "type": "string",
      "enum": ["frontend", "stirling", "wps-worker"],
      "description": "后端类型: frontend=纯前端, stirling=Stirling-PDF, wps-worker=WPS Worker"
    },
    "route": { "type": "string", "description": "前端路由路径" },
    "subdomain": { "type": "string", "description": "子域名(可选)" },
    "keywords": {
      "type": "array",
      "items": { "type": "string" }
    },
    "options": {
      "type": "array",
      "description": "工具特定选项",
      "items": {
        "type": "object",
        "required": ["name", "type"],
        "properties": {
          "name": { "type": "string" },
          "type": { "enum": ["boolean", "select", "text"] },
          "default": {},
          "options": { "type": "array" }
        }
      }
    },
    "about": {
      "type": "object",
      "description": "关于工具的内容(SEO)",
      "properties": {
        "title": { "type": "string" },
        "content": { "type": "string" },
        "features": { "type": "array", "items": { "type": "string" } },
        "scenarios": { "type": "array", "items": { "type": "string" } },
        "faq": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "q": { "type": "string" },
              "a": { "type": "string" }
            }
          }
        }
      }
    }
  }
}
```

---

## 三、工具注册表实例

文件位置: `schemas/tools.registry.json`

```json
[
  {
    "id": "pdf-to-word",
    "name": "PDF 转 Word",
    "description": "高质量 OCR 转换,支持扫描件",
    "category": "pdf",
    "icon": "📄",
    "icon_color": "pdf",
    "is_premium": true,
    "backend": "wps-worker",
    "route": "/pdf-to-word",
    "subdomain": "pdf",
    "keywords": ["pdf转word", "pdf转换", "ocr识别"],
    "options": [
      {
        "name": "ocr",
        "type": "boolean",
        "default": true,
        "description": "启用 OCR 识别(扫描件必选)"
      },
      {
        "name": "format",
        "type": "select",
        "default": "docx",
        "options": ["docx", "doc"],
        "description": "输出格式"
      }
    ],
    "about": {
      "title": "关于 PDF 转 Word",
      "content": "PDF 转 Word 是日常办公中最常用的文档转换需求之一...",
      "features": [
        "标准转换(免费):基于 LibreOffice 引擎",
        "高质量转换(付费):基于 WPS 引擎 + OCR 识别",
        "隐私安全:文件处理后 1 小时自动删除"
      ],
      "scenarios": [
        "学生:扫描教材、论文转 Word 编辑",
        "财务:扫描发票、报表转 Excel/Word",
        "行政:合同、公文 PDF 转 Word 修改"
      ],
      "faq": [
        {
          "q": "扫描件 PDF 转换后是图片还是文字?",
          "a": "标准转换不会识别图片中的文字,输出仍是图片。高质量转换启用 OCR,能将扫描图片中的文字识别为可编辑文本。"
        }
      ]
    }
  },
  {
    "id": "pdf-merge",
    "name": "PDF 合并",
    "description": "多个 PDF 合并为一个文件",
    "category": "pdf",
    "icon": "📑",
    "icon_color": "pdf",
    "is_premium": false,
    "backend": "stirling",
    "route": "/pdf-merge",
    "keywords": ["pdf合并", "合并pdf"],
    "about": { "title": "关于 PDF 合并", "content": "..." }
  },
  {
    "id": "pdf-compress",
    "name": "PDF 压缩",
    "description": "无损压缩 PDF 体积",
    "category": "pdf",
    "icon": "🗜️",
    "icon_color": "pdf",
    "is_premium": false,
    "backend": "stirling",
    "route": "/pdf-compress",
    "keywords": ["pdf压缩", "压缩pdf"]
  },
  {
    "id": "pdf-ocr",
    "name": "PDF OCR",
    "description": "扫描件文字识别提取",
    "category": "pdf",
    "icon": "🔤",
    "icon_color": "pdf",
    "is_premium": true,
    "backend": "stirling",
    "route": "/pdf-ocr",
    "keywords": ["pdf ocr", "文字识别"]
  },
  {
    "id": "image-compress",
    "name": "图片压缩",
    "description": "无损/有损压缩可选",
    "category": "image",
    "icon": "🗜️",
    "icon_color": "image",
    "is_premium": false,
    "backend": "frontend",
    "route": "/image-compress",
    "keywords": ["图片压缩", "压缩图片"]
  },
  {
    "id": "id-photo",
    "name": "证件照制作",
    "description": "AI 抠图换底,各类尺寸",
    "category": "image",
    "icon": "🎨",
    "icon_color": "image",
    "is_premium": true,
    "backend": "wps-worker",
    "route": "/id-photo",
    "keywords": ["证件照", "ai抠图"]
  },
  {
    "id": "json-format",
    "name": "JSON 格式化",
    "description": "JSON 美化/压缩/校验",
    "category": "text",
    "icon": "🔤",
    "icon_color": "text",
    "is_premium": false,
    "backend": "frontend",
    "route": "/json-format",
    "keywords": ["json格式化", "json美化"]
  },
  {
    "id": "password-gen",
    "name": "密码生成",
    "description": "随机安全密码生成",
    "category": "calc",
    "icon": "🔑",
    "icon_color": "calc",
    "is_premium": false,
    "backend": "frontend",
    "route": "/password-gen",
    "keywords": ["密码生成", "随机密码"]
  }
]
```

完整注册表见 `schemas/tools.registry.json`(30+ 工具)。

---

## 四、代码生成

### 4.1 前端生成路由

```typescript
// scripts/gen-routes.ts
import registry from '../schemas/tools.registry.json'
import fs from 'fs'

const routes = registry.map(tool => ({
  path: tool.route,
  name: tool.id,
  component: () => import(`@/tools/${tool.id}/index.vue`),
  meta: { tool }
}))

fs.writeFileSync(
  'src/router/auto-routes.ts',
  `export const autoRoutes = ${JSON.stringify(routes, null, 2)}`
)
```

### 4.2 前端生成首页卡片

```typescript
// scripts/gen-cards.ts
import registry from '../schemas/tools.registry.json'

export const tools = registry
export const categories = [...new Set(registry.map(t => t.category))]
export const premiumTools = registry.filter(t => t.is_premium)
```

### 4.3 后端校验工具 ID

```python
# app/api/convert.py
from app.utils.tool_registry import TOOLS

@router.post("/api/convert")
async def create_convert(..., tool_id: str = Form(...)):
    if tool_id not in TOOLS:
        raise HTTPException(400, detail={
            "error_code": "E404",
            "message": f"工具 {tool_id} 不存在"
        })
    tool = TOOLS[tool_id]
    # ...
```

### 4.4 后端启动时加载

```python
# app/utils/tool_registry.py
import json
from pathlib import Path

_registry_path = Path(__file__).parent.parent.parent / "schemas" / "tools.registry.json"
_registry = json.loads(_registry_path.read_text(encoding="utf-8"))
TOOLS = {t["id"]: t for t in _registry}

def get_tool(tool_id: str):
    return TOOLS.get(tool_id)

def list_tools(category: str = None):
    if category:
        return [t for t in TOOLS.values() if t["category"] == category]
    return list(TOOLS.values())
```

---

## 五、添加新工具流程

### 5.1 步骤

1. 在 `schemas/tools.registry.json` 添加工具定义
2. 用 schema 校验: `ajv validate -s tool.schema.json -d tools.registry.json`
3. 前端: 创建 `src/tools/{id}/index.vue`
4. 后端: 如果是新 backend 类型,实现对应 handler
5. 测试: 启动前端,首页自动出现新卡片

### 5.2 AI 添加工具规范

AI 接到"添加 XX 工具"时:
1. 先读 `schemas/tool.schema.json` 了解字段
2. 读 `schemas/tools.registry.json` 确认 ID 不重复
3. 添加工具定义,执行 schema 校验
4. 创建前端页面
5. 在 [01-progress.md](./01-progress.md) 记录变更

详见 [08-ai-collaboration.md](./08-ai-collaboration.md)。

---

## 六、版本管理

- 新增工具: 在数组末尾追加
- 修改工具: 保持 ID 不变,改其他字段
- 删除工具: 标记 `deprecated: true`,不直接删除(避免外链失效)
- 变更记录在 [01-progress.md](./01-progress.md)
