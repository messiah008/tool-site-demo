# 一致性链接方案 (Consistency Link)

> **版本**: v1.0
> **最后更新**: 2026-07-02
> **状态**: 核心方案,所有开发必须遵循
> **目的**: 多模块、多 AI 协作时保证一致性

---

## 一、什么是一致性链接

### 1.1 问题背景

开发中常见的一致性问题:
- 前端改了字段名,后端没改 → 接口对不上
- 设计师改了主色,部分页面没更新 → 视觉不一致
- AI A 加了工具,AI B 不知道 → 重复造轮子
- 文档与代码不同步 → 接手者困惑

### 1.2 一致性链接的定义

**一致性链接(Consistency Link)** = 单一事实来源(SSOT)+ 代码生成 + 强制校验

核心思想:
- 所有"真相"集中定义在 schema 文件中
- 代码从 schema 自动生成,不手写
- 修改必须从 schema 开始,CI 强制校验一致性
- 多个模块、多个 AI 都引用同一份 schema,自然保持一致

### 1.3 类比

像数据库的外键约束 — 你不能直接改子表的引用,必须改主表,所有引用自动同步。

---

## 二、四大一致性链接

本项目有四条核心链接,每条链接都保证一个维度的一致性:

```
┌─────────────────────────────────────────────────────┐
│ 链接 1: 设计一致性 (Design Link)                     │
│   design-tokens.json → CSS 变量 → 所有页面           │
├─────────────────────────────────────────────────────┤
│ 链接 2: 工具一致性 (Tool Link)                       │
│   tools.registry.json → 路由/卡片/API → 前后端共享   │
├─────────────────────────────────────────────────────┤
│ 链接 3: 接口一致性 (API Link)                        │
│   openapi.yaml → TS 类型 + Python DTO → 前后端同步   │
├─────────────────────────────────────────────────────┤
│ 链接 4: AI 上下文一致性 (Context Link)               │
│   CLAUDE.md/AGENTS.md → 所有 AI 共享工作规范         │
└─────────────────────────────────────────────────────┘
```

---

## 三、链接 1: 设计一致性

### 3.1 链条

```
schemas/design-tokens.json (单一来源)
    │
    ├─→ style-dictionary 构建
    │     ├─→ frontend/src/assets/styles/tokens.css (CSS 变量)
    │     ├─→ tailwind.config.js (Tailwind 主题)
    │     └─→ scss/_tokens.scss (SCSS 变量)
    │
    └─→ 设计师 Figma Tokens 插件同步
```

### 3.2 一致性保证

- 前端**禁止**硬编码颜色/间距/圆角,必须引用 CSS 变量
- 改令牌 → 执行 `npm run build:tokens` → 所有页面自动更新
- CI 检查:扫描 `.vue` 文件,发现硬编码色值则失败

### 3.3 校验脚本

```bash
# scripts/check-design-tokens.sh
# 检查前端代码是否有硬编码色值
grep -rn '#[0-9a-fA-F]\{6\}' frontend/src/ \
  --include='*.vue' --include='*.css' \
  | grep -v 'tokens.css' \
  && echo "❌ 发现硬编码色值" && exit 1 \
  || echo "✓ 无硬编码色值"
```

详见 [05-design-tokens.md](./05-design-tokens.md)。

---

## 四、链接 2: 工具一致性

### 4.1 链条

```
schemas/tools.registry.json (单一来源)
    │
    ├─→ 前端
    │     ├─→ 路由自动生成 (scripts/gen-routes.ts)
    │     ├─→ 首页卡片自动渲染
    │     └─→ SEO 元数据自动填充
    │
    ├─→ 后端
    │     ├─→ 启动时加载到内存
    │     ├─→ API 校验 tool_id
    │     └─→ 调度到对应 backend (frontend/stirling/wps-worker)
    │
    └─→ AI
          └─→ 添加工具前读 schema,改完更新 registry
```

### 4.2 一致性保证

- 新增工具**只改** `tools.registry.json`
- 前端路由、首页卡片自动出现
- 后端 API 自动支持新工具 ID
- AI 不需要改多处代码,只改一份 JSON

### 4.3 校验

```bash
# CI: 校验 registry 符合 schema
npx ajv validate -s schemas/tool.schema.json -d schemas/tools.registry.json

# CI: 校验前端路由与 registry 一致
node scripts/check-routes.js
```

详见 [06-tool-registry.md](./06-tool-registry.md)。

---

## 五、链接 3: 接口一致性

### 5.1 链条

```
schemas/api/openapi.yaml (单一来源)
    │
    ├─→ 前端
    │     ├─→ openapi-typescript 生成 TS 类型
    │     ├─→ openapi-typescript-codegen 生成 API Client
    │     └─→ 前端调用 API 用生成的 Client,不手写
    │
    ├─→ 后端
    │     ├─→ FastAPI 自动生成 OpenAPI (与 yaml 比对)
    │     ├─→ Pydantic 模型从 yaml 生成
    │     └─→ CI 校验代码与契约一致
    │
    └─→ 测试
          ├─→ Prism 启动 Mock 服务
          └─→ 契约测试(Pact)
```

### 5.2 一致性保证

- 接口变更**先改** `openapi.yaml`
- 前端 API Client 重新生成
- 后端 FastAPI 实现必须匹配契约
- CI 比对前后端 OpenAPI,不一致则失败

### 5.3 校验流程

```bash
# 1. 前端生成
npm run gen:api
# → frontend/src/api/types.ts
# → frontend/src/api/client/

# 2. 后端导出实际 OpenAPI
uvicorn app.main:app &
curl http://localhost:8000/openapi.json > /tmp/actual.json

# 3. 比对
python scripts/compare_openapi.py schemas/api/openapi.yaml /tmp/actual.json
# 不一致 → 退出码 1,CI 失败
```

### 5.4 变更示例

**场景**: 给 ConvertRequest 增加字段 `language`

1. 改 `schemas/api/openapi.yaml`:
```yaml
ConvertRequest:
  properties:
    language: { type: string, default: 'zh' }
```

2. 前端执行 `npm run gen:api` → TS 类型自动更新
3. 后端 FastAPI 实现 `language` 字段 → Pydantic 模型更新
4. 前端代码使用新字段:
```typescript
await ConvertService.createConvert({ ..., language: 'zh' })
```

5. CI 通过 → 合并

**全程零手写接口定义**,前后端天然一致。

详见 [04-api-contract.md](./04-api-contract.md)。

---

## 六、链接 4: AI 上下文一致性

### 6.1 链条

```
.claude/CLAUDE.md (项目根)
    │
    ├─→ 引用 schemas/ 下的单一来源
    ├─→ 定义 AI 工作规范
    │
    └─→ 所有 AI 接手时必读
          ├─→ kscc (Claude Code)
          ├─→ Cursor
          ├─→ GitHub Copilot Chat
          └─→ 其他 AI 助手
```

### 6.2 多 AI 协作场景

| 场景 | 一致性保证 |
|------|----------|
| AI A 加工具 | 改 registry,AI B 接手时读 CLAUDE.md 知道从 registry 加 |
| AI A 改 API | 改 openapi.yaml,AI B 接手时读 CLAUDE.md 知道先生成 Client |
| AI A 改视觉 | 改 design-tokens.json,AI B 接手时读 CLAUDE.md 知道不硬编码 |
| 多 AI 并行 | 都引用同一份 schema,改完各自生成代码,CI 校验 |

### 6.3 跨环境部署

用户提到"在其他环境部署时 AI 交互",一致性链接的关键:

1. **携带**: 把整个 `tool-site-project/` 目录复制到新环境
2. **AI 接手**: 新环境的 AI 读 `README.md` → 找到所有 schema
3. **工作**: AI 修改从 schema 开始,生成代码,CI 校验
4. **同步**: 把改完的 schema + 生成代码带回原环境,合并

详见 [08-ai-collaboration.md](./08-ai-collaboration.md)。

---

## 七、CI/CD 强制校验

### 7.1 GitHub Actions 校验流程

```yaml
# .github/workflows/consistency-check.yml
name: Consistency Check
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 链接 1: 设计令牌
      - name: Check design tokens
        run: |
          npm install -D style-dictionary
          npm run build:tokens
          git diff --exit-code frontend/src/assets/styles/tokens.css \
            || (echo "❌ tokens.css 未更新,请执行 npm run build:tokens" && exit 1)

      # 链接 1: 硬编码色值检查
      - name: Check no hardcoded colors
        run: bash scripts/check-design-tokens.sh

      # 链接 2: 工具注册表 schema
      - name: Validate tool registry
        run: |
          npm install -g ajv-cli
          ajv validate -s schemas/tool.schema.json -d schemas/tools.registry.json

      # 链接 3: OpenAPI 一致性
      - name: Check API contract
        run: |
          # 启动后端
          cd backend && pip install -r requirements.txt &
          uvicorn app.main:app &
          sleep 5
          # 比对
          curl http://localhost:8000/openapi.json > /tmp/actual.json
          python scripts/compare_openapi.py schemas/api/openapi.yaml /tmp/actual.json

      # 链接 3: 前端 API Client 已生成
      - name: Check frontend API client
        run: |
          cd frontend
          npm run gen:api
          git diff --exit-code src/api/ \
            || (echo "❌ API Client 未更新,请执行 npm run gen:api" && exit 1)
```

### 7.2 失败场景

| 场景 | CI 失败原因 | 修复 |
|------|-----------|------|
| 前端硬编码色值 | check-design-tokens.sh 退出 1 | 改用 CSS 变量 |
| registry 不符合 schema | ajv 校验失败 | 修正字段 |
| 后端 API 与契约不符 | compare_openapi.py 退出 1 | 改后端实现或改契约 |
| 前端 API Client 过期 | git diff 有变化 | 执行 `npm run gen:api` |

---

## 八、落地检查清单

新接手 AI 或开发者,确认以下都成立:

- [ ] `schemas/design-tokens.json` 是视觉的唯一来源
- [ ] `schemas/tools.registry.json` 是工具的唯一来源
- [ ] `schemas/api/openapi.yaml` 是接口的唯一来源
- [ ] `.claude/CLAUDE.md` 是 AI 工作规范的唯一来源
- [ ] 前端代码无硬编码色值(用 CSS 变量)
- [ ] 前端路由从 registry 自动生成
- [ ] 前端 API Client 从 OpenAPI 自动生成
- [ ] 后端 FastAPI 路由与 OpenAPI 一致
- [ ] CI 校验全部通过

---

## 九、常见问题

### Q1: 紧急修复可以直接改代码吗?

**A**: 可以,但必须在 24 小时内回写到 schema。否则下次 CI 会失败。

### Q2: AI 不知道有 schema 怎么办?

**A**: 接手 AI 必读 `README.md` 和 `CLAUDE.md`,里面明确指向所有 schema。详见 [08-ai-collaboration.md](./08-ai-collaboration.md)。

### Q3: schema 之间有冲突怎么办?

**A**: 以 `schemas/` 目录为准,代码让步于 schema。修改 schema 后重新生成代码。

### Q4: 跨环境部署如何同步?

**A**: 整个 `tool-site-project/` 目录是自包含的。复制到新环境,AI 读 README 接手,改完 schema 后用 git 同步回原环境。
