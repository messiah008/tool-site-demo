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
2. `./tool-site-project/22-platform-pivot-strategy.md` — **平台转型战略(当前最新)**(对标 bmcx+120 工具)
3. `./tool-site-project/23-quick-start-operations.md` — **每日执行清单**(可勾选)
4. `./tool-site-project/01-progress.md` — 当前进度(v0.4.0 平台转型启动)
5. `./tool-site-project/07-consistency-link.md` — 一致性机制(核心)
6. `./tool-site-project/08-ai-collaboration.md` — AI 工作规范
7. `./tool-site-project/12-implementation-playbook.md` — 旧版实施流程(参考)
8. `./tool-site-project/33-seo-optimization.md` — **SEO 优化方案(加新工具必读,seo 字段规范)**
9. `./tool-site-project/32-ui-design-spec.md` — **产品设计规范 v2.2**(工具改造/新页必读)
10. `./tool-site-project/36-user-payment-cardkey-modules.md` — **用户/支付/卡密三模块补全方案**(商业化必读, v1.2 含对外卡密 API)
11. `./tool-site-project/37-development-plan.md` — **三模块详细开发计划**(36号落地执行版,含代码现状对齐/踩坑红线/任务依赖,执行P0必读)
12. `./tool-site-project/38-placeholder-integration-checklist.md` — **占位代码接入清单**(外部资源到位后逐一替换占位,含文件:行/接入步骤/所需资源)
13. `./tool-site-project/38-competitor-and-conversion-gap.md` — **竞品分析+转换能力缺口+Stirling实测**(P0:stirling-worker 接通 12 个"假就绪"工具,含实测端点表/worker 设计/落地路线)
14. `./tool-site-project/40-paid-tool-productization-rules.md` — **付费工具产品化通用规则**(扣费保障/进度用时真化/结果展示/余额UX/错误码/SSOT三处对齐,付费转换工具上线前门禁)
15. 与任务相关的具体文档

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

### 调试命令(AI 必看)

| 命令 | 作用 |
|------|------|
| `python scripts/diagnose.py --latest` | 查前端最新报错(用户报问题时第一步) |
| `python scripts/diagnose.py --limit 20` | 查最近 20 条错误 |
| `python scripts/diagnose.py --clear` | 清空日志(修复后) |
| `python scripts/verify_all.py` | 全量验证(53 项,改动后必跑) |
| `python scripts/check_consistency.py` | schema 一致性校验 |

**用户报告页面异常时,AI 第一步执行 `python scripts/diagnose.py --latest` 拿错误,无需让用户手动 F12。** 详见 [26-log-collection.md](./26-log-collection.md)。

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
- P3 工具矩阵:Stirling 集成(⚠️实测 12 工具假就绪、链路从未接通,P0 需先写 stirling-worker,见 [38-competitor-and-conversion-gap.md](./38-competitor-and-conversion-gap.md) §五)
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
- ❌ 加新工具不填 `seo.title`/`seo.description`(validate 会拦截,见 33-seo-optimization.md)

## 项目背景

知鸟工具箱是在线工具站,核心特点:
- 50+ 免费小工具(引流)
- 高质量 PDF 转 Word 付费服务(WPS + OCR)
- 卡密兑换机制(对接闲鱼/拼多多)

战略: Fork Stirling-PDF + WPS Worker 增强(不从零自建)

## 当前阶段

- v1.5.1: 119 工具(110+ 活跃),SSG 预渲染 SEO 落地,verify_all 119/119
- 三模块(用户/支付/卡密)P0-P4 全部完成(兑换/在线支付/用户系统/卡密运营/对账风控)
- ⚠️ Stirling 转换链路待接通:12 个 backend=stirling 工具实测为假就绪(无消费者),P0 写 stirling-worker(见 [38-competitor-and-conversion-gap.md](./38-competitor-and-conversion-gap.md))
- 待外部资源:域名/SSL/备案/VPS/微信支付资质/SMTP(详见 01-progress)

详见 `01-progress.md`

## 测试卡密

| 卡密 | 用途 |
|------|------|
| `ZN-DEMO-2026-TEST` | 演示用,兑换后 100 次 |
| `ZN-ABCD-1234-EFGH` | 演示用,兑换后 50 次 |
