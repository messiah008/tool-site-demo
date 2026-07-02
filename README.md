# 知鸟工具箱 - 项目文档总览

> **项目类型**: 在线工具站(工具矩阵 + 高质量付费转换)
> **当前版本**: v0.2.1 (原型期)
> **最后更新**: 2026-07-02
> **文档语言**: 中文

---

## ⭐ 先读这个

**新接手者(人或 AI)请按顺序读**:
1. 本 README.md(项目总览)
2. [12-implementation-playbook.md](./12-implementation-playbook.md) — **实施流程总纲**(怎么落地+迭代)
3. [01-progress.md](./01-progress.md) — 当前进度
4. [07-consistency-link.md](./07-consistency-link.md) — 一致性机制
5. [CLAUDE.md](./CLAUDE.md) — AI 项目记忆

---

## 项目简介

知鸟工具箱是一个面向中文用户的在线工具集合站点,核心差异化:
- 50+ 免费小工具(引流)
- 高质量 PDF 转 Word 付费服务(WPS + OCR 引擎)
- 卡密兑换机制(对接闲鱼/拼多多电商渠道)

## 文档导航

### 必读文档(按顺序)

| # | 文档 | 用途 | 读者 |
|---|------|------|------|
| 00 | [README.md](./README.md) | 项目总览和文档索引 | 所有人 |
| 01 | [01-progress.md](./01-progress.md) | 迭代进度记录 | 所有人 |
| 02 | [02-frontend-design.md](./02-frontend-design.md) | 前端设计方案 | 前端/AI |
| 03 | [03-backend-architecture.md](./03-backend-architecture.md) | 后端架构方案 | 后端/AI |
| 04 | [04-api-contract.md](./04-api-contract.md) | API 契约(OpenAPI) | 前后端/AI |
| 05 | [05-design-tokens.md](./05-design-tokens.md) | 设计令牌(视觉一致性) | 前端/AI |
| 06 | [06-tool-registry.md](./06-tool-registry.md) | 工具注册表(SSOT) | 前后端/AI |
| 07 | [07-consistency-link.md](./07-consistency-link.md) | **一致性链接方案(核心)** | 所有人 |
| 08 | [08-ai-collaboration.md](./08-ai-collaboration.md) | AI 协作规范 | 所有 AI |
| 09 | [09-deployment.md](./09-deployment.md) | 部署方案 | 运维 |
| 10 | [10-worker-integration.md](./10-worker-integration.md) | Worker 接入标准与解耦 | 后端/Worker 开发 |
| 11 | [11-microservice-architecture.md](./11-microservice-architecture.md) | 多微服务架构 | 后端/架构 |
| **12** | **[12-implementation-playbook.md](./12-implementation-playbook.md)** | **实施流程总纲(落地+迭代)** | **所有人** |

### 资源文件

| 路径 | 内容 |
|------|------|
| `prototypes/` | 4 个静态 HTML 原型(首页/工具页/兑换页/用户中心) |
| `schemas/tool.schema.json` | 工具定义 JSON Schema |
| `schemas/design-tokens.json` | 设计令牌 JSON |
| `schemas/api/openapi.yaml` | OpenAPI 3.0 规范 |

## 核心概念:一致性链接(Consistency Link)

本项目采用"**单一事实来源 + 代码生成**"模式,确保多模块、多 AI 协作时的一致性。详见 [07-consistency-link.md](./07-consistency-link.md)。

```
设计令牌(JSON) ──┐
工具注册表(JSON) ─┼──→ 代码生成 ──→ 前端 CSS/路由/卡片
API 契约(OpenAPI) ┤                ──→ 后端 Controller/DTO
                  └──→ 类型生成 ──→ 前后端 TS 类型
```

**核心原则**:
1. 改 schema → 自动生成代码,不手写
2. 改契约 → 前后端类型同步更新
3. AI 修改代码前必须读 schema,改完必须更新 schema
4. 文档与代码版本绑定,变更记录在 01-progress.md

## 快速开始

### 查看原型
直接双击 `prototypes/index.html` 在浏览器打开。

### 完整流程演示
1. 打开 `prototypes/redeem.html`,输入测试卡密 `ZN-DEMO-2026-TEST`
2. 兑换成功后跳转 `pdf-to-word.html`
3. 上传 PDF,选"高质量转换",点开始
4. 余额扣减,转换完成
5. 打开 `user-center.html` 查看记录

### 测试卡密
| 卡密 | 用途 |
|------|------|
| `ZN-DEMO-2026-TEST` | 演示用,兑换后获得 100 次 |
| `ZN-ABCD-1234-EFGH` | 演示用,兑换后获得 50 次 |

## 技术栈预览

| 层 | 技术 | 状态 |
|----|------|------|
| 前端 | Vue 3 + Vite + TypeScript | 计划中 |
| UI | Element Plus / Naive UI + Tailwind | 计划中 |
| 后端 | FastAPI (Python) 或 NestJS (Node) | 计划中 |
| 数据库 | PostgreSQL + Redis | 计划中 |
| 对象存储 | MinIO | 计划中 |
| Worker | Python + pywinauto + WPS 客户端 | 设计完成 |
| 部署 | Docker Compose + Cloudflare | 计划中 |

## AI 接手须知

如果你是新接手的 AI,请按顺序读:
1. 本 README.md
2. [12-implementation-playbook.md](./12-implementation-playbook.md) — 实施流程(怎么落地+迭代)
3. [01-progress.md](./01-progress.md) — 了解当前进度
4. [07-consistency-link.md](./07-consistency-link.md) — 理解一致性机制
5. [08-ai-collaboration.md](./08-ai-collaboration.md) — 你的工作规范
6. [CLAUDE.md](./CLAUDE.md) — 项目记忆
7. 与你任务相关的具体文档

**关键**: 不要绕过 schema 直接改代码,所有变更从 schema 开始。实施流程见 [12-implementation-playbook.md](./12-implementation-playbook.md)。
