# 知鸟工具箱 - AI 协作上下文 (CLAUDE.md)

> 本文件是 kscc/Claude Code 等 AI 助手的项目记忆文件。
> 接手本项目时必读本文件,并按规范工作。

## 项目位置

- 项目根: `C:\Users\Administrator\tool-site-project\`
- 文档目录: `./tool-site-project/`
- 静态原型: `./tool-site-project/prototypes/`
- Schema(单一事实来源): `./tool-site-project/schemas/`

## 必读文档(按顺序)

1. `./tool-site-project/README.md` — 项目总览
2. `./tool-site-project/12-implementation-playbook.md` — **实施流程总纲**(怎么落地+迭代)
3. `./tool-site-project/01-progress.md` — 当前进度
4. `./tool-site-project/07-consistency-link.md` — 一致性机制(核心)
5. `./tool-site-project/08-ai-collaboration.md` — AI 工作规范
6. 与任务相关的具体文档

## 核心规范

**所有变更从 schema 开始,不直接改代码**

### 四大单一事实来源

| Schema | 路径 | 控制维度 |
|--------|------|---------|
| 设计令牌 | `schemas/design-tokens.json` | 视觉一致性 |
| 工具注册表 | `schemas/tools.registry.json` | 工具一致性 |
| API 契约 | `schemas/api/openapi.yaml` | 接口一致性 |
| AI 规范 | `.claude/CLAUDE.md` (本文件) | 上下文一致性 |

### 生成命令

| 命令 | 作用 |
|------|------|
| `npm run build:tokens` | design-tokens.json → CSS 变量 |
| `npm run gen:api` | openapi.yaml → TS API Client |
| `npm run validate` | 校验所有 schema |
| `ajv validate -s schemas/tool.schema.json -d schemas/tools.registry.json` | 校验工具注册表 |

## 任务执行流程

**详见 [12-implementation-playbook.md](./12-implementation-playbook.md)**

核心循环(每个变更都走这 8 步):
```
1. 识别需求 → 2. 改 Schema(SSOT) → 3. 生成代码 → 4. 实现业务
→ 5. 本地校验 → 6. 更新进度 → 7. CI 校验 + 提交 → 8. 上线 + 监控
```

落地路线图(P0-P5):
- P0 验证期(第1周):闲鱼验证需求
- P1 基础设施(第2-3周):后端骨架+数据库+队列
- P2 核心闭环(第4-5周):卡密+PDF转Word 跑通
- P3 工具矩阵(第6-8周):Stirling+10个工具
- P4 商业化(第9-10周):支付+用户系统
- P5 上线运营(第11-12周):公网+监控

每阶段有 Go/No-Go 验收门禁,过不了不进下一阶段。

## 任务对照表

| 任务 | 改什么 | 生成命令 |
|------|--------|---------|
| 加新工具 | `tools.registry.json` + 前端页面 | 无(运行时加载) |
| 改视觉 | `design-tokens.json` | `npm run build:tokens` |
| 改接口 | `openapi.yaml` | `npm run gen:api` |
| 改业务逻辑 | 业务代码 | 无 |
| 加新页面 | 前端代码 + registry(如工具页) | 无 |

## 禁止行为

- ❌ 硬编码色值/间距(用 CSS 变量)
- ❌ 手写 API Client(从 OpenAPI 生成)
- ❌ 手动改路由文件(从 registry 生成)
- ❌ 绕过 schema 直接改代码
- ❌ 跳过 CI 校验提交
- ❌ 不更新 01-progress.md
- ❌ 删除 schema 文件

## 项目背景

知鸟工具箱是在线工具站,核心特点:
- 50+ 免费小工具(引流)
- 高质量 PDF 转 Word 付费服务(WPS + OCR)
- 卡密兑换机制(对接闲鱼/拼多多)

战略: Fork Stirling-PDF + WPS Worker 增强(不从零自建)

## 当前阶段

- v0.2.0: 静态原型 + 卡密兑换闭环(已完成)
- v0.3.0: Vue 工程化 + 后端骨架(进行中)

详见 `01-progress.md`

## 测试卡密

| 卡密 | 用途 |
|------|------|
| `ZN-DEMO-2026-TEST` | 演示用,兑换后 100 次 |
| `ZN-ABCD-1234-EFGH` | 演示用,兑换后 50 次 |
