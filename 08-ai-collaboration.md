# AI 协作规范

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **适用对象**: 所有 AI 助手(kscc/Cursor/Copilot Chat 等)
> **状态**: 强制规范,接手必读

---

## 一、AI 接手须知

### 1.1 必读文档(按顺序)

接手本项目的 AI 必须按顺序读:

1. [README.md](./README.md) — 项目总览
2. [01-progress.md](./01-progress.md) — 当前进度
3. [07-consistency-link.md](./07-consistency-link.md) — 一致性机制(核心)
4. **本文档** — 你的工作规范
5. 与任务相关的具体文档

### 1.2 核心原则

**所有变更从 schema 开始,不直接改代码**:

```
✅ 正确流程:
  改 schema → 生成代码 → 修改业务逻辑 → CI 校验 → 提交

❌ 错误流程:
  直接改代码 → 忘改 schema → CI 失败 → 一致性破坏
```

### 1.3 四大单一事实来源

| Schema | 路径 | 控制维度 |
|--------|------|---------|
| 设计令牌 | `schemas/design-tokens.json` | 视觉一致性 |
| 工具注册表 | `schemas/tools.registry.json` | 工具一致性 |
| API 契约 | `schemas/api/openapi.yaml` | 接口一致性 |
| AI 规范 | `.claude/CLAUDE.md` | 上下文一致性 |

---

## 二、任务执行规范

### 2.1 接到任务后的流程

```
1. 识别任务类型(前端/后端/工具/视觉/接口)
2. 读对应的 schema 文件
3. 改 schema(如需要)
4. 执行生成命令
5. 修改业务代码
6. 本地校验
7. 更新 01-progress.md
8. 提交
```

### 2.2 任务类型对照表

| 任务 | 改什么 | 生成命令 |
|------|--------|---------|
| 加新工具 | `tools.registry.json` + 前端页面 | 无(运行时加载) |
| 改视觉 | `design-tokens.json` | `npm run build:tokens` |
| 改接口 | `openapi.yaml` | `npm run gen:api` |
| 改业务逻辑 | 业务代码 | 无 |
| 加新页面 | 前端代码 + registry(如工具页) | 无 |

---

## 三、典型任务示范

### 3.1 任务: 添加"PDF 加水印"工具

**步骤**:

1. 读 `schemas/tool.schema.json` 了解字段
2. 读 `schemas/tools.registry.json` 确认 ID `pdf-watermark` 不存在
3. 在 registry 末尾添加:
```json
{
  "id": "pdf-watermark",
  "name": "PDF 加水印",
  "description": "批量添加文字/图片水印",
  "category": "pdf",
  "icon": "📝",
  "icon_color": "pdf",
  "is_premium": false,
  "backend": "stirling",
  "route": "/pdf-watermark",
  "keywords": ["pdf水印", "加水印"]
}
```

4. 校验: `ajv validate -s schemas/tool.schema.json -d schemas/tools.registry.json`
5. 创建前端页面 `frontend/src/tools/pdf-watermark/index.vue`
6. 启动前端,首页自动出现卡片
7. 在 `01-progress.md` 记录变更

**禁止**: 不要手动改路由文件、不要手动加首页卡片(自动生成)。

### 3.2 任务: 把主色从紫色改成蓝色

**步骤**:

1. 改 `schemas/design-tokens.json`:
```json
"primary": { "$value": "#2563eb" }
```

2. 执行 `npm run build:tokens`
3. 所有页面自动更新主色
4. 检查无硬编码色值: `bash scripts/check-design-tokens.sh`
5. 提交 `design-tokens.json` + `tokens.css`(生成产物)

**禁止**: 不要逐个页面改颜色。

### 3.3 任务: 给 Convert API 加 `language` 字段

**步骤**:

1. 改 `schemas/api/openapi.yaml`:
```yaml
ConvertRequest:
  properties:
    language:
      type: string
      default: 'zh'
```

2. 前端: `npm run gen:api` → TS 类型自动更新
3. 后端: FastAPI 路由加 `language` 参数
4. 后端校验: `python scripts/compare_openapi.py`
5. 前端调用处使用新字段
6. 提交 `openapi.yaml` + 生成产物 + 业务代码

**禁止**: 不要手写 TS 类型,不要手写 API Client。

---

## 四、多 AI 并行协作

### 4.1 分工原则

| AI 负责 | 可改 schema | 不可改 schema |
|---------|-----------|-------------|
| 前端 AI | design-tokens.json | openapi.yaml(只读) |
| 后端 AI | openapi.yaml | design-tokens.json(只读) |
| 工具 AI | tools.registry.json | 其他 |
| 视觉 AI | design-tokens.json | tools.registry.json(只读) |

### 4.2 冲突处理

- 多 AI 同时改同一 schema → git 合并冲突 → 人工协调
- 改 schema 前先 pull 最新
- 改完立即提交,避免长分支

### 4.3 上下文传递

新 AI 接手时,旧 AI 应:
1. 确保所有改动已提交
2. 在 `01-progress.md` 记录当前状态
3. 告知新 AI:"读 README.md 和 01-progress.md 接手"

---

## 五、跨环境部署

### 5.1 携带方式

整个 `tool-site-project/` 目录是**自包含**的:
- 复制到 U 盘 / 网盘 / git 仓库
- 在新环境解压
- 新环境的 AI 读 README.md 接手

### 5.2 新环境 AI 接手流程

```
1. AI 读 README.md → 了解项目结构和 schema 位置
2. AI 读 01-progress.md → 了解当前进度
3. AI 读 07-consistency-link.md → 理解一致性机制
4. AI 读本文档 → 了解工作规范
5. AI 接到具体任务 → 按 2.1 流程执行
```

### 5.3 同步回原环境

新环境的 AI 改完后:
1. 提交到 git 分支
2. 在原环境 pull / merge
3. CI 校验一致性
4. 合并到主分支

---

## 六、禁止行为

| 行为 | 原因 |
|------|------|
| 硬编码色值/间距 | 破坏设计一致性 |
| 手写 API Client | 破坏接口一致性 |
| 手动改路由文件 | 破坏工具一致性 |
| 绕过 schema 直接改代码 | 一致性链接断裂 |
| 跳过 CI 校验提交 | 一致性无法保证 |
| 不更新 01-progress.md | 接手者困惑 |
| 删除 schema 文件 | 单一来源丢失 |

---

## 七、推荐工具

| 工具 | 用途 |
|------|------|
| kscc (Claude Code) | 主开发 AI,读 CLAUDE.md |
| Cursor | 辅助开发,读 .cursorrules |
| ajv-cli | JSON Schema 校验 |
| style-dictionary | 设计令牌生成 |
| openapi-typescript | OpenAPI → TS 类型 |
| openapi-typescript-codegen | OpenAPI → API Client |
| Prism | Mock 服务 |
| pytest | 后端测试 |

---

## 八、CLAUDE.md 模板

在项目根创建 `.claude/CLAUDE.md`,内容如下(供 kscc 读取):

```markdown
# 知鸟工具箱 - AI 协作上下文

## 项目位置
- 文档: ./tool-site-project/
- 原型: ./tool-site-project/prototypes/
- Schema: ./tool-site-project/schemas/

## 必读文档
1. ./tool-site-project/README.md
2. ./tool-site-project/01-progress.md
3. ./tool-site-project/07-consistency-link.md
4. ./tool-site-project/08-ai-collaboration.md

## 核心规范
- 所有变更从 schema 开始,不直接改代码
- 四大单一事实来源:
  - schemas/design-tokens.json (视觉)
  - schemas/tools.registry.json (工具)
  - schemas/api/openapi.yaml (接口)
  - .claude/CLAUDE.md (本文件,上下文)

## 生成命令
- npm run build:tokens  # 设计令牌 → CSS
- npm run gen:api        # OpenAPI → TS Client
- npm run validate       # 校验所有 schema

## 禁止
- 硬编码色值
- 手写 API Client
- 手动改路由文件
- 绕过 schema 改代码
```

---

## 九、质量检查清单

AI 提交前自检:

- [ ] 改了对应的 schema 吗?
- [ ] 执行了生成命令吗?
- [ ] 本地校验通过吗?
- [ ] 更新了 01-progress.md 吗?
- [ ] 没有硬编码色值吗?
- [ ] 没有手写 API Client 吗?
- [ ] 提交信息清晰吗?

全部 ✓ 才能提交。
